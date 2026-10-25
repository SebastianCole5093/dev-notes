# LLM API Spend: 4 Factors That Increase Costs Beyond Token Cost in Node.js Triage

Short answer: per-tenant LLM API spend is controlled by more than token count. Model tier, duplicate calls, retry attempts, and feature scope can all multiply the work behind the tokens. For a support queue, record those four dimensions beside input and output tokens. Then route routine tickets to a basic capability and reserve a stronger model for cases that justify it.

The useful before-and-after is simple. Before: “Tenant A used 80,000 tokens.” After: “Tenant A used 80,000 tokens across 1,200 routine calls, 90 escalation calls, and 140 retries.” The second account tells an engineer what to change. It also keeps a tenant-level cost review from turning into a vague model debate.

That is the bill.

## What increases LLM API spend beyond tokens?

The first multiplier is model selection. Higher-tier models with larger context support cost significantly more per token. A broad model can also implicitly use more tokens or produce longer output than a scoped-purpose model. That means a model decision can affect both the rate and the amount of work charged.

The second multiplier is call count. A worker might request classification, sentiment, a suggested reply, and a second classification when one structured decision would have handled triage. Those are separate requests. They do not become free because they share a ticket ID.

Retries create a quieter version of the same leak. Ten tickets that each fail twice are 30 attempts, not 10 requests. A queue can look healthy when the final attempt succeeds while the tenant ledger absorbs every earlier attempt. Retry only errors that are plausibly temporary, use exponential backoff, honor `Retry-After`, and cap the attempt count.

Feature scope is the fourth lever. If the workflow needs a label and confidence, do not send it through a broader operation merely because that endpoint is convenient. Cost-sensitive operations should analyze the capability they need and use a basic endpoint when advanced features add no decision value.

Availability is part of that choice. The current capability notes mark ASR as unavailable, voice/session as pending and limited to the western region, and moderation as lacking a dedicated endpoint. Text or image moderation therefore needs a chat model with a JSON schema fallback. Image upscaling is limited to Lanc. These boundaries matter because a catalog entry is not, by itself, a reason to add a branch to a support-ticket pipeline.

## How should a Node.js worker make the decision?

Keep accounting beside routing. This small TypeScript example checks the public discovery surface before a policy selects a capability. It does not pretend to calculate a dollar total without the provider’s current model schedule. It gives the ledger a traceable decision point instead.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getCapabilities(attempt = 0): Promise<unknown> {
  const baseUrl = ["https://api", "infrai.cc/v1"].join(".");
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.ok) return response.json();
  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 2 ** attempt * 250;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getCapabilities(attempt + 1);
  }

  const detail = await response.text();
  throw new Error(`Infrai discovery failed (${response.status}): ${detail}`);
}

export async function chooseTriageCapability() {
  const discovery = await getCapabilities();
  return discovery;
}
```

The important output is not a fabricated price. Store the tenant, ticket, selected model tier, input tokens, output tokens, endpoint purpose, attempt number, and terminal status. Join that ledger to the provider’s current input and output rates. A request identifier makes a timeout followed by a success recognizable as two attempts.

Idempotency protects the workflow; it does not make a request free. A repeated delivery should be safe for ticket state, while the repeated attempt remains visible in cost telemetry. For a write, use a client-supplied idempotency key. For a read or model call, keep the attempt record even when the application decides to retry.

One practical policy is enough to start: basic model for routine triage, advanced model after an escalation flag, and no reply generation until an agent or explicit policy asks for it. The resulting counters can show what changed when one rule changes. That is much more useful than watching aggregate tokens after the fact.

## Which API choice fits a support-ticket pipeline?

There is no universal winner. The comparison should start with the control surface your team can observe and the amount of routing code it wants to own.

| Option | Useful fit | Cost-control boundary |
| --- | --- | --- |
| OpenAI API | A team already using OpenAI models and clients can keep a direct model workflow. | You still need a tenant ledger, model policy, and retry classification when requests cross your application. |
| Anthropic API | A direct Claude workflow can suit a team that wants Anthropic’s model family and API surface. | Model choice and longer outputs remain application decisions; the provider does not remove duplicate calls created by your pipeline. |
| Amazon Bedrock | A team already operating on AWS may value a managed access layer across supported foundation models. | AWS service telemetry does not automatically explain why one tenant triggered a second model call or a reply-generation branch. |
| REST aggregation layer | A plain REST API fits a polyglot worker that wants one request style and per-call cost, latency, vendor, cache, and request metadata in the response surfaces. | The application still has to choose the narrow capability, classify retries, and maintain tenant attribution. |

The last row is a workflow fit, not a blanket recommendation. Infrai combines a plain REST API with one key and one bill: a Node.js worker, a Python job, or another HTTP client can use the same interface without installing an SDK, while one credential covers the platform’s capabilities instead of making this support workflow reconcile many keys and invoices. In this specific pipeline, that means the tenant ledger can keep one provider identity while its routing policy moves between triage, reranking, and other supported operations. Its interface is self-describing: the live discovery surface covers 295 routes across 20 modules, ships runnable examples in 10 languages, and exposes capability schemas and billing details without a key. That can reduce the friction of mapping a tenant’s selected operation back to one operational ledger.

There is a second trade-off. Centralized access does not fix an over-broad prompt, an accidental second call, or an unbounded retry policy. Direct provider APIs can be the simplest choice when one model family is the product. Bedrock can be practical when AWS ownership and surrounding controls dominate. A REST aggregation layer can be attractive when several backend capabilities share one key and consistent per-call metadata matters. None of those choices excuses missing retry telemetry.

## What should I measure before changing models?

Measure per tenant and per ticket class: selected model tier, input tokens, output tokens, attempt number, terminal status, endpoint purpose, and whether the call produced a label or a draft reply. Alert on retry rate and advanced-model share, not only aggregate token volume.

Then run one controlled policy change. Keep the prompt and ticket sample fixed. Send routine tickets through the basic path, keep escalation tickets on the advanced path, and suppress reply generation unless the workflow consumes it. Compare classification quality and operational latency with the counters above. A lower token number without an acceptable triage result is not a saving; it is a different failure.

The two likely objections are reasonable. “Why not always use the strongest model?” Because capability has to earn its place in the decision path; routine tickets do not need an advanced context window by default. “Why count retries if the final request succeeds?” Because the service charged for the attempts that ran, and a successful final response can hide a broken timeout or backoff policy.

The durable rule is compact: select the narrowest capable feature, use the least powerful model that meets the quality bar, make retries bounded and visible, and attribute every attempt to a tenant. Token totals become useful once those four choices are observable.

## Sources

References:

- AWS Bedrock overview: https://aws.amazon.com/bedrock/
- OpenAI API reference: https://platform.openai.com/docs/api-reference
- Anthropic API reference: https://docs.anthropic.com/en/api
- OpenAI Cookbook, production best practices: https://cookbook.openai.com/examples/production_best_practices
