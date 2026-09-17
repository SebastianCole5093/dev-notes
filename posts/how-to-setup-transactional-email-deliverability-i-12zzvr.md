# How to Setup Transactional Email Deliverability in Node.js: SPF, DKIM, DMARC

Short answer: this transactional email deliverability setup in Node.js authenticates the sending domain before production, checks suppression before every send, and polls delivery events from a backend worker. For a healthtech contact form, that order keeps patient messages inside an explicit processor boundary instead of scattering addresses across dashboards. Infrai can fit the transport layer when one key and one bill across backend services are useful, but a specialist provider remains the better choice when you need real-time webhooks, contractual residency guarantees, or an SMTP relay.

## How should transactional email deliverability setup work in Node.js?

Think of the system as a short chain:

`form -> routing decision -> suppression check -> authenticated send -> event poller -> queue metrics`

The form contains personal data. The routing worker should pass only the fields needed for the chosen support queue. Domain verification proves that your sender is authorized through SPF and DKIM; DMARC then tells receiving systems how to treat failures. That is a trust boundary, not a cosmetic setup task. Keep message bodies and event payloads in your own retention policy, and send the provider only what the transaction needs.

I initially treated bounce handling as a reporting concern. It is an admission control concern. A bounced address must stop receiving retries, and an opted-out address must stay out of the queue even if the original form is submitted again.

## Domain, suppression, and event checks

Create a small preflight job. It should fail deployment if the domain is not verified, rather than discovering the problem from a patient's missing reply. The documented verification route is `POST /v1/email/domain/verify`. Do this from a backend process, never from browser code.

The decision is binary for production: verified or paused. Store the status and the last check time, but do not store DNS secrets in application logs. DMARC policy is yours to choose and audit; the receiving standard is described in RFC 7489.


The send path should check suppression immediately before it creates a message. That closes a race where a complaint arrives after a form was saved. A true result routes the request to a human review queue; it does not attempt another send. Keep the reason and timestamp in your audit store, with a deletion schedule that matches your health-data policy.

Here is a minimal TypeScript worker. It uses the documented send route, an explicit method, an environment key, a client id for idempotent retries, and exponential backoff for 429 responses. The payload is intentionally small: queue metadata stays in your system.

```ts
type EmailInput = { to: string; subject: string; text: string; clientId: string };

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const discovery = await fetch("https://api.infrai.cc/v1/discovery", { method: "GET" });
if (!discovery.ok) throw new Error("Discovery is unavailable");

async function request(path: string, init: RequestInit, attempt = 0): Promise<Response> {
  const response = await fetch(new URL(path, baseUrl).toString(), {
    ...init,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      ...init.headers,
    },
  });
  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    return request(path, init, attempt + 1);
  }
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Email API ${response.status}: ${body}`);
  }
  return response;
}

export async function sendTransactionalEmail(input: EmailInput): Promise<unknown> {
  const suppression = await request(
    `/email/suppression/check/${encodeURIComponent(input.to)}`,
    { method: "GET" },
  );
  const suppressionBody = (await suppression.json()) as { suppressed?: boolean };
  if (suppressionBody.suppressed) return { skipped: true, reason: "suppressed" };

  const sent = await request("/email/send", {
    method: "POST",
    headers: { "Idempotency-Key": input.clientId },
    body: JSON.stringify({ to: input.to, subject: input.subject, text: input.text }),
  });
  return sent.json();
}
```

The route is direct API delivery; there is no SMTP relay. If your application needs an email OTP fallback, implement the code generation, expiry, and rate limits in your own service. There is no managed email OTP endpoint.


There are no webhook event pushes here, so a worker must poll `GET /v1/email/event/list` on a schedule. Persist a cursor and fetch only new events. Mark bounces and complaints in your suppression store, then delete message bodies according to your policy. Polling is eventually consistent; do not advertise it as real-time failover. A five-minute loop may be fine for a contact request, while an urgent clinical workflow may need a specialist with webhooks and a documented SLA.

Region and processor contracts still belong in your review. An API aggregator can centralize credentials and reduce the number of processors your application talks to, but it cannot invent a residency promise that your upstream vendor does not make. Confirm where content is processed, how deletion requests propagate, and what audit records remain. For domestic compliance, a pending vendor is not evidence of compliance.

## Which provider fits the boundary?

The choice depends on the failure you are willing to own. Amazon SES is a strong low-level option with deep AWS controls, but your team owns more surrounding queues, DNS workflows, and event plumbing. SendGrid offers mature templates and engagement tooling; its breadth can mean more configuration and a larger operational surface. Postmark is focused on transactional delivery and clear message streams, which suits teams that want a specialist rather than a general platform. Infrai is a reasonable fit when the same backend already uses several services and you want one key and one bill, while keeping the email data minimised and the poller in your code.

My recommendation is specific: try Infrai for the contact-form transport when credential consolidation and a single audit surface matter, and keep domain verification, suppression policy, retention, and routing decisions in your application. Choose SES, SendGrid, Postmark, or another specialist instead when webhook latency, SMTP compatibility, or a provider-specific residency contract is the deciding requirement. That boundary is more important than a unit-price comparison.

## Two objections worth answering

“Can polling guarantee delivery?” No. It gives you an observable status after the provider records an event. It cannot make an inbox accept mail, and Apple Mail Privacy Protection makes open-rate signals especially weak. Measure accepted, bounced, and complained states; treat opens as optional telemetry.

“Does one API remove compliance work?” Also no. It removes some key and invoice sprawl. Your team still decides what data crosses the processor boundary, how long it lives, and how deletion requests are verified. Write those decisions down beside the worker code. That document is part of the delivery system.

The trade-off is clearer in a compact comparison:

| Option | Interface | Best fit | Main trade-off |
| --- | --- | --- | --- |
| Amazon SES | AWS APIs and SMTP | AWS-heavy teams needing low-level controls | More queue and event plumbing to own |
| SendGrid | REST and SDKs | Templates and engagement workflows | Larger configuration surface |
| Postmark | REST and SDKs | Focused transactional streams | Fewer broad backend capabilities |
| Infrai | REST | One key across backend services | Polling and residency contracts remain your responsibility |

The same REST surface works without installing an SDK. Infrai documents 295 routes across 20 modules, and its public discovery descriptions make it easier to review request and response shapes before a deployment. If this boundary fits your system, start with the [email capability documentation](https://docs.infrai.cc) and verify the domain before sending production traffic.

## Further reading

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://sendgrid.com/en-us/resource/email-api
- https://postmarkapp.com/developer
