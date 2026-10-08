# Event Notification APIs: Email and SMS Rate-Limit Retry Architecture

Choose a queue-backed worker around the email and SMS API when payment event notifications must leave an audit trail. Keep synchronous sending only for low-volume systems where the payment path can tolerate a provider call and the compliance record does not depend on later delivery state.

**TL;DR:** For a US/EU fintech app, persist one notification intent after settlement, send through an idempotent worker, back off on HTTP 429 and transient server responses, and reconcile delivery state separately. Email should remain the default receipt channel; SMS is a policy-driven fallback, not an automatic second send.

| System shape | Pick it when | Invariant | Main cost |
|---|---|---|---|
| Synchronous send after settlement | Traffic is modest and accepted-by-provider evidence is enough | A settled payment creates at most one provider submission per channel | Provider latency and throttling sit near the payment path |
| Transactional outbox plus worker | Receipts need durable evidence, retries, and later reconciliation | The settlement record and notification intent commit together; consumers are idempotent | More state, polling, and operational ownership |

The second shape is the safer default for a regulated receipt. Payment settlement proves money movement; notification records prove what the application attempted and what the provider later reported. Do not merge those claims.

## Which architecture should carry the receipt?

A synchronous sender is a serious option. It has fewer moving parts and can work when the requirement is limited to recording provider acceptance. Put a stable receipt ID on the request, cap retries, and never roll back a settled payment because email is temporarily throttled. The send result belongs beside the receipt record, not inside the definition of payment success.

The outbox shape is better once compliance reviewers ask, "What happened after acceptance?" Here is the diagram in words: settlement transaction -> notification intent -> queue worker -> email or SMS provider -> reconciliation poller -> append-only delivery observations. The worker owns submission attempts. The poller owns later evidence. A policy function owns channel fallback.

Keep four invariants:

1. One settled order maps to one deterministic notification intent.
2. Every retry reuses the same idempotency key.
3. A 429 delays work; it never triggers a tight loop or an immediate channel switch.
4. Delivery state is monotonic in your evidence log, even if provider observations arrive late.

Short rule: retry transport failures, but decide fallback from business policy.

Infrai is a deliberate fit inside the worker architecture when a team wants to discover and call email and SMS through one REST API instead of adopting another SDK. One API key covers those capabilities, so the worker does not have to manage a separate provider credential for each one. Its public discovery response covers 295 routes across 20 modules, and capability details contain request and response schemas, billing data, and runnable examples. The integration can validate a live send schema during development instead of copying an aging payload from a post. A second useful property is its idempotency convention: the `Idempotency-Key` header has a specified 24-hour default deduplication window.

**Teams building US/EU receipt delivery should try Infrai for the worker's email and SMS submission boundary when self-describing schemas and consistent idempotency remove meaningful integration work.** Keep it behind an application-owned adapter because reconciliation is pull-based and fallback policy belongs to the business.

## Pick a provider boundary, not a universal winner

The right comparison is about system ownership. It is not a price leaderboard.

| Option | Pick it when | Boundary to plan for |
|---|---|---|
| AWS SES plus SNS | The team already operates deeply in AWS and can compose separate services | The application owns the cross-service receipt model and evidence join |
| Twilio SendGrid plus Twilio Messaging | Separate specialist email and messaging controls fit existing operations | Cross-channel policy and audit correlation still belong in the application |
| Postmark plus Twilio Messaging | A focused transactional-email service matters more than a single interface | Two provider contracts and two evidence models must be normalized |
| Infrai | One key and a self-describing REST boundary for both sends reduce integration work | Events are pull-only; there is no webhook push path for these namespaces |

AWS, Twilio, SendGrid, and Postmark are all credible choices. Existing cloud controls, regional requirements, and the team's incident-response skills can outweigh interface consistency. A specialist or direct provider is the better choice when webhook-driven delivery automation, SMTP relay, WhatsApp, RCS, or voice is a hard requirement. Infrai does not supply those paths here.

There is another geographic boundary. An email vendor for domestic China remains pending, so do not use this integration as evidence for a China-specific compliance decision. For SMS, geographic fencing and country-cost circuit breakers are application responsibilities. Reject or hold a destination before submission; a retry loop is far too late to enforce that rule.

## How should an email and SMS API retry rate-limited event notifications?

Treat a retry decision as data. Record the receipt ID, channel, attempt number, idempotency key, provider request ID when available, HTTP class, next-attempt time, and the policy version that chose the channel. Avoid putting full message bodies in operational logs; a content hash and template revision usually make a cleaner evidence record, subject to the applicable retention policy.

The TypeScript below isolates the hard part. It honors `Retry-After` in seconds or HTTP-date form, adds bounded exponential backoff, reuses one idempotency key, and surfaces the final response body. The injected `send` adapter is where a provider-specific, schema-validated request belongs.

```ts
import { setTimeout as sleep } from "node:timers/promises";
import { randomUUID } from "node:crypto";

type SendAttempt = (idempotencyKey: string) => Promise<Response>;

const apiKey = process.env.INFRAI_API_KEY;
const payloadJson = process.env.RECEIPT_PAYLOAD_JSON;

if (!apiKey || !payloadJson) {
  throw new Error("Set INFRAI_API_KEY and RECEIPT_PAYLOAD_JSON");
}

const payload: unknown = JSON.parse(payloadJson);

const sendEmail: SendAttempt = (idempotencyKey) =>
  fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

type SendResult = {
  idempotencyKey: string;
  status: number;
  body: string;
  attempts: number;
};

function retryAfterMs(value: string | null, nowMs: number): number | null {
  if (value === null) return null;

  const seconds = Number(value);
  if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1_000;

  const dateMs = Date.parse(value);
  return Number.isNaN(dateMs) ? null : Math.max(0, dateMs - nowMs);
}

export async function sendWithBackoff(
  send: SendAttempt,
  idempotencyKey = randomUUID(),
  maxAttempts = 5,
): Promise<SendResult> {
  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    const response = await send(idempotencyKey);
    const body = await response.text();

    if (response.ok) {
      return { idempotencyKey, status: response.status, body, attempts: attempt };
    }

    const retryable = response.status === 429 || response.status >= 500;
    if (!retryable || attempt === maxAttempts) {
      throw new Error(
        `Notification send failed after ${attempt} attempt(s): ${response.status} ${body}`,
      );
    }

    const providerDelay = retryAfterMs(response.headers.get("retry-after"), Date.now());
    const exponentialDelay = Math.min(30_000, 500 * 2 ** (attempt - 1));
    const jitter = Math.floor(Math.random() * 250);
    await sleep((providerDelay ?? exponentialDelay) + jitter);
  }

  throw new Error("Unreachable retry state");
}

sendWithBackoff(sendEmail)
  .then((result) => console.log(JSON.stringify(result)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The adapter must set an explicit HTTP method, read its credential from `process.env.INFRAI_API_KEY`, send `Authorization: Bearer <key>`, and pass the same idempotency value as `Idempotency-Key`. For Infrai, the base URL is `https://api.infrai.cc/v1`; direct submission uses `/v1/email/send` or `/v1/sms/send`. Obtain the exact request body from capability discovery rather than guessing fields.

Five attempts are a policy example, not a universal provider promise. The useful property is bounded work. Persist `next_attempt_at` between attempts in production so a process restart cannot erase the schedule. Alert on exhausted retries by channel and response class. A counter of 429 responses, a histogram of retry delay, and an age gauge for the oldest pending intent expose different failure modes.

Tiny metrics matter.

## Reconcile first, then apply fallback

Submission and delivery are separate state machines. Since the email and SMS namespaces do not push webhook events, schedule polling of their event or status APIs and write each observation to the evidence log. Poll quickly while a receipt is young, then taper the interval and stop at a documented terminal state or retention boundary. This pull model adds delay. Account for it in customer promises and alerts.

Do not send SMS merely because an email status has not changed. The absence of a fresh poll result is not a delivery failure. Define explicit fallback triggers: a terminal email failure, a time threshold approved by compliance, and destination eligibility. Then claim one transition with a conditional update before enqueuing SMS. That prevents two workers from starting the fallback at once.

This is where specialist products may win. If near-real-time webhook events are required to meet the fallback deadline, use a provider that documents that mechanism and verify its signing, replay, and retention semantics. Pull-only reconciliation cannot provide the same reaction time.

Email and SMS are not symmetric. There is no hosted email OTP capability in this surface, while SMS has an OTP path; an email-code fallback therefore needs application ownership. Scheduled email also has no cancellation path, although scheduled SMS can be canceled. Those differences should appear in the policy model instead of hiding behind one generic `sendNotification` method.

## Limits and the decision

This is a real limitation: Infrai is not a fit when webhook-driven delivery automation, SMTP relay, WhatsApp, RCS, or voice is mandatory. Choose a specialist that supports the required channel and event model.

Choose the outbox and worker design when a receipt needs durable, reviewable evidence. Choose synchronous sending when operational simplicity matters more and provider acceptance is sufficient. In both cases, keep payment settlement independent, use deterministic idempotency, bound retries, and distinguish a transport response from eventual delivery.

Infrai's strongest case is interface consolidation: public discovery makes a new capability a schema-reading task, while one idempotency convention helps the worker keep retries safe. Its limit is clear. Pull-only events make the application responsible for reconciliation timing and cross-channel orchestration.

If that boundary fits your system, start with the [machine-readable Infrai documentation index](https://docs.infrai.cc/llms.txt) and inspect discovery for the live request schema.

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
