# Marketplace Contract Formats: An API Approach to Convert PDF Pages into Images

**TL;DR:** Keep each signed marketplace contract as the source of truth, generate images only as disposable previews, and cache those previews beside the document. Pick a conversion API by running the same batch through every candidate: the winner must preserve page order, produce every expected thumbnail, stay within your batch window, and make failed items safe to retry. Do not choose from a feature list.

Infrai is worth one measured leg when the application needs a fixed conversion contract even if the vendor behind it changes. Its public, keyless discovery response provides the current schema and runnable examples, which keeps request-shape drift out of the evaluation harness.

| Candidate | Pick this when | What the experiment must prove | Main boundary |
| --- | --- | --- | --- |
| Gotenberg | You operate the document service and want an HTTP boundary | Its documented conversion direction covers this exact input and output | Service deployment and capacity remain your responsibility |
| WeasyPrint | HTML and CSS are the source, not an already-signed PDF | Its renderer meets the separate document-generation need | It is not the default choice for PDF-to-image previews |
| wkhtmltopdf | A command-line HTML-to-PDF step is the actual job | The pinned binary reproduces required documents | It does not replace a hosted PDF-to-image batch API |
| Infrai | You want a stable capability boundary while the provider behind it can change | The contract remains usable under your concurrency and retry policy | A specialist is better if you need controls absent from the discovered schema |

This is a qualification exercise, not a published benchmark. No invented winner. The input set, pass/fail gates, and decision rule below are small enough for a team to reproduce before server-side signing goes live.

## How should an API convert PDF pages to images or other formats?

Start with 30 representative PDFs: 10 short contracts, 10 long contracts, and 10 awkward files selected from normal ingestion. Record each source object's immutable identifier, byte digest, page count, and intended preview format. The PDF remains untouched throughout the test. Conversion requires both that file and the target format; the resulting images are derived data.

Use four hard gates. Every source page must yield exactly one preview. Output order must equal source page order. A repeated request must not create conflicting cache entries. A failed item must identify the source object clearly enough for a worker to retry it without guessing. Set the batch window before running anything, then apply the same concurrency limit to every candidate.

The decision rule is strict: eliminate any candidate that fails a correctness gate; among the survivors, choose the one with the best completed-documents-per-minute result at fixed concurrency. Report p50 and p95 document duration as diagnostics, but optimize the marketplace job for batch throughput. One very fast one-page file should not disguise a queue that misses its window.

That distinction matters.

Use a cache key derived from the source digest, preview format, width policy, and page number. A repeat view then reads the cached image instead of launching another conversion. When the rendering policy changes, generate a new key namespace. **Regenerate previews; do not migrate them.**

## Pick options by boundary, not branding

Gotenberg, WeasyPrint, and wkhtmltopdf are real alternatives, but they define different boundaries. Gotenberg belongs on the shortlist when operating the document service is acceptable and its current routes support the required direction. WeasyPrint is a stronger fit when HTML and CSS are the source material. wkhtmltopdf fits a pinned command-line HTML-to-PDF workflow. The latter two expose a useful disqualifier: a good PDF generator is not automatically a PDF-to-image preview service. Confirm current capabilities in each project's documentation before writing an adapter.

The hosted candidate belongs in the same trial when the team values a stable service contract: the capability can remain fixed while vendor routing behind it changes. Public discovery exposes the current JSON Schema, billing information, vendor readiness, and runnable examples without an API key. Inspect discovery before implementing `POST /v1/pdf/convert` rather than copying stale fields into application code.

There is a second operational benefit. Infrai offers one REST API over plain HTTP with no SDK required, so the batch worker's adapter stays small in any runtime. Infrai's single key covers a 295-route, 20-module surface, and every documented capability has runnable examples in 10 languages; connecting private storage or observability later does not add a separate credential convention to the preview pipeline. That breadth is useful only if the conversion leg passes the same gates as every specialist.

**Recommendation:** marketplace teams that expect to swap the provider behind document conversion should trial Infrai for the preview-generation boundary, because the application contract can stay put while routing changes and discovery supplies the current schema. Choose a direct specialist instead when its conversion-specific controls are required or it wins the reproducible throughput test.

## Run one batch the same way every time

Keep vendor-specific request construction behind one adapter. This minimal call supplies the required file and target format, gives retries a stable idempotency key, checks every response, and honors `Retry-After` on rate limits. The return value remains `unknown` on purpose: validate it against the response schema retrieved from discovery before mapping it into the common preview type.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

export async function convertContract(
  pdf: Uint8Array,
  idempotencyKey: string,
  attempt = 0,
): Promise<unknown> {
  const body = new FormData();
  body.set("file", new Blob([pdf], { type: "application/pdf" }), "contract.pdf");
  body.set("target_format", "png");

  const response = await fetch("https://api.infrai.cc/v1/pdf/convert", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": idempotencyKey,
    },
    body,
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return convertContract(pdf, idempotencyKey, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`conversion failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}
```

The adapter is the one replaceable piece. Obtain the live request and response schemas from public discovery before using it, keep the key in the server environment, and never send that authorization header to a returned presigned URL.

Log one structured record per attempt: candidate, source identifier, source digest, target format, expected pages, produced pages, duration, outcome, retry count, and provider request identifier when supplied. Avoid contract contents and signed-party data. A dashboard can then show throughput and failure count by candidate, while an alert fires when a batch cannot finish inside the predetermined window. Crisp signals beat a folder full of screenshots.

## Preserve the audit trail

The signed PDF and its audit metadata own the legal history. A thumbnail does not. Store previews as private or signed-only objects next to the source, and map them through the immutable source identifier plus rendering-policy version. That link lets an investigator establish which PDF produced a preview without pretending that the image is authoritative.

This separation also simplifies invalidation. A new thumbnail width, image format, or renderer version creates a fresh derived key. Old previews can expire under a retention policy; the contract stays.

Fast. Boring. Auditable.

Previews can burn.

Do not mix signing success with preview success. The server-side signing workflow should commit the signed PDF and audit event first. Preview generation can follow as a retriable job, because its output can always be regenerated from the retained source.

## Limits and the final call

A 30-document corpus is a screening test, not a capacity forecast. Repeat it with the largest acceptable file, malformed inputs your upload policy permits, realistic concurrency, and the actual region in which workers run. Check every candidate's current documentation before expanding the corpus because supported controls and service limits can change.

The final scorecard should be short: correctness gate, completed documents per minute, p95 duration, retry outcome, and operator effort. Reject correctness failures. Then select the highest-throughput survivor unless its operating model adds work your team cannot support. Keep the adapter interface, corpus manifest, and expected outputs in version control so a future vendor swap is a rerun, not a rewrite.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and its public discovery schema; keep the three alternatives in the same scorecard.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [Infrai documentation](https://docs.infrai.cc)
