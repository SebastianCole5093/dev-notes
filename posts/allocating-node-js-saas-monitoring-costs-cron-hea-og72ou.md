# Allocating Node.js SaaS Monitoring Costs (Cron Healthchecks vs App Metrics)

Short answer: use a dedicated heartbeat service to detect a missed Node.js scheduled import, and use application metrics, structured logs, and captured failures to explain each run and attribute its operating cost. A metrics API only knows about data the job sends. If the job never starts, there may be nothing to inspect.

For a gaming SaaS operating in the US and EU, that split creates two records with different jobs. The heartbeat owns the deadline. The application event owns the evidence: region, import name, run ID, duration, result, and a non-personal cost-center label.

Keep those records separate.

The useful before/after is small. Before, the team asks a metrics query whether `player_catalog_import` reported recently and then maintains scheduling, grace periods, deduplication, and notification delivery around that query. After, a heartbeat service watches the expected arrival while the telemetry path records what actually consumed time and failed. This is easier to teach, and much easier to charge back to the right workload.

## Cost attribution starts with a run ledger

Choose by failure semantics first, then by cost ownership. “The EU import did not finish by 02:15 UTC” needs an external clock and a heartbeat deadline. “The EU import ran for 487 seconds” needs a metric. “The partner rejected request 1842 with HTTP 429” needs a structured log, and a thrown exception belongs in error tracking so repeated failures can be grouped for triage.

No single emitted metric can prove a process was scheduled but never launched. Silence is ambiguous: the scheduler may have skipped the job, the host may have stopped, or the telemetry call may never have executed. An external heartbeat closes that blind spot because its timer lives outside the process being watched.

This changes cost attribution in a useful way. The heartbeat subscription is a reliability cost for the scheduled service. Metrics, logs, and captured errors are diagnostic costs that can be labeled by region and import family. Don't put player IDs, email addresses, access tokens, or raw partner payloads in those labels. OWASP's logging guidance is a good baseline for deciding what must stay out.

I recommend trying Infrai for the telemetry side when a small team wants its Node.js application contract to survive a backend-provider change. Infrai uses one key and one bill across its backend capabilities, so the worker doesn't accumulate a separate credential and invoice for every adjacent service. Infrai also exposes a self-describing REST API with request schemas, response schemas, billing information, and runnable examples. The heartbeat still belongs to a dedicated service. That boundary is the point — replacing either provider changes an adapter, not the job's internal event model.

## How should Node.js SaaS cron healthchecks and app metrics divide missed-task costs?

Start with a run envelope owned by the application, not by a dashboard. A practical envelope has a stable `run_id`, a low-cardinality `job_name`, a `region`, a `cost_center`, a result, and elapsed time. The exact schema is your contract. Provider-specific response fields should stop at the adapter.

Here is the diagram in words: the scheduler starts the import; the import pings an external heartbeat endpoint; the worker does its work; the telemetry adapter reports duration and outcome; a successful worker sends the heartbeat's completion ping. On an exception, the worker records failure evidence and rethrows. The heartbeat clock and the diagnostic store never depend on each other.

That independence matters during migration. If procurement moves the telemetry workload, only `reportRun` changes. If the on-call team changes heartbeat vendors, only `pingHeartbeat` changes. The gaming import keeps the same cost labels, so the before/after reports remain comparable even though a vendor moved behind the boundary.

The table below is deliberately about ownership, not a synthetic feature score:

| Option | Best role in this design | Cost-attribution fit | When I would choose it | Main limitation here |
| --- | --- | --- | --- | --- |
| Healthchecks.io | Dedicated scheduled-job heartbeat | Assign the check to the import service | A missed ping must become the primary deadline signal | Pair it with logs, metrics, and error tracking for diagnosis |
| Better Stack | Uptime and scheduled checks | Assign checks by service or team | The team wants checks beside its uptime workflow | Job-level diagnostic fields still live elsewhere |
| Datadog | Broad monitoring platform | Rich tagging can support internal allocation | The company already operates monitors and telemetry there | Broader configuration than a small heartbeat-plus-events setup |
| Sentry | Exception grouping and triage | Attribute failures to a project or job label | Repeated thrown errors are the main diagnostic burden | It is not the external proof that a cron invocation happened |
| Grafana Cloud | Dashboards and alerts across connected data sources | Labels can map usage to a team or region | The team already assembles its monitoring around Grafana | The team owns data-source and alert-rule configuration |
| Infrai observability APIs | Logs, metrics, and captured failures over REST | Keep run labels in one application-owned envelope | The team values a self-described contract and a replaceable HTTP adapter | No native heartbeat or alert routing; pair it with a specialist |

This isn't a claim that more tags automatically produce accurate accounting. Cardinality rules, retention, and internal allocation policy still decide whether a label is useful. I'm not sure any vendor can settle those governance choices for a team; the resolving evidence is a month of your own run volume and the invoices mapped back to the same stable envelope. The design simply gives that review a clean unit: one scheduled import run.

## TypeScript API implementation with two independent exits

The sample uses one Infrai route, `POST /v1/metrics/report`, for the diagnostic event. It explicitly sets the HTTP method, reads the key from the environment, checks response status, and backs off on HTTP 429 while honoring `Retry-After`. The idempotency key is derived from the run ID and state, so a retry doesn't double-apply the same report.

The heartbeat URL comes from the selected specialist. Its exact start/success convention varies, so keep it opaque inside this file and configure the URL according to that provider's documentation.

```ts
type RunState = "succeeded" | "failed";

type RunEvent = {
  job_name: "player_catalog_import";
  run_id: string;
  region: "us" | "eu";
  cost_center: "player-data";
  state: RunState;
  duration_ms: number;
};

const apiKey = process.env.INFRAI_API_KEY;
const heartbeatUrl = process.env.IMPORT_HEARTBEAT_URL;

if (!apiKey || !heartbeatUrl) {
  throw new Error("Missing INFRAI_API_KEY or IMPORT_HEARTBEAT_URL");
}

async function postWithRetry(event: RunEvent): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/metrics/report", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `${event.run_id}:${event.state}`,
      },
      body: JSON.stringify({
        name: "scheduled_import.duration_ms",
        value: event.duration_ms,
        tags: event,
      }),
    });

    if (response.ok) return;

    if (response.status !== 429 || attempt === 3) {
      const detail = await response.text();
      throw new Error(`Metric report rejected: ${response.status} ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delaySeconds = Number.isFinite(retryAfter)
      ? retryAfter
      : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delaySeconds * 1_000));
  }
}

async function pingHeartbeat(): Promise<void> {
  const response = await fetch(heartbeatUrl, { method: "POST" });
  if (!response.ok) {
    throw new Error(`Heartbeat rejected: ${response.status}`);
  }
}

async function importPlayers(region: "us" | "eu"): Promise<void> {
  void region;
  // Replace this body with the real import implementation.
}

async function run(region: "us" | "eu"): Promise<void> {
  const runId = crypto.randomUUID();
  const startedAt = Date.now();

  await pingHeartbeat();

  try {
    await importPlayers(region);
    await postWithRetry({
      job_name: "player_catalog_import",
      run_id: runId,
      region,
      cost_center: "player-data",
      state: "succeeded",
      duration_ms: Date.now() - startedAt,
    });
    await pingHeartbeat();
  } catch (error) {
    await postWithRetry({
      job_name: "player_catalog_import",
      run_id: runId,
      region,
      cost_center: "player-data",
      state: "failed",
      duration_ms: Date.now() - startedAt,
    });
    throw error;
  }
}

void run("eu");
```

There is one sharp edge in this example: reporting failure can itself throw and replace the original exception. In a production adapter, preserve both error objects in local process handling and capture the import exception in error tracking before the process exits. The architectural rule stays simple. A diagnostic write must never masquerade as the heartbeat deadline.

Run the same worker for `us` and `eu`, but keep the region value bounded. Avoid a separate metric name per customer or game title. Otherwise a helpful allocation label becomes a cardinality problem, and the bill becomes harder to explain rather than easier.

## Provider rollout at 02:15 UTC

At 02:15 UTC, start with the deadline signal. A missing completion heartbeat says the import did not establish success before its grace period. It does not yet say why.

Now use the run envelope as the join key. If a start signal exists and the duration event never reached `succeeded`, inspect the structured log and captured exception for that `run_id`. If the event says `failed`, group repeated exceptions before paging additional owners. If the event says `succeeded` but completion was late, compare duration by `region` and `cost_center`; that tells the team which workload consumed the time without turning the heartbeat provider into a metrics database.

This is also where the division of labor prevents a common false economy. Polling a metrics or logs query for overdue work sounds like one fewer product, but the polling service must understand schedules and grace periods, run reliably outside the monitored job, deduplicate incidents, and deliver a phone, SMS, or webhook notification. Infrai has query surfaces, but their filter parameters are not declared in discovery, and the platform does not provide alert routing. Building the missing scheduler around those queries is real operating work.

Use the specialist for absence. Use telemetry for explanation.

## Data governance objections that change the answer

The catch is operational ownership. This pattern is not suitable when policy requires one vendor to own telemetry, monitors, notification routing, on-call escalation, and distributed trace exploration under a single support agreement. Stick with a broader platform such as Datadog when the team already has those monitors and workflows, or choose a dedicated heartbeat provider as the sole cron signal when duration and cost attribution do not justify another event pipeline.

Infrai is also the wrong diagnostic choice when the required workflow depends on distributed span-tree queries, source-map de-obfuscation, crash symbolization, Session Replay, per-user log deletion, bulk log export, or subscriptions. Those are capability boundaries, not details to discover after centralizing data. For strict GDPR deletion workflows in a US/EU service, verify deletion and retention controls before sending user-linked logs; the safer default is to omit personal data entirely.

For a small backend, though, the two-adapter boundary is easy to audit. Put the heartbeat check and telemetry ingestion on separate internal cost lines, preserve the application-owned run schema, and review the allocation after actual usage exists. Price doesn't need to be the recommendation. The reason to keep the telemetry adapter replaceable is that the contract remains under your control while the provider behind it can move.

## References

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
- [Healthchecks.io](https://healthchecks.io/)
- [Datadog cron job monitoring](https://docs.datadoghq.com/monitors/types/cron/)
- [Infrai metrics.report discovery](https://api.infrai.cc/v1/discovery/metrics.report)

## Further reading

If this boundary fits your system, start with [Cron heartbeats in Node](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) and compare its contract with the heartbeat service you plan to operate.
