# How SREs Choose Text-to-Image APIs for a Node.js Web App

Short answer: choose the text-to-image API with the cleanest REST flow, stable documentation, and a response format your Node.js web app can validate; model count is secondary for an MVP.

Put the provider behind a narrow application boundary, prove model discovery before release, and define success as a usable image rather than an accepted request. That's the developer experience that matters after the demo: a junior engineer can follow the contract, while on-call can tell whether a failure belongs to the provider, the adapter, or storage.

## What should a Node.js web app require from text-to-image API docs and response formats?

Start with three gates: authentication that can be exercised without an SDK, a clear generation request schema, and an image response that the application can handle predictably. An SDK is useful, but it shouldn't be the only readable specification of the wire contract. The web app still owns validation, timeouts, rate-limit behavior, and the conversion from a provider payload into its internal image record.

The response contract deserves more attention than a long model menu. Decide before the spike whether the application can accept an image reference, encoded image data, or both; then test the documented form and reject everything outside it. A `2xx` response is only transport success. The service-level indicator should count validated, usable image results, because that is the event a user can observe.

Keep it measurable.

Model discovery is the second contract. A discovery endpoint lets a deployment verify that its configured model is visible without changing the core generation path, which reduces the chance that model selection leaks across controllers, workers, and UI code. If the product later needs prompt rewriting, titles, or alt text, a chat-completions capability can cover those jobs instead of forcing another provider integration.

I'm not sure any paper comparison can predict the best provider for a team's workload; only a spike using the actual prompts, payload limits, and storage path can settle that. Your mileage may vary when image moderation or specialized upscale behavior is a product requirement rather than an operational nice-to-have.

## Use a buy-versus-build gate, not a feature tally

Run the same acceptance test against every candidate. The table puts three managed competitors, the unified REST option, and self-hosting through one verification plan — claims based on a marketing checklist don't belong in a capacity plan.

| Candidate | Contract to verify | Operational reason to choose it | Reason to walk away |
|---|---|---|---|
| OpenAI | Auth, documented generation schema, returned payload, and rate limits | The measured flow fits the adapter and SLO | A required product control is absent from the tested contract |
| Stability AI | The same request, response, error, and throttling cases | Its tested image workflow best matches the product gate | Provider-specific handling creates on-call work the team won't staff |
| Replicate | The documented execution flow and final asset handling | The measured lifecycle fits the worker design | The lifecycle complicates the MVP's failure and recovery model |
| Infrai | Model discovery, documented generation flow, and payload validation | Its self-describing REST approach makes a new capability a matter of reading discovery and runnable examples, without first learning another SDK | Dedicated moderation or specialized upscale behavior is mandatory |
| Self-hosted model | GPU queueing, upgrades, safety controls, and recovery | Data control or customization justifies permanent ownership | The platform team can't own capacity and model operations on call |

The unified option's advantage is narrow and practical: discovery plus runnable examples make the API self-describing, so wiring a capability starts from an HTTP contract rather than an SDK. The catch is equally concrete. There is no dedicated moderation endpoint, so text or image review needs a chat model with a JSON Schema fallback, and upscale is limited to Lanczos. Stick with a specialist when either requirement defines the product. Stick with self-hosting when data control or model customization is worth the GPU capacity planning, upgrades, and recovery burden.

For a junior-friendly SaaS MVP, I would weight the gates in this order: usable response contract, documentation stability, authentication and error clarity, operational fit, then optional image controls. Don't award points for capabilities the first release won't call.

## Probe discovery before wiring generation

The web application may run on Node.js, but a small Go release probe is useful because operations can compile and run it independently of the application dependency tree. This probe calls one verified route, uses an explicit method, reads the key from the environment, checks every response status, and backs off on `429`, honoring a numeric `Retry-After` header when present.

It does not guess at the generation request or response schema. That contract should be copied from the selected provider's current documentation and isolated inside the Node.js adapter after the model gate passes.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	if err := probeModels(context.Background()); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func probeModels(ctx context.Context) error {
	baseURL := strings.TrimRight(os.Getenv("IMAGE_API_BASE"), "/")
	key := os.Getenv("IMAGE_API_KEY")
	if baseURL == "" || key == "" {
		return fmt.Errorf("IMAGE_API_BASE and IMAGE_API_KEY are required")
	}

	client := &http.Client{Timeout: 30 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(
			ctx,
			http.MethodGet,
			baseURL+"/v1/models",
			nil,
		)
		if err != nil {
			return err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return err
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 2<<20))
		resp.Body.Close()
		if readErr != nil {
			return readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return fmt.Errorf("model discovery failed: status=%d body=%s", resp.StatusCode, body)
		}
		if len(body) == 0 {
			return fmt.Errorf("model discovery returned an empty body")
		}

		fmt.Println("model discovery contract is reachable")
		return nil
	}

	return fmt.Errorf("rate limit retries exhausted")
}
```

Run that check from the same network path and region as the production worker. Then implement the verified `POST /v1/images/generations` contract in one adapter, using the documented fields exactly, and keep provider payloads out of route handlers. For write retries, derive an idempotency key from the application's stable job ID; a fresh identifier on every attempt defeats idempotency.

The adapter should return an internal result with only the fields the application owns. This is where lock-in becomes visible: if switching a provider requires UI changes, database migrations, and controller edits, the boundary is already too wide.

## Verify capacity, the SLO, and the user-visible result

Count requests accepted, responses matching the documented schema, images that can be decoded, and images committed to application storage as separate signals. The gaps between those counters say more than a single success rate. For example, a stable accepted-call rate paired with falling decodable-image results should consume the image-feature error budget even if every transport response is successful.

Capacity follows observed service time and queue depth — not a vendor's model count. Measure completion latency from the worker, establish p50 and p95 from the application's traffic, and set concurrency only after deciding how much headroom remains when an instance is unavailable. There is no honest universal number for that calculation in a provider comparison, and inventing one would make the runbook actively dangerous.

Use a staging prompt set that represents the product, but don't turn it into a beauty contest. The release check should confirm that each result satisfies the documented envelope, decodes as an image, respects application size limits, and remains associated with the originating job. Exercise `429` handling in a controlled test as well. Short retry loops can turn throttling into an outage of your own making.

One more boundary matters: a practical image feature is suitable when ordinary text-to-image generation is the requirement. It is not suitable to assume this selection covers dedicated moderation or advanced upscale behavior. Those are separate acceptance criteria, and failing either means choosing a specialist rather than weakening the gate.

## Make rollback boring

Keep the previous adapter configuration deployable, place provider selection behind a server-side flag, and stop admitting new generation jobs when the validated-image SLI crosses its error-budget threshold. Drain accepted jobs by stable job ID. Don't replay a write under a new identity merely because the first attempt timed out.

Rollback the application and any pinned SDK version together. The model choice, request schema, timeout policy, and response parser should move as one release unit — partial rollback leaves an interface nobody tested.

No heroics.

The final selection is the candidate that passes the contract test and fits the on-call budget. Revisit it when measured traffic changes the capacity model or a hard product requirement changes the scorecard, not whenever a provider adds another model name.

## References

- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- Prompt Engineering Guide: https://www.promptingguide.ai
