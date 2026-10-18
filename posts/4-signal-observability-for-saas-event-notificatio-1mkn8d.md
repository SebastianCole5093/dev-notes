# 4-Signal Observability for SaaS Event Notification Email and SMS Provider Alerts

TL;DR: For a marketplace SaaS event notification, choose an email provider as the normal path and use SMS only when a measured escalation rule justifies the extra integration surface. Evaluate every candidate by running the same instrumented order event through its adapter. The useful denominator is not messages submitted. It is sellers notified within your deadline, with duplicates and unsupported destinations counted as failures.

That answer avoids a fragile cheapest-provider claim. Published rates change by destination, volume, and contract. Your own delivery path also adds retries, support work, and duplicate risk. A four-signal contract—accepted, delivered, engaged, and expired—turns those hidden differences into evidence your team can review.

## Replace the send call with four observable signals

The tempting mental model is short: order created, call provider, receive `200`, done. It makes every integration look alike and makes the first price table seem decisive.

The useful model is longer: order created, notification intent stored, provider accepts an attempt, a delivery event arrives, the seller acts, or the alert expires. Each transition has an identifier and timestamp. Now the team can distinguish an API acceptance from a useful notification.

Picture it as a diagram in words. The order service writes one durable intent. A dispatcher selects email or SMS. An adapter translates the intent into a provider request. Provider events flow back into one normalized event stream. A seller action closes the loop. A deadline marks anything still open as expired.

That last step matters.

Track four signals with channel, provider, destination region, template version, and attempt number as dimensions:

1. `accepted`: the provider acknowledged responsibility for the attempt.
2. `delivered`: the channel reported its delivery-level outcome.
3. `engaged`: the seller opened the order workflow from a signed, expiring link or otherwise acknowledged it.
4. `expired`: the business deadline passed without the required seller action.

Do not put an email address, phone number, message body, or raw order payload into logs or metric labels. Keep a stable internal notification ID and the minimum routing metadata needed to debug the path. Short labels make dashboards useful. They also prevent cardinality from exploding.

## A copyable channel-neutral implementation

The integration boundary should be boring. Good. One interface lets a test harness exercise all five candidates without leaking provider response shapes into order handling. The example below records accepted and rejected attempts; delivery and engagement enter later through authenticated callbacks and the order workflow.

```ts
type Channel = "email" | "sms";
type Region = "US" | "EU";

type OrderNotice = {
  notificationId: string;
  sellerId: string;
  orderId: string;
  channel: Channel;
  region: Region;
  destination: string;
  expiresAt: string;
};

type Attempt = {
  notificationId: string;
  provider: string;
  providerMessageId: string;
  acceptedAt: string;
};

interface MessagingAdapter {
  send(notice: OrderNotice): Promise<Attempt>;
}

interface Telemetry {
  increment(
    name: "notification.accepted" | "notification.rejected",
    labels: Record<string, string>,
  ): void;
  duration(name: "notification.submit_ms", value: number, labels: Record<string, string>): void;
}

export async function dispatch(
  notice: OrderNotice,
  provider: string,
  adapter: MessagingAdapter,
  telemetry: Telemetry,
): Promise<Attempt> {
  const startedAt = Date.now();
  const labels = { provider, channel: notice.channel, region: notice.region };

  try {
    const attempt = await adapter.send(notice);
    telemetry.increment("notification.accepted", labels);
    telemetry.duration("notification.submit_ms", Date.now() - startedAt, labels);
    return attempt;
  } catch (error) {
    telemetry.increment("notification.rejected", labels);
    throw error;
  }
}
```

Persist the intent before calling `dispatch`. Give the intent a uniqueness constraint derived from the order, seller, notification purpose, and template version. A retry may then create another attempt under the same intent instead of another seller-visible notification.

The callback side needs equal discipline: authenticate the request using the mechanism documented by the candidate, retain the provider event ID for deduplication, store the raw event under a limited retention policy if audit needs require it, and translate the result into your small internal vocabulary. Return success only after durable ingestion. Provider retries should be harmless.

This is also where integration effort becomes measurable. Count adapter code, callback verification, event mapping, destination validation, local test support, and operational runbooks. A pleasant send API covers only one of those rows.

## How should a SaaS event notification compare email and SMS providers?

The stated candidate set contains Resend, Postmark, and SendGrid for email, plus Twilio and Plivo for SMS. These are not interchangeable channels, so test candidates within a channel first. Then examine the completed email path and completed SMS path against the seller deadline. Product names belong in the experiment record, not in the architecture.

A compact scorecard keeps the exercise honest:

| Question | Evidence to retain | Why it changes integration effort |
| --- | --- | --- |
| Can an attempt be correlated end to end? | Internal ID, provider ID, callback ID | Missing joins turn support into log archaeology |
| Are callback retries safely deduplicated? | Replayed callback test and state transitions | Duplicate events must not trigger duplicate alerts |
| Can US and EU destinations be validated before send? | Accepted and rejected fixtures by region | Early validation produces clearer failures |
| Does the channel meet the seller deadline? | Accepted-to-delivered and accepted-to-engaged distributions | Averages hide the late tail |
| What does one useful outcome consume? | Attempts, callbacks, retries, and engineering time | Submitted-message price omits failed work |

Run a small shadow test with non-customer destinations controlled by the team. Include malformed addresses, unavailable destinations, duplicate callbacks, late callbacks, adapter timeouts, and an expired order notice. Use synthetic order IDs. Never test by surprising real sellers.

For email, domain authentication is part of the integration. Google's sender guidelines specify authentication requirements and additional requirements for bulk senders, including alignment and unsubscribe behavior in applicable mail. Validate the current guideline against your sending pattern before rollout. For SMS, collect destination-specific requirements from each candidate during evaluation because the supplied evidence here does not establish a universal rule across the US and Europe.

No single run proves deliverability. It does expose missing joins, ambiguous states, and retry bugs before the provider decision hardens into production code.

## What if SMS looks faster in the dashboard?

First check what the chart means. `accepted` latency measures your API path. It does not establish delivery, attention, or seller action. Compare like with like: accepted-to-delivered for transport behavior and accepted-to-engaged for the marketplace outcome. Show p50 and a tail percentile, plus the expired count; a lone average can make a small group of very late alerts disappear.

Then ask whether urgency belongs to every order. A practical policy can send email immediately and schedule SMS only if the seller has opted in, the order remains unacknowledged, the destination is eligible, and the deadline is close. Cancel the scheduled escalation as soon as engagement arrives. This policy adds state and testing, but it prevents two channels from racing on every order.

Keep security messages separate from commerce notifications. NIST SP 800-63B discusses out-of-band authentication and treats use of the public switched telephone network as a restricted authenticator. A new-order alert is not an authenticator. Do not let a notification-channel experiment silently become an account-authentication design.

## The decision rule

Choose the candidate with the lowest integration burden that meets the measured deadline and failure-handling requirements for its channel. Record the decision as an architecture note with the corpus version, observation window, regions, template version, missing data, and rollback condition. Do not freeze a transient rate card into the design rationale.

The crisp before-and-after is the real payoff. Before, the system knew that it called a messaging API. After, it knows which seller notification was accepted, delivered, acted on, or allowed to expire—and the team can change adapters without changing what success means.

Start with one channel, one normalized event model, and one alert on expired notifications. Add escalation only when outcome data supports it. Four signals are enough to make the trade-off visible.

## Further reading

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- NIST SP 800-63B, Digital Identity Guidelines (Authenticators): https://pages.nist.gov/800-63-3/sp800-63b.html
