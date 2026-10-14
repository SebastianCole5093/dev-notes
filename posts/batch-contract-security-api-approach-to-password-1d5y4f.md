# Batch Contract Security: API Approach to Password Protect and Encrypt PDF Files

TL;DR: Encrypt each signed contract before it enters email infrastructure, keep the password out of that email, and record an audit event for every transition. For a batch-heavy fintech workflow, throughput comes from bounded concurrency and backpressure, not from firing every PDF job at once. Password protection is a delivery control. It does not replace a digital signature, access authorization, or authenticated customer retrieval.

The useful before/after mental model is small. Before: sign, attach, and send, with one application request waiting on every step. After: sign, seal the immutable artifact, enqueue, encrypt in a worker, verify, then release to email. Each arrow emits an event. This separation makes the slow stage visible and keeps a burst of contracts from becoming a burst of timeouts.

## How should an API password-protect and encrypt a PDF?

Email is a handoff to infrastructure with different retention, access, and retry behavior. Once an unencrypted attachment crosses that boundary, later password protection cannot protect copies already accepted by a mail transfer agent, stored in a queue, or written to logs. The release rule should therefore be strict: the mailer may consume only an artifact whose encryption verification succeeded.

Keep three concepts separate. A contract signature supports integrity and signer intent according to the signing scheme. PDF encryption controls opening and permitted operations in conforming readers. Transport security protects a connection between mail systems. None is a substitute for the others. ISO 32000-2 defines PDF, including its security model; the application still owns password delivery, authorization, retention, and evidence.

Passwords need their own channel. A password derived from data printed in the same message is convenient, but it collapses two controls into one mailbox. Prefer an authenticated customer portal or another independently authenticated channel. Store neither the cleartext password nor a reusable derivation input in job payloads, logs, traces, analytics, or dead-letter records.

Permission flags are not a vault.

PDF permission flags are reader-enforced usage controls, not a guarantee that a recipient cannot copy rendered content. Make the open password and the operational release gate carry the security boundary.

## Build a batch pipeline that can prove what happened

The pipeline is a state machine, not one giant endpoint. Give each contract an immutable artifact identifier and a signing revision. Then move it through four explicit states: `SIGNED`, `ENCRYPTING`, `VERIFIED`, and `RELEASED`. A retry may repeat work for the same idempotency key, but it must not create a second customer delivery.

Diagram in words: the signing service writes an immutable PDF; the queue carries only identifiers; a bounded worker reads the source, encrypts to a new object, reopens that object with the intended password, hashes the encrypted bytes, and commits the verification event; the mailer reads only verified object identifiers. The audit store sits beside the path. It never carries document bytes or passwords.

That shape protects throughput in two ways. The request path finishes after durable enqueueing, and worker concurrency becomes an explicit capacity knob. PDF transforms can consume CPU and memory, so concurrency should be derived from load tests using representative page counts and input sizes. Pick a memory ceiling first. Increase workers until queue age stops improving or resource headroom reaches the operating limit, then leave room for retries. No universal batch size is honest here.

The audit trail should answer who requested the operation, which exact input revision was used, which policy and encryption implementation version ran, when each transition occurred, and which encrypted artifact was released. Record content hashes, byte counts, durations, outcome codes, and correlation identifiers. SHA-256 produces a 256-bit digest; that digest identifies bytes, while the contract revision identifies business meaning. Do not record secrets. For tamper evidence, each event can include the hash of the preceding event plus a canonical encoding of its own security-relevant fields. Protect the audit store with independent access controls and retention policy; a hash chain cannot prevent deletion by an administrator who controls the only copy.

That distinction matters.

## A copyable TypeScript worker boundary

Keep the PDF engine behind a narrow interface. This example deliberately leaves the standards-conforming encryption implementation replaceable. It shows the harder application guarantees: bounded concurrency, idempotent claims, post-encryption verification, and release only after an audit commit.

```ts
type ContractJob = {
  contractId: string;
  signedRevision: string;
  sourceObject: string;
  destinationObject: string;
  passwordRef: string;
};

type AuditEvent = {
  contractId: string;
  revision: string;
  stage: "ENCRYPTING" | "VERIFIED" | "RELEASED" | "FAILED";
  artifactSha256?: string;
  durationMs?: number;
  reason?: string;
};

interface PdfEncryptor {
  encrypt(input: Uint8Array, password: string): Promise<Uint8Array>;
  verifyOpen(encrypted: Uint8Array, password: string): Promise<void>;
}

interface Dependencies {
  claim(key: string): Promise<boolean>;
  readObject(key: string): Promise<Uint8Array>;
  writeObject(key: string, bytes: Uint8Array): Promise<void>;
  resolveSecret(reference: string): Promise<string>;
  appendAudit(event: AuditEvent): Promise<void>;
  releaseForEmail(contractId: string, objectKey: string): Promise<void>;
  encryptor: PdfEncryptor;
}

async function processContract(job: ContractJob, deps: Dependencies): Promise<void> {
  const key = `${job.contractId}:${job.signedRevision}:encrypt`;
  if (!(await deps.claim(key))) return;

  const started = performance.now();
  await deps.appendAudit({
    contractId: job.contractId,
    revision: job.signedRevision,
    stage: "ENCRYPTING",
  });

  try {
    const [source, password] = await Promise.all([
      deps.readObject(job.sourceObject),
      deps.resolveSecret(job.passwordRef),
    ]);
    const encrypted = await deps.encryptor.encrypt(source, password);
    await deps.encryptor.verifyOpen(encrypted, password);

    const digest = await crypto.subtle.digest("SHA-256", encrypted);
    const artifactSha256 = Buffer.from(digest).toString("hex");
    await deps.writeObject(job.destinationObject, encrypted);
    await deps.appendAudit({
      contractId: job.contractId,
      revision: job.signedRevision,
      stage: "VERIFIED",
      artifactSha256,
      durationMs: Math.round(performance.now() - started),
    });

    await deps.releaseForEmail(job.contractId, job.destinationObject);
    await deps.appendAudit({
      contractId: job.contractId,
      revision: job.signedRevision,
      stage: "RELEASED",
      artifactSha256,
    });
  } catch (error) {
    await deps.appendAudit({
      contractId: job.contractId,
      revision: job.signedRevision,
      stage: "FAILED",
      reason: error instanceof Error ? error.name : "UnknownError",
    });
    throw error;
  }
}

async function runBatch(
  jobs: ContractJob[],
  concurrency: number,
  deps: Dependencies,
): Promise<void> {
  if (!Number.isInteger(concurrency) || concurrency < 1) {
    throw new RangeError("concurrency must be a positive integer");
  }

  let cursor = 0;
  const worker = async (): Promise<void> => {
    while (cursor < jobs.length) {
      const job = jobs[cursor++];
      await processContract(job, deps);
    }
  };

  await Promise.all(
    Array.from({ length: Math.min(concurrency, jobs.length) }, worker),
  );
}
```

The interface is intentional. A library, sidecar, or internal service can implement `PdfEncryptor`, but the surrounding invariants stay testable. The resolver returns a secret only at execution time. The failure event stores an error class rather than the exception message, because exception text can contain paths, document metadata, or upstream payload fragments.

There is one ordering trade-off in the sample. `RELEASED` means the artifact was handed to the email boundary, not that a customer received or opened it. Delivery and bounce events belong later in the same correlation chain. Define those semantics in the runbook, or dashboards will tell a comforting but false story.

## Operate the queue, not just the endpoint

For throughput, watch queue age before raw request count. A growing oldest-job age says capacity is losing the race even when workers still report successful transforms. Pair it with transform duration by input-size bucket, active workers, retry rate, verification failures, and memory saturation. Never label a metric with a contract ID; that creates unbounded cardinality and leaks customer-linked identifiers into telemetry. Put the correlation ID in structured logs instead, under the same retention and access rules as the audit system. Alert on symptoms with customer impact. A sustained rise in queue age, any release without a corresponding verified artifact, or repeated failures for one signing revision deserves action. A single retryable transform failure may not. This distinction keeps alerts crisp during batch spikes. Test the failure edges: kill a worker after object write but before audit commit, deliver the same queue message twice, rotate the secret between enqueue and execution, feed in malformed and already-encrypted PDFs, and simulate a full batch larger than worker capacity. Then assert two invariants: no unverified object reaches email, and one contract revision produces at most one release decision. Deployment needs compatibility checks too. Pin the encryption implementation version, retain known-answer fixtures without real customer data, and open generated files with more than one standards-conforming reader used by the target population. Roll out behind a small worker pool while comparing verification failures and queue age. If a new implementation cannot reproduce required viewer compatibility, stop the rollout without changing signing or mail delivery.

Watch the oldest job.

## Isn't a password-protected attachment enough?

It is useful for reducing exposure of an attachment at rest in a mailbox or after forwarding, provided the password remains separate. It is weak as the only access model. Passwords can be shared, phished, reused, or disclosed alongside the file, and an opened document can still be photographed or re-exported.

For high-assurance contracts, an authenticated portal can offer revocation, access policy, and access-event evidence that a static attachment cannot. The trade-off is customer friction and a live service dependency. A practical decision rule is direct: use encrypted attachments when offline possession is a requirement and the residual sharing risk is accepted; use authenticated retrieval when revocation and per-access authorization matter. Some workflows may offer both, but each path needs the same signed-artifact identity and audit semantics.

## Can't we encrypt inline and skip the queue?

Yes, when volume is predictably low, documents are tightly bounded, and the caller can tolerate transform latency. Inline work has fewer moving parts. It also couples signing availability to PDF processing and email latency, which makes batch spikes harder to contain.

Choose with measurements from representative contracts. If signing requests must remain responsive while a scheduled batch releases many agreements, use durable asynchronous jobs and bounded workers. If immediate response is the dominant requirement, keep an inline path but enforce size and time limits, verify before returning success, and make client retries idempotent. The deciding constraint is not API fashion. It is where backpressure can safely wait.

The finished design has a clean release rule: **no verified encrypted artifact, no email handoff**. Keep password delivery independent, measure queue age, and preserve evidence around the exact signed revision. That gives a fintech team both batch control and an audit story without confusing encryption with identity.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Node.js Web Crypto API: https://nodejs.org/api/webcrypto.html
- NIST FIPS 180-4, Secure Hash Standard: https://csrc.nist.gov/pubs/fips/180-4/upd1/final
- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
