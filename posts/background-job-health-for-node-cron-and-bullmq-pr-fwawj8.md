# Background Job Health for node-cron and BullMQ — Pricing Rollback Guardrails

TL;DR: A Node.js background job running in node-cron or BullMQ needs an external heartbeat monitor to detect a missed start. Send the heartbeat only after the pricing-rule evaluation succeeds, then use a success metric and structured logs to explain failures. For an edtech rollout where rollback safety matters, Healthchecks.io is the least complex default. Pick another tool only when its response path fits your team better.

| Pick | Best fit for this rollout | Boundary to test before committing |
|---|---|---|
| Healthchecks.io | A focused external heartbeat for a scheduled evaluator | Confirm its notification and hosting choices match your incident process |
| Cronitor | Teams evaluating a dedicated cron-monitoring workflow | Validate the check-in and escalation flow with one intentionally missed run |
| Better Stack Heartbeats | Teams that want heartbeat checks alongside their existing Better Stack monitoring | Check how the heartbeat becomes an actionable page in your setup |
| Sentry Crons | Teams already using Sentry and wanting scheduled-job check-ins in that workflow | Decide whether job health belongs with application errors for your responders |
| Infrai metrics and logs plus your own poller | One key and one bill across backend services matter more than a managed heartbeat workflow | It has no built-in heartbeat or notification route, so silent non-starts still require an external tool |

The table separates two signals that are easy to blur together. A completion metric proves the evaluator finished. A remote deadline proves that the expected completion arrived on time. I prefer keeping both because they answer different incident questions.

## How should node-cron and BullMQ background jobs report a heartbeat?

A process that never starts writes no start log, no error, and no stack trace. Silence looks exactly like a healthy, quiet system if the monitor only searches emitted events. That is the dangerous failure during a flag rollout: schools may continue receiving the previous pricing rule while the team assumes the new evaluation ran.

Silence wins.

Use a deadline outside the scheduler's failure domain. Configure the external monitor around the real schedule plus a deliberate grace window. For a job scheduled every 5 minutes, for example, the team might decide that the tenth minute is the rollback boundary; that interval is a policy choice, not a measured universal default. Then ping after the full evaluation succeeds, never at job start. An early ping turns a half-finished run into a false success, which is the specific trap this design avoids.

The internal trail still earns its keep. Emit a timestamp metric or increment a success counter after completion. Log `job_started`, `job_finished`, and `job_failed` with a stable job ID. Those events answer a different question: did the job start and then fail, time out, or finish? Avoid putting student or customer details in those fields; data minimization applies to operational telemetry too.

## Pick this when the response path matters

Choose Healthchecks.io when the requirement is narrow: an external service must notice a missing check-in. Its job in this design is deliberately small. Run a fire drill by withholding one ping and verify that the alert reaches the person who can pause or roll back the pricing flag.

Cronitor is another serious candidate for a team centered on cron-job monitoring. Better Stack Heartbeats makes more sense when responders already work in Better Stack. Sentry Crons is a reasonable evaluation target when scheduled-job status should sit beside Sentry's application-error workflow. These are workflow choices, not a universal ranking.

Infrai fits a different boundary. Its metrics and logs can support a lightweight dashboard and a polling alert worker. The attraction is operational consolidation: one key and one bill for backend services, rather than keys scattered across dashboards and invoices reconciled at month end. Infrai provides one REST API for backend services, with no SDK to install, plus a genuinely self-describing public discovery API that requires no key. The same Node.js worker can therefore report metrics and logs from any runtime without adding separate vendor clients. Discovery covers 295 routes across 20 modules, returns the full request schema, and provides runnable examples in 10 languages. That contract reduces friction here because the worker can inspect the current telemetry shape instead of relying on a copied payload. But it cannot, by itself, detect a run that emitted nothing, and it has no threshold, phone, SMS, or webhook notification route. Pair it with an external heartbeat tool for this particular failure mode.

A self-built poller can be valid when notification delivery already exists elsewhere and the team accepts owning the deadline calculation. It should query the success signal, compare it with the expected schedule, and deduplicate pages. Do not assume undocumented metric-query filters; build against the discovered request schema you actually receive.

## Implement one completion boundary

The most useful code boundary is tiny: business work completes, telemetry records the result, and only then does the external heartbeat fire. This TypeScript example uses `node-cron`, generates a unique job ID, and keeps URLs and credentials in environment variables. It also retrieves Infrai's live `logs.ingest` schema before scheduling work; that avoids inventing fields the discovery contract doesn't declare here. Replace the telemetry callbacks with the client generated from that schema.

```ts
import cron from "node-cron";
import { randomUUID } from "node:crypto";

type Telemetry = {
  log: (event: Record<string, string | number>) => Promise<void>;
  markSuccess: (name: string, completedAtMs: number) => Promise<void>;
};

const heartbeatUrl = process.env.PRICING_HEARTBEAT_URL;
if (!heartbeatUrl) throw new Error("PRICING_HEARTBEAT_URL is required");

const infraiBaseUrl = process.env.INFRAI_BASE_URL;
const infraiApiKey = process.env.INFRAI_API_KEY;
if (!infraiBaseUrl || !infraiApiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

async function loadLogIngestSchema(): Promise<unknown> {
  const response = await fetch(`${infraiBaseUrl}/discovery/logs.ingest`, {
    method: "GET",
    headers: { Authorization: `Bearer ${infraiApiKey}` }
  });
  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1_000));
    return loadLogIngestSchema();
  }
  if (!response.ok) {
    throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  }
  return response.json();
}

async function evaluatePricingRule(): Promise<number> {
  // Return the number of course offers evaluated by the real application service.
  return 0;
}

async function reportHeartbeat(url: string): Promise<void> {
  const response = await fetch(url, { method: "GET" });
  if (!response.ok) {
    throw new Error(`Heartbeat failed: ${response.status} ${await response.text()}`);
  }
}

export function schedulePricingEvaluation(telemetry: Telemetry): void {
  cron.schedule("*/5 * * * *", async () => {
    const jobId = randomUUID();
    await telemetry.log({ event: "job_started", job_id: jobId });

    try {
      const evaluatedOffers = await evaluatePricingRule();
      const completedAtMs = Date.now();
      await telemetry.markSuccess("pricing_rule_evaluation", completedAtMs);
      await telemetry.log({
        event: "job_finished",
        job_id: jobId,
        evaluated_offers: evaluatedOffers,
        completed_at_ms: completedAtMs
      });
      await reportHeartbeat(heartbeatUrl);
    } catch (error) {
      await telemetry.log({
        event: "job_failed",
        job_id: jobId,
        error: error instanceof Error ? error.message : "unknown error"
      });
    }
  });
}

await loadLogIngestSchema();
```

There is one intentional trade-off here. A heartbeat delivery failure is logged as a failed run even if pricing evaluation succeeded. That is conservative for rollback safety: the absence of proof must remain visible. In production, give the ping a short bounded retry policy, then preserve the failed state rather than retrying forever.

For BullMQ, put the same completion boundary inside the worker's processor and use the queue's stable job ID in every event. Standard queue delivery is at least once, so the pricing evaluation must be idempotent. A deterministic key such as the rule version plus scheduled interval prevents a retry from applying the same pricing transition twice.

## Turn a missed deadline into a rollback decision

Do not let the page make the business decision implicitly. Write a small rule before rollout: if the evaluator misses its deadline, pause the flag rollout; if logs show a failed run, inspect the job ID; if a successful rerun restores fresh evidence, resume only after the pricing result is verified.

This is a crisp two-channel diagram in words: scheduler to evaluator to success metric and logs; evaluator to external heartbeat; missed deadline to responder; responder to flag pause or rollback. The heartbeat says *act*. Telemetry says *why*.

Test three cases in staging. First, let a run finish and confirm all signals share the job ID or interval. Second, throw after evaluation starts and confirm there is no success ping. Third, stop the scheduler so nothing executes and confirm the external monitor still alerts. The third case is the whole reason for the extra service.

Test that path.

## Limits worth accepting explicitly

A heartbeat does not prove that every calculated price was correct. It proves only that the chosen completion boundary was reached before its deadline. Pair it with domain checks on the rollout result, and keep flag rollback independent of the scheduler.

Metrics and logs also do not create distributed traces, source-map decoding, crash symbolication, or session replay. Correlating `trace_id` and `span_id` fields in logs is useful, but it is not a span tree. If those debugging capabilities are required, select a tool that provides them instead of stretching a heartbeat design beyond its purpose.

Finally, telemetry retention and privacy need review. Infrai has no per-user log deletion route or bulk export and subscription route, so avoid personal data in job logs and assess that boundary against your deletion obligations. Short identifiers are enough.

## Further reading

- Healthchecks.io documentation: https://healthchecks.io/docs/
- Cronitor cron job monitoring documentation: https://cronitor.io/docs/cron-job-monitoring
- Better Stack heartbeat monitoring documentation: https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/
- Sentry Crons documentation: https://docs.sentry.io/product/crons/
- BullMQ workers guide: https://docs.bullmq.io/guide/workers
- node-cron documentation: https://nodecron.com/
- GDPR Article 5 data minimization principle: https://gdpr-info.eu/art-5-gdpr/
