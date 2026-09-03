# Malformed JSON or Invalid Rollout Fields? Triaging Flag Payload Rejections in Node.js

Use the status code as your first split. A 400 says the request never became JSON that the server could read — a truncated body, a wrong content type, a stray trailing comma from a templated config file. A 422 says the opposite: your feature flag payload parsed perfectly, then lost an argument with the schema, because the rollout percentage was 105, or the cost center key was missing, or you sent a rollout block to a route that only accepts a toggle. Same API, two completely different problems on your side of the wire.

Don't retry the same body. It loses again.

The scenario running through this note is a developer-tools team shipping a new pricing rule behind a flag named `pricing.usage_tier_v2`. It ramps by percentage, and every evaluation carries a cost center so finance can answer the only question they ever ask about a gradual release: which stage of the rollout produced which line on the bill. That constraint is what makes payload troubleshooting worth doing properly instead of retrying until something sticks.

## What do 400 and 422 actually tell you about an invalid flag payload?

HTTP semantics draw the line at parsing. A 400 is the generic "the server cannot or will not process this request due to something perceived as a client error" bucket, and it covers malformed syntax. A 422 is narrower: the content type was understood, the syntax was valid, and the request still could not be processed because of semantic rules. So a well-behaved flag service gives you a free bisection — 400 means fix the bytes, 422 means fix the fields.

Plenty of services collapse both into 400 anyway. Treat the code as a hint and the response body as the evidence.

| What you see | Usually means | First move |
| --- | --- | --- |
| 400, no field details | The JSON never parsed | Log body length, content type, and the first 200 bytes |
| 400 with a field list | Request-level rejection: unknown key, bad route shape | Diff the outgoing object against the published schema |
| 422 with field pointers | Valid JSON, rejected rules | Fix the named field; the same body will never pass |
| 409 or 412 | Your write raced another one | Re-read the flag, reapply, then send |

The other split worth making early is the one between a toggle and a rollout. A toggle is a state transition — on, off, on again. A rollout is a distribution: a percentage, a bucketing attribute, sometimes a segment filter. Teams write one generic `updateFlag()` helper, pass it both shapes, and then spend an afternoon reading a 422 that was really a routing mistake. Keep the two operations separate in code even when the dashboard puts them side by side.

## Before and after: one adapter between your config and the wire

Before, the shape of the failure is diffuse. Application code builds a flag object inline, `JSON.stringify` runs somewhere in a client library, the request goes out, and a rejection surfaces three layers away as `Error: flag update failed`. Nobody can tell whether the config file was broken, whether the object was missing a field, or whether the release job simply pointed at the wrong environment. Every investigation starts by reconstructing what was actually sent, which is exactly the information nobody logged.

After, there is one narrow path and it reads as a straight line: config text -> parse -> validate -> attribute -> send -> classify. Each arrow can fail in exactly one way, and each failure has a name.

That is the whole trick. Everything below is mechanics.

## A set-and-rollout adapter, in TypeScript

This is deliberately dependency-free so you can paste it into a release script and read every line of it. A schema validator is a fine substitute for the hand-written checks; keep the structure, swap the middle.

```ts
// flag-adapter.ts — the only place in the repo that talks to the flag service.
type Rollout = {
  key: string;
  enabled: boolean;
  rollout: { percentage: number; bucketBy: "account_id" | "user_id" };
  costCenter: string;
};

class PayloadError extends Error {
  constructor(message: string, readonly issues: string[]) {
    super(message);
    this.name = "PayloadError";
  }
}

// Step 1: prove it is JSON at all. This is where a 400 gets caught before it is sent.
function parseConfig(raw: string): unknown {
  try {
    return JSON.parse(raw);
  } catch (cause) {
    const detail = cause instanceof Error ? cause.message : String(cause);
    throw new PayloadError("malformed JSON in flag config", [detail]);
  }
}

// Step 2: prove it satisfies the rules the server enforces. This is the 422 class.
function validateRollout(input: unknown): Rollout {
  const issues: string[] = [];
  const p = (input ?? {}) as Partial<Rollout>;
  const pct = p.rollout?.percentage;

  if (typeof p.key !== "string" || !/^[a-z0-9][a-z0-9._-]*$/.test(p.key)) {
    issues.push(`key: expected a slug, got ${JSON.stringify(p.key)}`);
  }
  if (typeof p.enabled !== "boolean") {
    issues.push(`enabled: expected boolean, got ${typeof p.enabled}`);
  }
  if (typeof pct !== "number" || !Number.isInteger(pct) || pct < 0 || pct > 100) {
    issues.push(`rollout.percentage: expected integer 0-100, got ${JSON.stringify(pct)}`);
  }
  if (typeof p.costCenter !== "string" || p.costCenter.length === 0) {
    issues.push("costCenter: required so spend from this ramp can be attributed");
  }
  if (issues.length > 0) throw new PayloadError("invalid rollout payload", issues);
  return p as Rollout;
}

export async function setRollout(raw: string, traceparent: string) {
  const payload = validateRollout(parseConfig(raw));
  const started = Date.now();

  const res = await fetch("https://flags.internal.example.com/flags/rollout", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      authorization: `Bearer ${process.env.FLAG_API_TOKEN ?? ""}`,
      "idempotency-key": `${payload.key}:${payload.rollout.percentage}`,
      traceparent, // W3C Trace Context: the flag service logs the same trace id
    },
    body: JSON.stringify(payload),
  });

  const text = await res.text();
  if (res.status === 429) {
    // Transport-level, not payload-level: back off and send the same body again.
    const waitMs = Number(res.headers.get("retry-after") ?? 2) * 1000;
    throw new Error(`rate limited, retry ${payload.key} after ${waitMs}ms`);
  }
  if (res.status === 400 || res.status === 422) {
    // Problem details come back as JSON; keep them verbatim, they name the field.
    throw new PayloadError(`rejected ${payload.key} with ${res.status}`, [text.slice(0, 500)]);
  }
  if (!res.ok) throw new Error(`flag service replied ${res.status}: ${text.slice(0, 200)}`);

  console.log(JSON.stringify({
    event: "flag.rollout.set",
    flag_key: payload.key,
    percentage: payload.rollout.percentage,
    cost_center: payload.costCenter,
    status: res.status,
    duration_ms: Date.now() - started,
  }));
  return text.length > 0 ? JSON.parse(text) : null;
}
```

Three details do the actual work here. The parse step is separate from the validation step, so the error message tells you which one failed instead of making you guess. The rejection branch keeps the response body — truncated, but kept — because a service that follows the problem-details convention names the offending field and a service that doesn't at least gives you a string to grep. And the success path emits one structured line with the flag key, the percentage, and the cost center in it, which is the record you will want three weeks later when someone asks why the invoice jumped.

Sending a percentage as a string is the single most common way I've seen this call fail, and it survives every `typeof`-free check because `"25"` looks fine in a log.

## Cost attribution is why the flag key belongs in your telemetry

A rejected payload is a five-minute problem. An unattributed rollout is a month-end problem, and the second one is what the pricing scenario is really about. If a flag gates a pricing rule, then the flag key, the variant, and the ramp percentage are cost dimensions, not debugging trivia.

There are conventions for this. OpenTelemetry publishes semantic attributes for flag evaluation — flag key, provider name, variant — so evaluations can be recorded as span events rather than as ad-hoc log strings, and W3C Trace Context defines the `traceparent` header that carries a shared trace id across the release script, the flag service, and the billing pipeline. Adopt the attribute names even if you never adopt a tracing backend; they cost nothing and they stop each team from inventing `flagName` versus `flag_key` versus `feature`.

The rule I'd apply for a cost-attributed ramp: log the flag key on the write, log it on every evaluation that changes a billable path, and make sure both carry the same trace id and the same cost center string. Then a spend anomaly resolves to a specific rollout stage instead of a shrug.

One caution about volume. Recording an event for every flag evaluation on a hot path can produce more telemetry than the feature itself produces revenue, so sample evaluations and keep writes unsampled. Writes are rare and interesting. Evaluations are frequent and mostly identical.

## Should you validate on the client, and should you ever retry a 422?

The retry question is easy. A 422 is deterministic: the same bytes will be rejected the same way forever, so retrying just multiplies the log noise. Retry the transport failures — connection resets, 429s with a `Retry-After`, 503s — and route the payload rejections to a human or a failing CI step. Anything that mixes those two paths into one exponential backoff loop will eventually hammer an endpoint 200 times with a body that was invalid the first time.

Client-side validation deserves more hedging. Duplicating the server's rules in your own code creates a second source of truth, and the catch is drift: your validator stays on the old shape, the service adds a required field, and now your adapter cheerfully sends payloads that fail on the wire while your unit tests stay green. Fetch the published schema at startup if the service exposes one, pin it deliberately if you'd rather review changes, and either way check it in CI rather than trusting a hand-maintained interface. If the flag service is unambiguously the source of truth and your release path is one script run by one team, local validation isn't a good fit — stick with sending the request and reading the problem details, and spend the effort on logging instead.

There's a third position I hold less firmly. Some teams put flag configuration in version control and validate it in a pre-merge check, which catches malformed JSON before it ever reaches a release script; that works well for slow-moving pricing rules and badly for flags that operators flip during an incident. Your mileage may vary depending on who owns the toggle.

Whatever you pick, write down the smallest useful record for each attempt: operation, flag key, schema version, status, trace id, duration. Not the targeting rules, which may carry user attributes. That one line is the difference between troubleshooting and archaeology.

## Further reading

- HTTP Semantics, RFC 9110 (status code definitions, including 400 and 422): https://www.rfc-editor.org/rfc/rfc9110
- The JSON data interchange syntax, RFC 8259: https://www.rfc-editor.org/rfc/rfc8259
- Problem Details for HTTP APIs, RFC 9457: https://www.rfc-editor.org/rfc/rfc9457
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- OpenTelemetry feature flag semantic conventions: https://opentelemetry.io/docs/specs/semconv/feature-flags/
- OpenFeature specification: https://openfeature.dev/specification/
- JSON Schema validation vocabulary: https://json-schema.org/draft/2020-12/json-schema-validation
- MDN, 422 Unprocessable Content: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/422
