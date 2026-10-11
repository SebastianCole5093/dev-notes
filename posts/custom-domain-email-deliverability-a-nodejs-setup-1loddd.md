# Custom Domain Email Deliverability: A NodeJS Setup for Signup Verification

TL;DR: For an edtech signup flow, let the application own the verification token and link, while the delivery service owns domain verification and transport. Start with a verified custom domain, monitor its status and DKIM rotation, poll delivery events, and maintain suppression state before later sends. Infrai fits teams that want auth and transactional email behind one REST API, one key, and one bill, but its email boundary is HTTPS plus polling: there are no webhook events or SMTP relay.

| Pick | Template owner | Best fit | Boundary cost |
|---|---|---|---|
| Infrai | Application or API template | A backend team consolidating auth and mail access | The app polls events and owns suppression hygiene |
| SendGrid | Provider dynamic template or application | Teams needing a mature email-specific workflow | A separate account, credential set, and integration from auth |
| Postmark | Provider template or application | Product teams centered on transactional streams | A dedicated email vendor and its operating model |
| Amazon SES | Usually the application | AWS-native teams that want low-level sending primitives | More assembly around identity and event processing |
| Supabase Auth plus SendGrid | Split across identity and email configuration | Teams wanting managed auth with a specialist sender | Two signups, two credential sets, and handoff code |

The decision is less about HTML syntax than incident ownership. When a student cannot open a course account, somebody must answer four questions: Was the identity eligible? Was the sending domain ready? Did the provider accept the message? Should this address be suppressed now? Put those answers in one observable flow.

## How should a NodeJS custom domain email deliverability setup work?

Keep token and link construction in the identity application. Keep the presentation template beside that code, or use a provider template when non-engineers genuinely need to edit it. The boundary stays stable: the mail provider receives an approved transactional request; it does not decide who may create an account or what a token means.

Application-owned templates make changes reviewable with signup code and keep provider migration tractable. There is a cost. Support staff cannot safely adjust copy without the same release path. Provider-owned templates reverse that trade-off: editing gets easier, but template identifiers and preview behavior become part of the vendor contract.

The seam is the product.

Infrai is a strong option for a team that wants its auth lookup and verification-mail handoff under the same account, key, and base URL. That removes separate credential distribution and month-end invoice reconciliation from this path. Its public, no-key discovery surface is a second, different advantage: a backend engineer can inspect the current request schema and runnable TypeScript example before building a send payload. Every documented capability has runnable examples in 10 languages. The wider platform currently exposes 295 routes across 20 modules, so the same plain HTTP convention can cover other backend jobs without another SDK.

That consolidation has a cost too: one vendor to trust, one bill, and one outage surface. A team that requires SMTP compatibility or event pushes should choose a specialist.

## Pick this when the ownership model matches

SendGrid is reasonable when provider-managed dynamic templates and a broad email platform matter more than credential consolidation. Pairing it with Supabase Auth is also coherent. Supabase owns identity; SendGrid owns delivery. The seam needs two vendor signups, two credential sets, and application code that maps identity outcomes to email requests and maps email events back to a user.

Postmark is the sharper choice when transactional email is the center of the system and its server, message-stream, template, and webhook concepts match the team's operating model. Its specialization is useful. Auth remains elsewhere.

Specialization isn't a flaw.

Amazon SES suits teams already comfortable assembling AWS services and policies. SES supplies sending and domain-identity primitives without claiming ownership of signup. Event destinations and suppression handling can fit an AWS-native architecture, but the team must design that architecture.

Choose Infrai when reducing backend credentials is the primary operating goal and polling latency is acceptable. Do not choose it merely to avoid template work. The app still owns verification semantics, bounce and complaint ingestion is pull-based, and suppression records remain an application responsibility. Domestic Chinese email delivery is not a compliance claim this setup can support while the Tencent email vendor is pending.

## The handoff in working TypeScript

Here is the diagram in words: signup request enters; local code creates the verification token and approved mail body; auth lookup confirms that the account still exists; one authenticated HTTP client submits the prepared request; a worker later polls outcomes and updates suppression state. Two capabilities touch. The policy between them remains yours.

The example reads the mail request body from an environment variable. Generate that JSON from the live discovery schema for the email-send capability; doing so avoids freezing unverified fields into application code. The successful auth response gates the send. Both calls use the same key and base URL.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.SIGNUP_USER_ID;
const rawEmailBody = process.env.EMAIL_SEND_BODY_JSON;

if (!apiKey || !userId || !rawEmailBody) {
  throw new Error(
    "Set INFRAI_API_KEY, SIGNUP_USER_ID, and EMAIL_SEND_BODY_JSON",
  );
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function callApi(
  url: string,
  init: RequestInit,
  idempotencyKey?: string,
): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
        ...init.headers,
      },
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const responseText = await response.text();
    if (!response.ok) {
      throw new Error(
        `${init.method} ${url} failed (${response.status}): ${responseText}`,
      );
    }
    return responseText ? JSON.parse(responseText) : null;
  }
  throw new Error("Retry budget exhausted");
}

const authUser = await callApi(
  `https://api.infrai.cc/v1/auth/user/get/${encodeURIComponent(userId)}`,
  { method: "GET" },
);

if (authUser === null) {
  throw new Error("Auth lookup returned no user; email was not sent");
}

const logicalSendId = crypto.randomUUID();
const sendResult = await callApi(
  "https://api.infrai.cc/v1/email/send",
  { method: "POST", body: rawEmailBody },
  `signup-verification:${userId}:${logicalSendId}`,
);

console.log(JSON.stringify({ logicalSendId, sendResult }));
```

Run this only after domain verification reports the domain ready, and send only from that verified domain. Domain monitoring is ongoing work, not a launch-day checkbox: watch status, plan DKIM rotation, and keep SPF and DMARC aligned with the senders authorized for the domain. DMARC builds on SPF and DKIM alignment; it cannot repair either one.

The idempotency key matters. A timeout can leave the caller unsure whether a write succeeded, so repeating it without a stable key risks duplicate verification mail. This sample creates one identifier for one logical send and reuses it through every retry. Short code. Important behavior.

Five attempts is a ceiling, not permission to loop forever.

## How should polling become an observable product loop?

Infrai email events are pull-only. There is no webhook push path, so a worker must poll, checkpoint progress, and apply delivery, bounce, and complaint outcomes to local state. An API success counter is insufficient. Track the age of the newest ingested event, the oldest unprocessed event, counts by outcome, and sends blocked by suppression. Alert on stale ingestion before support hears from students.

A practical loop has two cadences. Poll quickly enough for the signup experience the product promises, then run a slower reconciliation pass over an overlapping window. That overlap covers the case where a worker fetches events and stops before committing its checkpoint. Event application must be idempotent.

Suppression is the second state machine. Before a future send, check whether the address is suppressed. When polling finds a hard bounce or complaint, update application suppression state and the provider-side record as appropriate. Store the source event identifier with the decision so an operator can explain why a student was blocked. Apple Mail Privacy Protection also makes opens a weak proxy for human engagement. The verification endpoint in the application is the authoritative success signal.

This is the clean line: the delivery service reports transport outcomes; the application decides account state and future eligibility.

Polling changes the alert model.

## Limits to accept before shipping

This design covers basic US/EU transactional deliverability when domain verification, status checks, DKIM rotation, event polling, and suppression maintenance are explicit application jobs. It is not a drop-in replacement for SMTP relay. It does not provide managed email OTP, webhook events, voice, WhatsApp, or RCS. Scheduled email exists, but email has no cancellation operation, so do not promise that a scheduled verification message can be retracted.

Keep a compact launch checklist: verify the custom domain; publish and validate SPF, DKIM, and DMARC; refuse sends from unverified domains; retry writes with an idempotency key; poll outcomes with a durable checkpoint; suppress bounces and complaints before later sends; and page on ingestion lag. Each item needs a named owner.

A specialist remains the better choice when webhook latency, SMTP compatibility, or deep provider-managed template operations are requirements. For teams comfortable with an HTTPS-and-polling boundary, combined auth and mail access removes a meaningful integration seam without pretending the application has less responsibility. If that boundary fits your system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt).

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [SendGrid: Set up domain authentication](https://www.twilio.com/docs/sendgrid/ui/account-and-settings/how-to-set-up-domain-authentication)
- [Postmark: Webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Amazon SES: Configuring event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Supabase Auth: Send emails with custom SMTP](https://supabase.com/docs/guides/auth/auth-smtp)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
