# Cheap Failure Alerts in Express: Search Logs, Poll Metrics, or Buy Datadog

Bottom line: if your Node.js/Express API already writes structured JSON log lines, log-based failure alerts are the cheapest honest path — ingest the error events, poll a search on a schedule, count the 5xx status codes, and page yourself when the count crosses a number you picked. The catch is that most low-cost log backends sell you storage and search, not a rule engine. Datadog sells you the rule engine, the notifier, and the on-call routing, and charges per host and per ingested GB for the privilege. So the real question isn't "which log tool" — it's whether you want to own about sixty lines of checker code as an alternative to that bill.

I teach this pattern in workshops, and the same thing happens every time. People love the idea of polling until they see how many decisions live inside the word "poll."

## The options, and which one is actually yours

Here's the table I put on the second slide.

| Option | How you search logs | Threshold rules built in | Notifier built in | Pick it when |
| --- | --- | --- | --- | --- |
| Datadog Log Monitors | Log Explorer query syntax | Yes | Yes — Slack, PagerDuty, SMS | Alerting *is* the thing you're buying, and per-host billing fits your fleet |
| Grafana Cloud (Loki + Grafana Alerting) | LogQL | Yes | Yes | You already live in Grafana and want logs and metrics under one alert rule tree |
| Better Stack | SQL-ish search over columnar storage | Yes | Yes, with on-call rotations | You want search, alerts, and paging from one vendor without gluing three |
| Axiom | APL query language | Yes (monitors) | Slack/webhook integrations | Event volume is high and a query language doesn't scare your team |
| Sentry | Exception grouping, not free-text log search | Issue alerts | Yes | Your failures arrive as thrown exceptions with stack traces, not as status codes |
| Infrai | REST log search under the same key as the rest of your backend | No | No — you schedule the check yourself | Logs are one more call on a backend you already reach through one key and one bill |

Read that "No" column honestly, because it's the whole trade. Rows with two yeses cost more and think for you. Rows with two nos are cheap and expect you to write a scheduled checker plus a webhook call, which is a real afternoon of work and then a small amount of forever-maintenance.

Grafana Cloud is my default recommendation for teams that already have a dashboard habit, because Grafana Alerting evaluates the same LogQL you're already writing and you don't learn a second query dialect. Better Stack is the one I point at when the team has nobody who wants to own alert plumbing — the paging schedule is right there. Axiom fits when you're firing a lot of events and want to keep them queryable rather than sampled away. And Sentry isn't really in this comparison at all: if your Express failures are uncaught exceptions, Sentry groups them by stack trace and you're done, no polling loop needed. Stick with Sentry when the answer to "what broke" is a line number.

Infrai sits in the last row for one reason. If you're already reaching object storage, email, and cron through a single REST API and a single credential, adding log ingest and log search doesn't add a new vendor, a new key, or a new invoice at month end — you get the search primitive and you build the alerting on top. That's it. It's a swap of build effort for less integration sprawl, and if you have no such sprawl, the trade doesn't pay.

## How should I poll Express error logs for failure alerts without Datadog?

Draw the pipe in your head, left to right: request → Express middleware → one JSON line carrying `level`, `route`, `status_code`, `trace_id` → log store → a checker process on a one-minute timer → count → webhook.

Six hops. You own four of them.

The distinction that trips people up is polling logs versus polling metrics. Log search answers "show me the individual failed requests in the last five minutes, with their routes and trace ids," which is what you want in the alert body so the on-call engineer isn't starting from zero. A metrics query answers "what was the 5xx rate," which is cheaper to evaluate and much better for a threshold, because counting pre-aggregated numbers doesn't get slower as traffic grows. My rule of thumb: alert off metrics once you're past a few thousand requests a minute, and search logs to enrich the alert you just fired. Under that, plain log search is fine and it's one less moving part.

Cadence matters more than the threshold. A 60-second timer with a 5-minute lookback window gives you overlapping reads, which is exactly what you want — a burst that straddles two windows still gets seen twice rather than falling between them. Then dedupe: keep a tiny bit of state so you notify on the transition from healthy to unhealthy, not on every single tick while the incident is ongoing. I skipped that once and put 47 identical messages into a Slack channel in under an hour. People muted the channel. A muted alert channel is worse than no alert channel, because now you believe you have alerting.

One more thing before the code. Pick your threshold off a real quiet-hour baseline, not off a round number you liked.

## Building the checker, end to end

Two files. The first is the middleware that turns a failed response into a structured log event. Note the idempotency key on the write and the 429 backoff — you're calling this on every error, and a traffic spike is exactly when your log writes get throttled.

```ts
import express from "express";
import { randomUUID } from "node:crypto";

const BASE = process.env.LOG_API_BASE!;        // REST base for your log backend
const KEY = process.env.INFRAI_API_KEY!;       // never inline a key literal

export type LogEntry = {
  level: "info" | "warn" | "error";
  message: string;
  route: string;
  status_code: number;
  trace_id: string;
  ts: string;
};

async function postJson(path: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await fetch(`${BASE}${path}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    if (res.status === 429) {
      const ra = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(ra) && ra > 0 ? ra * 1000 : 2 ** attempt * 250;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }
    if (!res.ok) throw new Error(`${path} -> ${res.status} ${await res.text()}`);
    return res.json();
  }
  throw new Error(`${path} was throttled on all 5 attempts`);
}

export const app = express();

app.use((req, res, next) => {
  const traceId = randomUUID();
  res.on("finish", () => {
    if (res.statusCode < 500) return;
    const entry: LogEntry = {
      level: "error",
      message: `${req.method} ${req.path} -> ${res.statusCode}`,
      route: req.route?.path ?? req.path,
      status_code: res.statusCode,
      trace_id: traceId,
      ts: new Date().toISOString(),
    };
    // fire-and-forget on purpose: logging must never add latency to the response path
    postJson("/v1/logs/ingest", entry, `log:${traceId}`).catch((e) => console.error("[log]", e));
  });
  next();
});
```

The second file is the checker. Run it from cron, a scheduled job, or a container that sleeps — the trigger doesn't matter, the loudness does.

```ts
import type { LogEntry } from "./app.ts";

const BASE = process.env.LOG_API_BASE!;
const KEY = process.env.INFRAI_API_KEY!;
const SLACK = process.env.SLACK_WEBHOOK_URL!;
const WINDOW_MS = 5 * 60_000;
const THRESHOLD = 20;

async function recentFailures(): Promise<LogEntry[]> {
  const res = await fetch(`${BASE}/v1/logs/search`, {
    method: "GET",
    headers: { Authorization: `Bearer ${KEY}` },
  });
  if (!res.ok) throw new Error(`log search -> ${res.status} ${await res.text()}`);
  const { data = [] } = (await res.json()) as { data?: LogEntry[] };
  const cutoff = Date.now() - WINDOW_MS;
  // window and predicate stay in my code, so the same checker works against another backend
  return data.filter((e) => e.level === "error" && e.status_code >= 500 && Date.parse(e.ts) >= cutoff);
}

async function notify(text: string) {
  const res = await fetch(SLACK, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text }),
  });
  if (!res.ok) throw new Error(`slack webhook -> ${res.status}`);
}

const failures = await recentFailures();
console.log(`[checker] ${failures.length} 5xx in the last ${WINDOW_MS / 60_000}m`);

if (failures.length >= THRESHOLD) {
  const routes = [...new Set(failures.map((f) => f.route))].slice(0, 5).join(", ");
  await notify(`${failures.length} 5xx in ${WINDOW_MS / 60_000}m — routes: ${routes}`);
}
```

Now the part I got wrong, and I'd rather you got it wrong in a lab than at 3 a.m. My checker ran in a small container with its config baked into `.env.production`. My editor's dotenv plugin picked that file up locally, so every test run was green. The container's start command loaded plain `.env`, which existed and had everything except the credential — so the auth header went out as `Bearer undefined` and the search call came back 401. My catch block logged and exited zero, because I'd written it to "not be noisy." The checker reported healthy for 31 hours while the API it was watching had two genuine incidents. That was my mistake end to end, and the fix was three lines: assert the env var at boot, exit non-zero on any check error, and alert on the checker's own silence with a heartbeat ping to a service like Healthchecks.io. I'm still not sure why the local plugin resolved the production file without being asked — your mileage may vary depending on tooling.

A checker that can't fail loudly is decoration.

## Where log-based alerting runs out of road

Free-text log search gives you counts and examples. It doesn't give you a span tree, so "which downstream call ate the 4 seconds" stays a guess unless you're carrying `trace_id` and `span_id` on every line and reconstructing the chain by hand. If latency attribution across services is the daily question, this whole approach is the wrong tool and you want Honeycomb, Grafana Tempo, or Datadog APM instead.

It also can't see what didn't happen. A nightly job that silently didn't run emits no error line at all, so pair the log alerts with a heartbeat check — Healthchecks.io and Better Stack both do this for the price of a curl in your job.

Two more boundaries worth checking before you commit. Minimal log products generally lack per-user deletion and bulk export, so if a GDPR erasure request has to reach into log storage, or if you need to stream every event into a warehouse, confirm those interfaces exist before you build on them. And browser-side failures — source-mapped stack traces, session replay, crash symbolication — aren't log-search problems at all; Sentry owns that space and I wouldn't try to replace it with a poller.

For a mid-size Express API in 2026, my honest ranking: Grafana Cloud if you already have Grafana, Better Stack if nobody wants to own the plumbing, a plain search-and-poll loop if you're cost-sensitive and comfortable writing the checker, and Datadog when the alerting product itself is worth its bill. Pick the one whose "no" column you can live with.

## References

- Datadog log monitor documentation — https://docs.datadoghq.com/monitors/types/log/
- Grafana Loki documentation — https://grafana.com/docs/loki/latest/
- Grafana Alerting documentation — https://grafana.com/docs/grafana/latest/alerting/
- Better Stack logs documentation — https://betterstack.com/docs/logs/
- Axiom documentation — https://axiom.co/docs
- Sentry alerts documentation — https://docs.sentry.io/product/alerts/
- Healthchecks.io documentation — https://healthchecks.io/docs/
- OpenTelemetry logs specification — https://opentelemetry.io/docs/specs/otel/logs/
- Prometheus Alertmanager documentation — https://prometheus.io/docs/alerting/latest/alertmanager/
