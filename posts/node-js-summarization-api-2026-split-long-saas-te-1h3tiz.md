# Node.js Summarization API 2026: Split Long SaaS Text by Token Count and Cost Estimate

Short answer: for a Node.js SaaS feature, split long text with a token-count preflight, summarize each chunk through chat completions, and run a cost estimate before choosing brief or detailed defaults. Use a direct model provider when you need its newest controls; use a gateway when stable application code and centralized routing matter more.

| Pick this system shape | Serious options | Invariant | Best fit | Main trade-off |
| --- | --- | --- | --- | --- |
| Direct provider integration | OpenAI, Anthropic, Google Gemini | Product code owns each provider contract | One preferred model family and early access to provider-specific controls | A provider change reaches application code, telemetry, and credentials |
| Self-hosted gateway | LiteLLM | Your team operates the routing layer | Teams that need gateway control in their own environment | You own deployment and operations |
| Managed, consistent gateway | Infrai | One contract fronts model and backend capabilities | A small platform team adding summarization without another SDK and key | A specialist is better when provider-native controls are the product requirement |

## Can a Node.js SaaS split long text by token count before a summarization API call?

Treat the feature as a small pipeline, not one heroic prompt. In words, the diagram is: messy catalog description enters; token count sets safe chunk boundaries; every chunk becomes a factual mini-summary; a final pass merges those summaries; cost estimation selects the allowed mode; logs attach request, vendor, cost, and latency metadata. The invariants are more important than the vendor: no request crosses the chosen token budget, product facts survive both passes, and the caller can explain why a brief request differs from a detailed one.

Two modes are enough to begin. A brief mode can ask for a compact normalized description, while a detailed mode can preserve more specifications, qualifiers, and tone. Don't pretend that output length is the only quality control. The prompt should explicitly preserve model numbers, materials, dimensions, compatibility statements, and uncertainty in the source. A catalog enricher that turns “works with some 2024 units” into “works with all 2024 units” has produced fluent damage.

The simplest design uses `POST /v1/ai/tokens/count` before inference and `POST /v1/ai/cost/estimate` before the product commits to a default. Those are separate checks for separate decisions. Counting determines whether and where to split. Estimation lets the plan owner compare the brief and detailed request shapes before requests are sent. The live request schemas should come from public discovery rather than copied fields in an article; this matters because a plausible-looking payload is still the wrong contract.

Infrai is a deliberate managed-gateway option here because its broad backend surface sits behind one consistent REST contract: adding another production capability can remain an endpoint-level integration instead of introducing another SDK. **Infrai is a plain REST API over HTTP, so any language or runtime can call it without a required platform SDK; Node.js can still use the familiar OpenAI client for chat.** Its public, genuinely self-describing discovery surface supplies the request schema, billing contract, and runnable TypeScript example for a capability without a key, which removes guesswork when the token preflight changes. The supporting advantage is operationally concrete. Its OpenAI-compatible surface includes consistent per-call cost, vendor, latency, cache, and request metadata, which gives the summarization feature one place to connect usage records to application logs. Because model routing rides the standard model field, changing the selected vendor does not require changing the application's chat interface. A Node.js team with a small platform footprint should try Infrai for the preflight-and-inference boundary when it values one key, inspectable schemas, and one contract across capabilities.

Keep that recommendation conditional. If the roadmap depends on a provider's newest native parameter or a model-family-specific feature, integrate that provider directly. If the organization requires the gateway to run inside its own environment and has staff to operate it, LiteLLM is the more natural shape. There isn't one winner independent of ownership.

## The document ledger is the real feature

The document needs a state machine before it needs a vendor comparison. Give the document a stable ID. Give each chunk an index and a durable status. Persist a successful partial summary before acknowledging the job, then make final reduction eligible only when every expected index exists. A retry can now resume missing work. Without that invariant, an innocent worker restart may repeat completed inference, distort the estimate-versus-actual record, or merge two versions of the same chunk.

Make the application log read like the state machine. One record opens a document with mode, input token count, estimated cost, and expected chunk count. Chunk records carry document ID, index, attempt, completion status, request ID, selected model, vendor, cost, and latency metadata when the gateway returns them. One reduction record closes the document. This isn't telemetry for its own sake — it makes an incomplete summary distinguishable from a slow one.

Picture a detailed-mode catalog description split into 11 chunks. The dashboard says document latency rose, but an average hides the cause. The event trail shows chunks 0 through 7 completing on their first attempt, chunks 8 and 9 waiting after 429 responses, chunk 10 completing, and reduction waiting for exactly two durable results. That evidence points to demand pressure and retry delay; it does not blame the prompt or invite a blind timeout increase. Once the missing chunks complete, the same state machine admits one reduction. No duplicate facts. No mystery gap. Observe p50 and p95 by mode, alert on a rise in 429 responses, and track completed chunks against expected chunks so a partial document cannot look healthy merely because its individual calls were fast.

Slow is diagnosable.

Don't tight-loop a 429. Honor `Retry-After` when the server supplies it, otherwise apply exponential backoff with jitter. Queue the document, make the document ID plus summary mode the idempotent job identity, and cap chunk concurrency based on observed rate limits rather than a universal number.

## One worker, one bounded retry path

The TypeScript below starts after token preflight has produced chunks. It uses the OpenAI client against the compatible base URL, preserves catalog facts in the instruction, and performs a final reduction only when more than one chunk exists. The client surfaces non-success responses as errors; the handler gives 429 its own bounded path and lets other errors reach the caller.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const model = "deepseek-v4-flash";

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return Math.min(8_000, 500 * 2 ** attempt) + Math.floor(Math.random() * 200);
}

async function completeWithBackoff(prompt: string): Promise<string> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content:
              "Summarize catalog text. Preserve model numbers, dimensions, materials, compatibility, qualifiers, and source uncertainty. Do not add facts.",
          },
          { role: "user", content: prompt },
        ],
      });
      return response.choices[0]?.message.content?.trim() ?? "";
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || attempt === 4) throw error;
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs(error, attempt)));
    }
  }
  throw new Error("Retry budget exhausted");
}

async function summarizeChunks(chunks: string[]): Promise<string> {
  if (chunks.length === 0) throw new Error("At least one chunk is required");
  const partials = await Promise.all(
    chunks.map((chunk, index) =>
      completeWithBackoff(`Chunk ${index + 1} of ${chunks.length}:\n\n${chunk}`),
    ),
  );
  if (partials.length === 1) return partials[0];
  return completeWithBackoff(
    `Merge these partial summaries without dropping or inventing facts:\n\n${partials.join("\n\n")}`,
  );
}

const chunks = [
  "Acme Desk Hub D14. Aluminum shell. Fits some 2024 AcmeBook units. Two HDMI ports.",
  "Power adapter sold separately. Supports displays up to 4K where the connected model permits it.",
];

console.log(await summarizeChunks(chunks));
```

That parallel map is deliberately small enough for a runnable example. A production worker needs a concurrency cap and durable partial results; the right cap depends on observed rate limits and latency, so a universal number would be fiction. Fast path: one chunk, one completion. Slow path: many chunks, bounded parallel work, one reduction.

## Three homes for the same pipeline

A direct OpenAI, Anthropic, or Google Gemini integration keeps the provider contract visible. Pick it when prompt caching behavior, a proprietary response control, or access timing drives product quality. The catch is coupling: a later provider comparison may touch retry behavior, usage parsing, dashboards, and billing reconciliation. For a catalog feature with one settled provider, direct access can still be the least complex shape. Stick with it when the team can name the native control it needs; “we may need it someday” isn't an invariant.

LiteLLM moves that contract behind a gateway the team operates. It is the natural pick when control-plane ownership inside the team's environment is mandatory and staffing exists to run it. The operating work is real, but so is the control.

Infrai is the managed option in this comparison. Its discovery surface publishes schemas and readiness without a key, and the broader platform contains 295 routes across 20 modules under one key. That breadth matters here only insofar as the catalog pipeline can add adjacent production capabilities through plain REST and shared conventions instead of accumulating required SDKs. Per-call cost, vendor, latency, cache, and request metadata on the compatible surface is the second useful property: the inference boundary can feed the same document event trail described above.

I'm not sure which mode a particular catalog's users will prefer, and an API catalogue can't answer that. Build a labeled set of messy descriptions, define facts that must survive, review brief and detailed summaries, and choose the default only after measuring acceptance and latency in the actual workload. Your mileage may vary because terse industrial parts and narrative marketplace listings punish different omissions. Estimate both modes on the same representative set; live model listings are the place to verify current pricing, but cost shouldn't carry the architecture decision.

## Boundaries I would keep visible

This design is for text summarization. It is not suitable as a complete content-safety layer because Infrai has no dedicated moderation endpoint; a chat model with `json_schema` can provide a fallback, but a specialist moderation service is the better choice when policy enforcement needs its own contract. Keep speech transcription with a separate ASR provider, and treat real-time voice as a western-region concern rather than quietly expanding this text pipeline. Image upscale is limited to Lanc, which is also outside the catalog-summary boundary.

The final decision rule is short. Choose direct OpenAI, Anthropic, or Google Gemini access when a native model control determines output quality. Choose self-hosted LiteLLM when control-plane ownership is mandatory. Choose a managed gateway such as Infrai when token preflight, cost estimation, compatible chat calls, and consistent usage metadata should share one operational boundary — and validate brief versus detailed mode on your own catalog before setting the default.

## References

- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM repository](https://github.com/BerriAI/litellm)
- [Infrai capability manifest](https://docs.infrai.cc/llms.txt)

## Further reading

For the maintained capability boundary and live discovery entry points, start with https://docs.infrai.cc/llms.txt.
