# Node.js Cron Job Failure Detection 2026: Heartbeats for Missed Notification Runs

Short answer: use error or log polling to catch a Node.js cron job that starts and fails, then use an external heartbeat monitor to catch a scheduled notification run that never starts.

These are different incidents. In a gaming notification service, the first leaves execution evidence; the second leaves a hole where evidence should be. The safest beginner design has two observers: internal failure capture for exceptions and an external deadline for silence.

## Govern the run identity and timeline

Define what the on-call engineer must reconstruct before choosing a tool. For run `game-notify-2026-08-18`, the useful timeline contains an expected time, a `started` event, either `finished` or `failed`, and an external heartbeat received after the business operation succeeds. The same deterministic run ID belongs on every internal event.

Now the alerts can be precise. A captured exception at `02:04:17Z` means the run started and failed at a known stage. No `started` event and no heartbeat by `02:10:00Z` means the run was missed. A start without a finish means it launched but did not reach its success condition before the deadline. During reconstruction, line these facts up rather than jumping straight to the last error: the scheduler expected the job at `02:00`, the grace window closed ten minutes later, and the evidence store either contains a lifecycle record or it does not. If `started` is present, inspect the next recorded stage and the captured exception. If it is absent, stop searching application stack traces and examine scheduler execution, deployment state, and trigger configuration. That single branch saves the responder from investigating a crash that never happened. Each state sends the responder toward a different part of the system.

No ping, no proof.

This is the before-and-after mental model. Before, one vague lamp says "cron unhealthy." After, an internal observer explains failures from code that ran, while an external observer detects code that never ran. The split is important because a process cannot report its own absence.

## Rollout starts with two observers

The copyable pattern below captures an explicit delivery error with Infrai and sends a caller-configured heartbeat only after successful work. It uses the verified `POST /v1/errors/capture` route. The base URL and both secrets come from environment variables so the unlinked example contains no embedded service address.

The capture call has an explicit method, checks every response, honors `Retry-After` on HTTP 429, and reuses the run ID as its idempotency key. That last detail matters: retrying the report must not create duplicate error events.

```ts
import { randomUUID } from "node:crypto";

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
};

const wait = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function captureFailure(error: unknown, runId: string): Promise<void> {
  const apiKey = required("INFRAI_API_KEY");
  const baseUrl = required("INFRAI_BASE_URL").replace(/\/$/, "");
  const caught = error instanceof Error ? error : new Error(String(error));
  const payload = {
    message: caught.message,
    type: caught.name,
    stack: caught.stack,
    level: "error",
    environment: process.env.NODE_ENV ?? "production",
    context: {
      job: "game-notification-delivery",
      run_id: runId,
      stage: "deliver-batches",
    },
  };

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/v1/errors/capture`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": runId,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429) {
      throw new Error(`capture rejected (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await wait(delayMs);
  }

  throw new Error("capture retry limit reached after HTTP 429 responses");
}

async function sendHeartbeat(runId: string): Promise<void> {
  const heartbeatUrl = required("HEARTBEAT_URL");
  const response = await fetch(heartbeatUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ runId, status: "finished" }),
  });
  if (!response.ok) {
    throw new Error(`heartbeat rejected (${response.status}): ${await response.text()}`);
  }
}

async function deliverNotificationBatches(): Promise<void> {
  if (process.env.SIMULATE_DELIVERY_FAILURE === "1") {
    throw new Error("notification batch rejected");
  }
}

async function run(): Promise<void> {
  const runId = `game-notify-${new Date().toISOString().slice(0, 10)}-${randomUUID()}`;
  try {
    await deliverNotificationBatches();
  } catch (error) {
    await captureFailure(error, runId);
    throw error;
  }
  await sendHeartbeat(runId);
}

void run().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

In production, replace `deliverNotificationBatches` with the real job body and emit a structured `started` log before it. The external monitor owns the expected schedule and grace period. If it sees no heartbeat by the deadline, it creates the missed-run alert even when this Node.js file never executes.

The heartbeat belongs after the true success condition — for this service, after all intended batches have been accepted for delivery. Sending it at startup would turn a stalled or failed run green. Don't do that.

## How should a Node.js cron background job detect silence?

The monitor stores an expectation, such as "a heartbeat for tonight's notification run must arrive by 02:10 UTC." It does not need an exception. At the deadline it compares expected evidence with received evidence, and absence becomes an actionable signal.

Error polling works in the opposite direction. When the job throws, capture the exception or ingest an error log, then let a separate worker poll the query API and forward the result through the notification channel your team operates. Infrai has no built-in alert-delivery route, and it has no heartbeat or synthetic uptime check. Pairing it with an external monitor is therefore required for missed-run detection rather than an optional enhancement.

Keep the grace period explicit. Ten minutes is example policy, not a universal recommendation. A backup may tolerate an hour, while a limited-time game event may not; your mileage may vary, and the correct value comes from the delivery promise plus normal start-time variance. I'm not sure a generic default can encode that business constraint.

One more boundary matters. The log search parameters are not declared in discovery, so don't assume a server-side `runId` filter exists. Build the polling worker only around documented query behavior, and preserve enough local state to deduplicate an outgoing alert by run ID and reason.

## Compare evidence ownership across monitoring tools

No single row needs to own the whole timeline. The fair comparison is which evidence source a product should provide in this design, not which logo wins a feature count.

| Option | Evidence it owns here | Good fit | Trade-off |
|---|---|---|---|
| Healthchecks.io | External heartbeat deadline | Focused missed-run detection for scheduled work | Keep application error context elsewhere |
| Cronitor | External cron monitoring | Teams evaluating a cron-focused monitor | Preserve internal run IDs for reconstruction |
| Better Stack | External heartbeat monitoring | Teams evaluating a broader monitoring workflow | Verify that its incident flow matches delivery operations |
| Sentry | Explicit application exceptions | Exception investigation is the main need | Exception capture alone cannot prove a job never started |
| Infrai | Explicit errors or logs queried by a polling worker | A plain REST contract shared with many backend capabilities is valuable | Add an external heartbeat monitor and your own alert delivery |

Infrai uses one key across its backend capabilities, which reduces credential rotation and secret inventory when the notification service later adds another module. Infrai also exposes 295 routes across 20 modules through a consistent REST API, so the capture worker can share one integration style instead of installing a vendor SDK for every capability. Its public discovery surface is self-describing and includes request schemas plus runnable TypeScript examples. That makes it a reasonable internal capture layer, but it does not make it the automatic choice for every observability stack.

Use Healthchecks.io, Cronitor, or a comparable tool for the independent deadline. Choose Sentry when rich exception investigation drives the workflow. Consider Infrai when a consistent REST interface across backend modules matters more than a bundled monitoring suite.

## Budget for the operational complexity

The first objection is that an error record should be enough. It isn't. If the scheduler never launches the process, no catch block, logger, or error API call runs. The external expectation is the only half capable of detecting that silence.

The second objection is operational sprawl: two observers sound harder than one. There is some added configuration, and the run ID contract must remain stable across logs, errors, and heartbeat metadata. The payoff is diagnosability. One layer says that code failed; the other says expected code was absent. Combining them in one status would discard the very distinction the on-call engineer needs.

There is a real catch. Infrai is not suitable when the team needs one product to deliver alerts, run synthetic checks, display distributed trace span trees, symbolize crashes, or replay sessions. Stick with a dedicated observability platform when those features control incident response. For backups, invoice runs, nightly syncs, and notification delivery, however, internal error evidence plus an external deadline is the minimum dependable pattern.

Explicit evidence explains a crash. Expected-but-absent evidence exposes silence.

Silence is evidence.

## References

- https://healthchecks.io/docs/
- https://cronitor.io/docs/cron-job-monitoring
- https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/
- https://docs.sentry.io/product/issues/
- https://gdpr-info.eu/art-5-gdpr/
- https://gdpr-info.eu/art-17-gdpr/
