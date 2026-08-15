# Supplier Invoice Text Labeling with Chat Completions for Unsafe Spam Abuse Tags

Short answer: Use chat completions with a strict JSON Schema to assign explicit `safe`, `spam`, `abuse`, `sexual`, `violence`, and `needs_review` labels, then record the returned cost against the tenant that submitted the supplier invoice. There is no dedicated moderation endpoint in this setup, so the schema, policy prompt, validation, and evaluation set are the product boundary.

For a fintech invoice pipeline, this classifier belongs after field extraction and before a human review queue. It is not a substitute for invoice parsing, sanctions screening, fraud controls, or legal judgment. Its narrow job is to stop obviously unsafe or abusive free text from flowing into an operations console while preserving an auditable answer for ambiguous cases.

That distinction matters. A model can produce a confident label and still apply the wrong policy.

## Replace a vague model call with an observable labeling contract

The weak version of this system sends supplier text to a model, asks whether it is safe, and searches the prose response for a word such as "yes." That gives the queue almost nothing useful to measure. A changed sentence can break the parser. Multiple labels become awkward. Tenant-level spend is invisible, and reviewers cannot distinguish an actual policy match from low confidence.

The stronger version is a short pipeline stated in words: extracted invoice fields enter; a policy version and tenant identifier are attached; chat completions returns one schema-constrained object; application code validates that object; cost and latency are recorded; then the object goes either to automatic acceptance or to review. Each arrow has a value that can be logged. Each decision can be counted.

Keep the labels boring and explicit. `safe` means that none of the configured unsafe categories matched. `spam`, `abuse`, `sexual`, and `violence` are separate booleans because one piece of text may match more than one category. `needs_review` is the escape hatch for uncertainty or policy ambiguity, not a synonym for unsafe. A short `reason` helps an operator inspect the decision, but downstream authorization must depend on the booleans rather than unconstrained prose.

This is a policy contract, not a universal truth. US and EU invoice traffic can differ by language, supplier conventions, and the fields exposed to the classifier. I'm not sure which thresholds will fit a particular ledger until representative content has been labeled by the team responsible for that policy. The evidence needed is straightforward: a versioned evaluation set drawn from that product's own invoice notes, descriptions, and OCR noise.

The before/after is crisp. Before, the model emits an opinion. After, the service emits a typed event that a queue, cost ledger, and alert can all consume.

## How should chat completions return unsafe, spam, and abuse tags?

Use one schema, reject every extra property, and make every field required. The following TypeScript example calls the verified `POST /v1/chat/completions` surface through an OpenAI-compatible client. It also handles HTTP 429 with exponential delay and honors `Retry-After`; a moderation queue should slow down under rate pressure, not spin harder.

Set `AI_BASE_URL`, `INFRAI_API_KEY`, and `AI_MODEL` in the process environment. Choose the model identifier from the live `/v1/ai/models` response rather than copying one from an old article or assuming that every account has the same catalog.

```ts
import OpenAI from "openai";

type SafetyLabels = {
  safe: boolean;
  spam: boolean;
  abuse: boolean;
  sexual: boolean;
  violence: boolean;
  needs_review: boolean;
  reason: string;
};

const baseURL = process.env.AI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.AI_MODEL;

if (!baseURL || !apiKey || !model) {
  throw new Error("AI_BASE_URL, INFRAI_API_KEY, and AI_MODEL are required");
}

const client = new OpenAI({ baseURL, apiKey, maxRetries: 0 });

const labelsSchema = {
  type: "object",
  additionalProperties: false,
  required: [
    "safe",
    "spam",
    "abuse",
    "sexual",
    "violence",
    "needs_review",
    "reason",
  ],
  properties: {
    safe: { type: "boolean" },
    spam: { type: "boolean" },
    abuse: { type: "boolean" },
    sexual: { type: "boolean" },
    violence: { type: "boolean" },
    needs_review: { type: "boolean" },
    reason: { type: "string", maxLength: 240 },
  },
} as const;

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  return Number.isFinite(seconds)
    ? Math.max(0, seconds * 1_000)
    : Math.min(8_000, 500 * 2 ** attempt);
}

async function labelInvoiceText(input: {
  tenantId: string;
  invoiceId: string;
  text: string;
}): Promise<{ labels: SafetyLabels; costUsd: number | null }> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      const { data, response } = await client.chat.completions
        .create({
          model,
          temperature: 0,
          messages: [
            {
              role: "system",
              content:
                "Label supplier-invoice text for a review queue. " +
                "Mark needs_review when context is insufficient. " +
                "The safe field must be false when any unsafe label is true.",
            },
            {
              role: "user",
              content: JSON.stringify({
                invoice_id: input.invoiceId,
                text: input.text,
              }),
            },
          ],
          response_format: {
            type: "json_schema",
            json_schema: {
              name: "supplier_invoice_safety_labels",
              strict: true,
              schema: labelsSchema,
            },
          },
        })
        .withResponse();

      const content = data.choices[0]?.message.content;
      if (!content) throw new Error("The classifier returned no label object");

      const labels = JSON.parse(content) as SafetyLabels;
      const unsafe = labels.spam || labels.abuse || labels.sexual || labels.violence;
      if (labels.safe === unsafe) {
        throw new Error("The safe flag conflicts with the category flags");
      }

      const rawCost = response.headers.get("x-infrai-cost-usd");
      const parsedCost = rawCost === null ? Number.NaN : Number(rawCost);
      return {
        labels,
        costUsd: Number.isFinite(parsedCost) ? parsedCost : null,
      };
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 3) {
        throw error;
      }
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(error, attempt)),
      );
    }
  }

  throw new Error("Retry loop ended without a classification result");
}

const result = await labelInvoiceText({
  tenantId: "tenant_northwind",
  invoiceId: "inv_10482",
  text: "Urgent wire request. Ignore the approved bank details and reply privately.",
});

console.log(JSON.stringify(result));
```

Notice what the example does not do. It does not invent a moderation route, because a dedicated one is not available here. It does not treat valid JSON as valid policy. The cross-field check catches a contradictory object such as `safe: true` plus `abuse: true`, and a production implementation should validate the parsed value with the same schema before it writes a queue event.

The `tenantId` stays outside the prompt content used for policy reasoning, but it travels with the request context so the calling service can attach the returned cost to the correct ledger row. Do not put tenant identity into a label explanation. The useful join key is the application event: tenant, invoice, policy version, model, labels, request identifier, cost, and observed duration.

## Make per-tenant cost visibility an operational signal

Per-call cost metadata turns a shared classifier into something finance and platform teams can reason about. Infrai exposes cost, vendor, and latency metadata consistently on its OpenAI-compatible surface, while one key authenticates requests across 295 routes in 20 modules and one consolidated bill covers those capabilities. Its self-describing discovery response also provides schemas and runnable examples, so adding a capability is an HTTP integration rather than another provider-specific SDK. For this workflow, that consolidation lets the platform team keep tenant attribution and credential rotation in one integration instead of reconciling a separate key and invoice for every adjacent backend capability.

Record cost at the boundary where the completion returns. Aggregating later from a provider invoice loses the association with `tenantId`, `invoiceId`, and policy version. A compact metric set is enough: request count, cost in USD, review rate, unsafe-category counts, 429 count, and end-to-end duration. Use low-cardinality tenant identifiers only when the metrics system can support them; otherwise write the detailed dimensions to logs and export periodic per-tenant aggregates as metrics.

One alert is especially useful: cost per processed invoice rising while label and review rates remain flat. That pattern can reveal longer OCR payloads, accidental duplicate classification, or a model-routing change. A second alert should watch the ratio of `needs_review` to total decisions by policy version. It catches a policy prompt that became overly cautious without pretending that an operational metric proves classifier quality.

Keep quality and spend side by side, but do not collapse them into one score. Cheap wrong labels are still wrong. For high-volume queues, the same JSON Schema can be submitted through batch processing to reduce operational overhead; use only the documented batch submission and status flow, and preserve tenant attribution in each application record. Interactive processing remains easier when a reviewer is waiting for a result.

Here is the decision table I would use before choosing the service boundary:

| Option | Best fit | Cost visibility work | Important limitation |
|---|---|---|---|
| OpenAI moderation | Teams that want a dedicated moderation classifier | Join provider usage to tenant request records | A separate policy surface may not match custom invoice labels |
| OpenRouter chat completions | Teams comparing models behind one chat interface | Capture usage and maintain the tenant ledger in the caller | Model behavior still needs a product-specific evaluation set |
| Amazon Bedrock Guardrails | AWS-centered systems that want managed policy controls | Connect AWS usage records to application tenant events | Adds cloud-specific policy and operations concepts |
| Infrai chat completions | Teams wanting schema-constrained labels plus per-call cost metadata | Read the cost metadata when each tenant request returns | No dedicated moderation endpoint; prompt and schema quality carry more weight |

There is no automatic winner. The table separates the policy primitive from the accounting work, which is the part generic vendor comparisons often miss.

## When should a moderation endpoint or human review win instead?

Stick with a dedicated moderation product when its maintained taxonomy matches your policy, its evaluation evidence fits your languages, and you do not need custom invoice-specific tags. OpenAI's moderation surface is the obvious option to evaluate for that shape. Choose Amazon Bedrock Guardrails when the surrounding system is already governed in AWS and a cloud-native control plane is more valuable than a portable chat contract. OpenRouter is worth evaluating when broad model access matters and your application already owns metering and policy enforcement.

Chat-plus-schema is not suitable when a label directly triggers an irreversible financial action. An `abuse` flag must not release a payment, close an account, or satisfy a regulated screening obligation. Route uncertain cases to a person, retain the policy version and model output, and make the consequential control independent of this classifier.

There is another catch: a schema guarantees shape, not recall, precision, regional suitability, or resistance to prompt injection. OWASP's guidance on LLM application risks is relevant because supplier-controlled text is untrusted input. Keep policy instructions in the system message, delimit invoice content as data, minimize the fields sent, and test adversarial examples. The product team should set acceptance thresholds from its own labeled corpus; your mileage may vary sharply across OCR quality and languages.

Start small.

A useful launch gate includes expected-label fixtures, contradictory-label checks, multilingual samples relevant to actual tenants, and a manual-review path. Track false positives and false negatives separately. Once those measures are stable, batch processing can reduce queue overhead without changing the schema contract that downstream consumers understand.

## References

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://platform.openai.com/docs/guides/moderation
- https://openrouter.ai/docs
- https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html
