# PDF Generation API: Synchronous and Background Jobs for Health Archives

A monthly health report becomes an archive record the moment someone may need to retrieve the same rendition after the original request has ended. That operational constraint changes the PDF generation API decision: use a background job for the archived report, and reserve synchronous generation for a small preview or an interaction that truly cannot proceed without the bytes.

Short answer: a synchronous PDF response is a good contract when the work is bounded, the caller is present, and no durable artifact promise exists. A background job is the better contract when the report must be rendered from a frozen input, retained independently, retried after a worker interruption, and explained months later.

The before-and-after model is useful. Before, one HTTP request validates the reporting period, reads live data, selects the current template, renders pages, stores a file, and sends bytes. After, accepting the request creates a durable render record; a worker renders one declared template revision from one declared data snapshot; the archive becomes visible only after persistence succeeds. RFC 9110 defines `202 Accepted` as acceptance for processing, not a promise that processing has completed.

The queue is a recovery boundary.

## Should PDF generation be synchronous or use a background job API?

Keep generation in the request path for a constrained preview: perhaps a clinician-facing view of a single, already-selected reporting period, with a known template and no archival obligation. The caller gets either a completed response or an ordinary request failure. There is less state to operate, fewer access paths to protect, and no status model to teach to a user.

That simplicity has a boundary. Once the same PDF must be kept as a monthly record, a completed response is insufficient evidence. A later reviewer may need to know the reporting period, the source snapshot, the template revision, and the artifact identity. Rendering again from today's template does not recreate the earlier document if the footer, grouping logic, or page layout has changed.

For health information, decide deliberately who can retrieve an archived artifact and what events are recorded around that retrieval. The HIPAA Security Rule describes safeguards for electronic protected health information. It does not justify putting report data, signed download URLs, or authorization headers into general-purpose application logs; the OWASP Logging Cheat Sheet makes the same operational distinction between useful events and sensitive values.

There is a trade-off here. A job introduces visible waiting, terminal failure states, retries, retention rules, and a support surface. Do not impose that machinery on a disposable preview. Use it when the PDF's life is longer than the request that started it.

## Template ownership decides what a job must freeze

The important design question is not "can this renderer run in a worker?" It is who owns the template and its revision history. In a monthly-report flow, the application team may own the data model while a compliance or document team owns the approved report template. The background contract needs to name the handoff. A queued record that says only `month = 2026-08` leaves a worker free to choose whatever template is current when it finally runs.

Freeze the inputs that make a rendition meaningful: the reporting period, a source snapshot identifier, the template revision, and the retention-policy revision. Then make retries refer to that immutable render identity. The cost is additional stored state and a migration path when the contract changes. The benefit is concrete: an operator can tell the difference between a report that was never rendered, one whose file was stored before the worker stopped, and one that is ready for authorized retrieval.

This is where a one-shot endpoint commonly gets awkward. The rendering step can finish, the archive write can finish, and the request can end before the application records completion. A retry then has no honest way to decide between returning the existing artifact, recording the successful write, and producing a duplicate. An idempotency key backed by a uniqueness constraint is more reliable than a preliminary "does it exist?" lookup, because two concurrent requests can both observe that no record exists.

PDF itself has a published standard in ISO 32000-2. Conformance matters for interchange. It does not prove which snapshot or template produced a particular report; those links are application records around the artifact.

## A small contract that survives retries

The public API can remain small. Create or retrieve a render record using an idempotency key, expose its status, and permit artifact retrieval only for a ready record. The implementation below uses generic storage, queue, renderer, and archive interfaces so the operational rule stays visible: readiness follows archive persistence.

```ts
type RenderState = "queued" | "rendering" | "ready" | "failed";

type MonthlyReportRequest = {
  reportingPeriod: string;
  sourceSnapshotId: string;
  templateRevision: string;
  retentionPolicyRevision: string;
  idempotencyKey: string;
};

type RenderJob = {
  id: string;
  state: RenderState;
  request: MonthlyReportRequest;
  artifactId?: string;
};

declare const jobs: {
  createOrGet(request: MonthlyReportRequest): Promise<RenderJob>;
  claim(id: string): Promise<RenderJob | undefined>;
  markReady(id: string, artifactId: string): Promise<void>;
};

declare const queue: {
  publish(message: { kind: "render-monthly-report"; jobId: string }): Promise<void>;
};

declare const renderer: {
  render(input: {
    sourceSnapshotId: string;
    templateRevision: string;
  }): Promise<Uint8Array>;
};

declare const archive: {
  store(input: { renderId: string; bytes: Uint8Array }): Promise<{ id: string }>;
};

async function requestMonthlyReport(
  input: MonthlyReportRequest,
): Promise<RenderJob> {
  const job = await jobs.createOrGet(input);
  await queue.publish({ kind: "render-monthly-report", jobId: job.id });
  return job;
}

async function renderMonthlyReport(jobId: string): Promise<void> {
  const job = await jobs.claim(jobId);
  if (!job || job.state === "ready") return;

  const bytes = await renderer.render({
    sourceSnapshotId: job.request.sourceSnapshotId,
    templateRevision: job.request.templateRevision,
  });
  const artifact = await archive.store({ renderId: job.id, bytes });
  await jobs.markReady(job.id, artifact.id);
}
```

`createOrGet` needs a database-enforced uniqueness rule over the idempotency key and the caller's relevant scope. Without that rule, retries can create more than one job. `claim` needs a lease or equivalent ownership rule so two workers do not process the same job at once. Those are implementation choices, but the contract should make their outcome observable.

Test the uncomfortable sequences before calling the pipeline done: submit the same key twice, interrupt a worker after the archive write, change a template while a job waits, and attempt retrieval after access is revoked. Track accepted-to-ready time, failed terminal jobs, retry counts, and artifacts per render identity. Averages can look calm while one stuck monthly report is the only one an operations team needs.

## Do background jobs make every report more reliable?

No. They make a different set of guarantees possible. A job with mutable inputs can faithfully create the wrong revision. A job with no durable status can leave the caller guessing. A queue whose ready state appears before archive persistence can report success for a file that cannot be retrieved.

The opposing objection is fair too: "Why make a user wait for an ordinary download?" Do not. For a bounded preview with no archive requirement, synchronous generation keeps the interaction direct. For the monthly health-report record, the background path gives the system a durable place to coordinate template ownership, idempotency, access control, and recovery.

Choose based on the artifact's promised life. The report that must be available and explainable after its request ends belongs in a background job; the transient view can remain synchronous.

## Sources

References:

- https://www.rfc-editor.org/rfc/rfc9110#name-202-accepted
- https://www.iso.org/standard/75839.html
- https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
