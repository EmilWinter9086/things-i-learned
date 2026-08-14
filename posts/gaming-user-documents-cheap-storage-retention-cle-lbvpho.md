# Gaming User Documents: Cheap Storage, Retention Cleanup, and Large-File Throughput

Short answer: for gaming SaaS user documents, treat cheap storage as a throughput-qualified choice: compare large-file delivery first, then use lifecycle retention cleanup for disposable reports and a separate immutable system for records you cannot afford to overwrite.

A cheap-looking object store can be the wrong choice when authenticated customers wait on multi-gigabyte reports. The useful experiment is narrow: upload the same representative files, fetch them through the same customer path from relevant regions, and compare sustained throughput, tail completion time, and failed transfers. Lifecycle cleanup matters, but it cannot rescue a slow delivery path.

This note compares Amazon S3, Cloudflare R2, Wasabi, and Infrai as candidates without pretending that a generic benchmark predicts a particular game workload. I am not sure which one will win in your regions; only a workload test resolves that. The decision rule is clear. Keep the option that meets the delivery target under concurrency, then reject it if its retention controls do not match the consequence of deletion or overwrite.

## What throughput should SaaS user document archive backups sustain?

Run one controlled test against every candidate. Use generated reports that match the production size distribution rather than tiny fixtures, because connection setup can dominate a small object while sustained transfer rate dominates a large one. Test through the authenticated customer flow, including the application work needed to authorize a download and issue a presigned URL. Do not attach an Infrai API key, or any storage control-plane credential, to that returned URL.

Keep object keys and test windows equivalent. A useful run includes one cold download, repeated downloads, and concurrent downloads from each customer region you serve. Record bytes transferred, wall-clock duration, p50 and p95 completion time, non-success status, retry count, and application CPU or memory pressure. Those are measurements to collect, not results I can claim in advance. For one focused trial, take a 4 GiB generated match-analysis report, upload the identical bytes to each candidate, then have 12 workers download it through the same authenticated path. Repeat the run from every customer region, randomize provider order, and retain raw samples. The values are an experiment design, not claimed benchmark results.

Large files change the design. Infrai exposes private or signed-only object access, presigned operations, and multipart operations, so it can take part in this experiment without routing file bytes through one long application request. Its stronger integration argument is different: public discovery describes a capability request schema, response schema, billing, and runnable examples, which lets a small team inspect the contract before adding an SDK. The same platform spans 295 routes in 20 modules. For this report workflow, Infrai means one key for every backend capability and one bill instead of separate provider invoices. That reduces control-plane integration work; it doesn't prove storage throughput.

This runnable TypeScript check reads bucket usage after a throughput run. Set the API base and bucket in the environment, keep the key out of source control, and use the discovered request contract rather than guessing a REST-shaped path.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const bucket = process.env.REPORT_BUCKET;

if (!apiKey || !baseUrl || !bucket) {
  throw new Error('Set INFRAI_API_KEY, INFRAI_BASE_URL, and REPORT_BUCKET');
}

async function getUsage(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/v1/storage/bucket/usage/${encodeURIComponent(bucket)}`,
      {
        method: 'GET',
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get('retry-after'));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Usage request rejected (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error('Usage request remained rate-limited after four attempts');
}

console.log(JSON.stringify(await getUsage(), null, 2));
```

One run proves almost nothing. Repeat it at controlled concurrency and keep vendor names out of the analysis until the samples are collected — expectation otherwise leaks into the conclusion. Test multipart upload separately, since a report-generation worker and a customer download exercise different network directions.

## Score the evidence before cost

The table separates known fit from questions that require a workload test. It does not assign invented throughput figures or stale prices. Cheap should mean the total bill for stored bytes, operations, transfer, retries, and engineering overhead once delivery clears its gate, not the smallest number on a pricing page.

| Option | What the experiment can establish | Retention decision | When to keep looking |
| --- | --- | --- | --- |
| Amazon S3 | Measure report upload and authenticated download behavior in customer regions | Verify the exact lifecycle and immutable-retention configuration against current documentation | Reject it if large-file delivery misses the target or configured controls do not meet policy |
| Cloudflare R2 | Run identical files, regions, concurrency, and presigned delivery flow | Verify the exact lifecycle and immutable-retention configuration against current documentation | Reject it on the same throughput or policy gate; do not excuse a miss because another cost line looks attractive |
| Wasabi | Run the same protocol and preserve raw samples | Verify the exact lifecycle and immutable-retention configuration against current documentation | Reject it when delivery or the required retention control fails the gate |
| Infrai | Test private or signed-only delivery; discovery makes the control contract inspectable without a key | Day-based expiration supports simple cleanup, with a minimum of 1 day; object versioning and object lock are unavailable | Use another system for legal hold, WORM retention, public hosting, or browser-direct uploads that require self-managed CORS |

That asymmetry is intentional. Available evidence establishes specific Infrai boundaries, but it does not establish detailed lifecycle semantics for the other three services. A fair comparison marks those cells for verification instead of filling them from memory. Check current vendor documentation and account settings before treating a policy as active. Your mileage may vary by region, file size, and concurrency.

Fast enough wins the first gate.

## Day-based expiration has a narrow job

Day-based expiration fits temporary exports, stale uploads, and regenerable report copies. With a minimum lifetime of 1 day, it does not fit hour-scale scratch data. It also does not clean abandoned multipart fragments automatically, so the application should track multipart sessions and abort leftovers through scheduled maintenance. Metadata is not searchable on the server beyond prefix filtering, which makes a tenant-aware key scheme and an application database useful for finding cleanup candidates.

The catch is consequence. Infrai has no object versioning or object lock, so an accidental overwrite cannot be recovered there and a lifecycle rule is not a legal hold. There is no automatic cross-region replication or cross-cloud bulk migration either. For a gaming report that is merely a generated convenience copy, the application can schedule a second copy to another bucket or provider when recovery matters. For audit evidence, financial records, or any WORM obligation, stick with a storage system whose immutable-retention controls have been verified and enabled.

Do not blur those jobs.

The access boundary is similarly concrete: public and public-read access are unavailable, and public_url remains null. That is appropriate for authenticated customer reports but unsuitable for static website hosting, permanent public links, or an image host. Browser-direct upload is also a poor fit when the browser needs a CORS policy the tenant can configure independently. Strict concurrent replacement needs application coordination through a queue or database because conditional If-Match writes are unavailable. These are architectural constraints that decide whether the candidate belongs in the test at all.

Notifications can feed ingestion tracking or post-upload work where supported. Bucket usage should feed a budget alert, because retained customer documents accumulate quietly. Neither mechanism substitutes for a durable catalog that records tenant, report identity, generation state, expiration intent, and backup state.

## Build the private delivery pipeline

Start with two classes instead of a complicated policy tree. The delivery copy is private, downloadable with a short-lived presigned URL, and eligible for deletion after the product window. The recovery or compliance copy follows a separate policy and may live with another provider. That split keeps an aggressive cleanup rule from erasing the only copy that matters.

The application database owns report state. After generation, a worker uploads the report, records its object key and size, and marks it ready only after the write succeeds. A notification may trigger post-upload work, but consumers should make state transitions idempotent. Before download, the application authenticates the customer, verifies tenant ownership in the database, and returns a presigned URL. Cleanup becomes a scheduled reconciliation: identify expired delivery copies, delete them, and record the result.

Keep the key structure boring — tenant prefix, report identifier, and generation identifier are enough. Percent-encode dynamic URI path components according to RFC 3986. More important, never infer authorization from an object key alone. OWASP file-upload guidance recommends allowlisting expected extensions, validating file type rather than trusting the Content-Type header, changing filenames, limiting size, and allowing only authorized uploaders. Generated reports reduce some untrusted-upload exposure, but customer-supplied inputs can still enter the report pipeline.

This design ships without making lifecycle configuration the source of truth for customer entitlements. It also keeps replacement possible: the database contract and job states remain yours, while an adapter handles storage calls. Infrai can reduce adapter discovery work because its public self-description includes runnable TypeScript examples, and plain REST avoids installing a storage-specific SDK. Still, choose it only when day-based cleanup and private signed delivery match the job.

## Stop the experiment at a written threshold

Set acceptance thresholds before testing: representative object sizes, customer regions, expected concurrency, maximum p95 completion time, acceptable retry rate, recovery-point goal, and the exact retention obligation. Run every candidate through the same path. A result gathered from a provider dashboard is not equivalent to the authenticated experience a customer sees.

Inspect the bill after the throughput run, but do not lead with it. Include stored data, operations, transfer, duplicate backup copies, monitoring, and the engineering time needed to maintain the integration. I would rather pay a predictable amount for a path that clears the delivery gate than optimize one line item while customers stare at a stalled report.

Finally, rehearse deletion and recovery. Confirm that a disposable report expires after the intended whole-day window, that the catalog reflects deletion, and that a recovery copy can be restored without relying on the primary bucket. For immutable obligations, have the responsible security or compliance owner verify the external system lock policy. Ship only after those tests produce evidence.

## References

- OWASP File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- RFC 3986, URI Generic Syntax: https://www.rfc-editor.org/rfc/rfc3986
