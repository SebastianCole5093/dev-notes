# Node.js Failure Detection: Health Endpoint, API Metrics, and Regional Checks

Short answer: the best simple Node.js uptime alert is a chain of independent evidence: poll a small health endpoint from outside the service, confirm failures across time or regions, use metrics to explain the symptom, and require a heartbeat from the polling process so silence cannot look healthy.

Don't start with a dashboard. Start with the decision the alert must trigger. A page should mean that a user-relevant path is unavailable and somebody can act; a ticket can mean that one region, metric, or monitor has degraded without proving a broad outage.

That distinction keeps the first version small.

## Replace the green dot with an evidence chain

The tempting design is a timer that calls `/health`, treats `200` as green, and sends an email for everything else. It is easy to demo. It also mixes observation, policy, and notification into one fragile loop. A delayed response, a rejected connection, a rate limit, and an application response become the same event, while a dead timer produces no event at all.

Use this before/after mental model instead. Before: **timer -> endpoint -> page**. After: **regional probe -> classified observation -> incident state -> notification**, plus a separate dead-man signal that expects the probe to check in. Metrics and logs hang off the classified observation; they explain it, but they don't silently override it.

Each arrow has one job. The probe records what it actually observed: HTTP status, elapsed time, region, and a bounded outcome category. The incident state applies policy, such as three consecutive failures. The notifier routes only state changes, so twelve failed polls don't become twelve pages. The dead-man monitor answers a different question: did the checker complete a cycle recently?

Keep `/health` narrow. It should reveal whether the instance can serve the path represented by the check, not dump environment variables, dependency addresses, stack traces, session identifiers, or tokens. OWASP's logging guidance warns that logs can contain sensitive data and recommends deliberate exclusion or masking. Apply the same discipline to health responses and alert payloads. A useful observation can contain a check ID, region, duration, outcome, and incident ID without carrying a response body.

The result is less flashy and far easier to reason about.

## How should Node.js health endpoint polls, API metrics, and heartbeat alerts work together?

Treat the three signals as answers to three separate questions. An external poll asks, “Can this network path complete a representative request?” Metrics ask, “Is the service approaching or entering a bad operating state?” A heartbeat asks, “Is the monitor itself still completing work?” Combining them in one oversized health response creates coupling: a slow metrics backend can then make the application look unavailable, and a live application can still look healthy after its monitor has stopped.

For the poll, define success before writing retries. A response within the deadline may be healthy; a timeout means the deadline expired; a transport failure means no usable HTTP response arrived; a rate-limited response deserves its own classification; and another application status is an application failure. Preserve the first observation. Hidden retries can erase exactly the evidence needed to understand intermittent failures.

Metrics should add context rather than vote the poll away. Request rate, error ratio, latency, saturation, and queue depth are common explanatory dimensions, but choose only measurements tied to an action your team understands. For example, a failing external poll plus rising request errors supports a service incident. A failing poll in one region with normal service metrics points toward a path-specific investigation. Normal metrics do not prove that DNS, TLS, or routing works from the user's side.

Send the probe heartbeat after the cycle finishes, not when the scheduler wakes up. The early heartbeat proves only that a timer fired. The later one proves that the probe reached the point where it classified and recorded the attempt. The dead-man deadline should exceed the normal interval plus expected execution jitter; the exact margin depends on the scheduler and notification latency, so I'm not sure a universal number would be honest. Your mileage may vary. Measure the cycle time, then set and test the deadline.

Regional checks also need declared semantics. If EU and US probes exercise a globally routed service, one failing region can open a degraded state while both failing can page. If the regions serve separate populations or separate deployments, each region owns its own incident. Calling one probe a “fallback” doesn't decide this — traffic ownership does.

## A copyable TypeScript monitor with explicit state

The example below separates collection from policy. It makes one attempt per cycle, classifies `429` independently, opens an incident after three consecutive failures, and resolves it on a healthy observation. The output callback is intentionally generic; connect it to a queue or notifier that your team already owns. Run one process per observation region and keep incident state in durable storage when process restarts must not reset the threshold.

```ts
type Observation =
  | { kind: "healthy"; status: number; durationMs: number }
  | { kind: "rate_limited"; status: 429; durationMs: number }
  | { kind: "application_failure"; status: number; durationMs: number }
  | { kind: "timeout"; durationMs: number }
  | { kind: "transport_failure"; durationMs: number; errorName: string };

type IncidentState = {
  consecutiveFailures: number;
  open: boolean;
};

type Event =
  | { type: "observation"; region: string; observation: Observation }
  | { type: "incident_opened"; region: string; observation: Observation }
  | { type: "incident_resolved"; region: string; observation: Observation }
  | { type: "probe_heartbeat"; region: string; completedAt: string };

async function observe(url: URL, timeoutMs: number): Promise<Observation> {
  const startedAt = Date.now();
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, {
      method: "GET",
      headers: { accept: "application/json" },
      signal: controller.signal,
    });
    const durationMs = Date.now() - startedAt;

    if (response.status === 429) {
      return { kind: "rate_limited", status: 429, durationMs };
    }
    if (response.ok) {
      return { kind: "healthy", status: response.status, durationMs };
    }
    return {
      kind: "application_failure",
      status: response.status,
      durationMs,
    };
  } catch (error) {
    const durationMs = Date.now() - startedAt;
    if (error instanceof Error && error.name === "AbortError") {
      return { kind: "timeout", durationMs };
    }
    return {
      kind: "transport_failure",
      durationMs,
      errorName: error instanceof Error ? error.name : "UnknownError",
    };
  } finally {
    clearTimeout(timeout);
  }
}

function advance(
  state: IncidentState,
  observation: Observation,
  failureThreshold: number,
): "none" | "open" | "resolve" {
  const failed = observation.kind !== "healthy";
  state.consecutiveFailures = failed ? state.consecutiveFailures + 1 : 0;

  if (!state.open && state.consecutiveFailures >= failureThreshold) {
    state.open = true;
    return "open";
  }
  if (state.open && !failed) {
    state.open = false;
    return "resolve";
  }
  return "none";
}

async function runCycle(
  url: URL,
  region: string,
  state: IncidentState,
  emit: (event: Event) => Promise<void>,
): Promise<void> {
  const observation = await observe(url, 5_000);
  await emit({ type: "observation", region, observation });

  const transition = advance(state, observation, 3);
  if (transition === "open") {
    await emit({ type: "incident_opened", region, observation });
  } else if (transition === "resolve") {
    await emit({ type: "incident_resolved", region, observation });
  }

  await emit({
    type: "probe_heartbeat",
    region,
    completedAt: new Date().toISOString(),
  });
}
```

There is one policy choice to revisit: this sample counts rate limiting as a failure. That is conservative because `429` does not prove the checked path is available to ordinary traffic. A team may instead open a capacity ticket for `429` and reserve the availability page for timeouts, transport failures, or application failures. Keep the category either way. Don't turn it green merely because an automatic retry later succeeds.

Test the state machine without real network calls by passing representative observations into `advance`: two failures must remain closed, the third must open, continued failures must not reopen, and the first healthy result must resolve. Then test the complete delivery path against a disposable endpoint. Trigger a failure, verify that the intended recipient gets one notification, restore the endpoint, and verify the resolution. This end-to-end exercise checks more than a unit test can: scheduler, state storage, routing, credentials, and the human destination.

## Choose deployment and alert policy by failure ownership

A free or cheap SaaS monitor is reasonable when the team wants an external scheduler and managed notification delivery, but price is a weak primary criterion. Evaluate probe locations, retention, export, notification routes, data handling, team access, and the behavior of missed checks. Read the data-processing terms if EU/US placement matters. A map pin for a probe does not by itself establish where alert data, logs, or account metadata is stored.

Self-hosting is useful for private networks, custom authentication, strict control of telemetry, or checks that require internal context. The catch is ownership: the team now operates the scheduler, state store, upgrades, and delivery integration. It is not suitable when the monitor would run beside the only workload it watches, because one infrastructure failure can silence both. Stick with an externally operated check when nobody can own that second system; use an internal probe when access constraints make the external path impossible, and have an independent dead-man signal watch it.

Use a small decision table during review:

| Decision | Page | Record or ticket |
|---|---|---|
| Both probes fail for the threshold window | Global service is unreachable from both represented paths | Raw observations and metrics context |
| One regional probe fails | Only if that region has independent users or deployment ownership | Degraded regional reachability |
| Polls stop arriving | Dead-man deadline expires | Scheduler delay below the deadline |
| Metrics degrade while the poll succeeds | Only when a user-impact threshold and response are defined | Early capacity or dependency signal |
| Notification delivery test fails | Escalate through an independently tested route | Repair the alert path before trusting it |

Privacy belongs in the same review. Alert events often travel through more systems and remain longer than request memory. Record stable pseudonymous correlation values when needed, and omit secrets, access tokens, session identifiers, and unnecessary personal data. For a locally installed CLI or probe that sends optional usage telemetry, honoring `DO_NOT_TRACK=1` gives operators a recognizable opt-out convention. It doesn't replace clear documentation or data controls, but it is an easy expectation to test.

## Two objections worth settling before rollout

“Can metrics replace synthetic polling?” No. Metrics can reveal application behavior and resource pressure, but they normally do not exercise the same DNS, TLS, routing, and request path as an outside client. Polling has the opposite blind spot: one synthetic request can succeed while an untested user journey fails. Use the poll as a narrow reachability claim and metrics as operational context. Add a deeper transaction only when its extra dependencies and side effects are justified.

“Won't three failures make detection too slow?” Maybe. The threshold trades detection speed for resistance to transient noise, and the interval determines the real delay. Don't choose either value by folklore. Write down the maximum tolerable detection time, subtract notification latency, and test a few controlled failures. For a low-traffic service, synthetic probes may be the only steady user-path evidence. For a high-traffic service, error and latency signals may justify a faster parallel page. Keep the raw observations even when the incident rule suppresses a notification.

The durable baseline is modest: one narrow endpoint, explicit outcome classes, incident transitions, explanatory metrics, protected logs, tested delivery, and a heartbeat watched by something independent. Ship that. Add regions or deeper checks only when they represent a real user path and change an operational decision.

## References

- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- Console Do Not Track convention: https://consoledonottrack.com/

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://consoledonottrack.com/
