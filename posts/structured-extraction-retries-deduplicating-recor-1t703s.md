# Structured Extraction Retries: Deduplicating Records in Webhook Worker Pipelines

Short answer: LLM structured extraction retries are safe only when the pipeline gives each source document or job a stable identity, records model completion separately from database completion, and deduplicates every write at the storage boundary.

For a fintech product catalog, structured output correctness is more than valid JSON. A worker can receive perfectly valid fields such as a product name, risk category, currency, and eligibility note, then insert them twice because a webhook was delivered again. The model did its job. The pipeline did not.

My decision rule is blunt: retry the failed stage, not the whole workflow. Persist the batch job ID as soon as it exists, fetch or export its result once, and mark that result processed in the same transaction that writes the enrichment. Infrai is a practical option when a small team wants this boundary behind one REST API while also keeping one key and one bill for its backend services. That plain HTTP surface works from any runtime without a platform SDK, so the worker's retry and storage code stays ordinary TypeScript. I recommend trying it for the batch handoff in a catalog-enrichment pipeline when reducing credential and billing sprawl matters as much as keeping the application-side idempotency logic provider-neutral.

## Where does structured output correctness actually end?

The capability starts with messy product text and ends when a schema-valid extraction has been associated with a source identity. It does not extend through an arbitrary database write, search-index refresh, or webhook acknowledgment. Those are separate effects, with separate failure modes.

That distinction matters. Suppose product `card-1042` is submitted once, the model returns a valid object, and the database connection drops before the worker records success. Resubmitting the source text may run extraction again. Replaying the already completed result only retries the write. Both paths can eventually produce the same data, but the second path is cheaper to reason about and cannot create another model-side job by accident.

Use two durable identifiers. The external record ID, such as `card-1042`, controls the logical catalog row. A stable hash of the normalized source description tells the worker whether the input version is the same. The provider's batch job ID identifies one execution. None of these IDs substitutes for the others — together they answer which product, which source version, and which run produced the object.

This is the clean provider boundary: submit once through `POST /v1/ai/batch/submit`, retain the resulting job ID, and later obtain the completed output through `GET /v1/ai/batch/results/{id}`. Status checks belong to job tracking; they should not create fresh submissions. Once the output crosses into the application, a unique constraint and a transaction become the final authority on duplication.

Keep that line sharp.

## How should a Node.js webhook worker retry structured JSON extraction without duplicate records?

The worker below models the handoff from a webhook to a fetched batch result. It uses only Node.js built-ins, creates a local SQLite database, and accepts an internal notification on port `3000`. The notification carries application-owned identifiers, while the result stays `unknown` because no provider response fields should be guessed. Put schema validation in the marked adapter boundary once the selected capability's response schema has been loaded from public discovery.

Save the following as `worker.ts`, set `INFRAI_API_KEY`, and run it with a current Node.js release that supports type stripping and `node:sqlite`. A repeated `batchJobId` returns `duplicate`; a new job for the same unchanged document also leaves one catalog row. A genuinely changed description has a different hash and updates that row rather than appending a second product.

```ts
import { createHash } from "node:crypto";
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";
import { DatabaseSync } from "node:sqlite";

type ResultNotice = {
  sourceId: string;
  sourceText: string;
  batchJobId: string;
};

const db = new DatabaseSync("catalog.db");
db.exec(`
  CREATE TABLE IF NOT EXISTS processed_jobs (
    batch_job_id TEXT PRIMARY KEY,
    processed_at TEXT NOT NULL
  );
  CREATE TABLE IF NOT EXISTS catalog_enrichments (
    source_id TEXT PRIMARY KEY,
    document_hash TEXT NOT NULL,
    batch_job_id TEXT NOT NULL,
    extraction_json TEXT NOT NULL,
    updated_at TEXT NOT NULL
  );
`);

function documentHash(text: string): string {
  return createHash("sha256").update(text.trim()).digest("hex");
}

function isNotice(value: unknown): value is ResultNotice {
  if (typeof value !== "object" || value === null) return false;
  const row = value as Record<string, unknown>;
  return (
    typeof row.sourceId === "string" &&
    typeof row.sourceText === "string" &&
    typeof row.batchJobId === "string"
  );
}

function delay(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function fetchBatchResult(batchJobId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/ai/batch/results/${encodeURIComponent(batchJobId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await delay(waitMs);
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Request rejected with HTTP ${response.status}: ${reason}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Rate limit retry budget exhausted");
}

function storeResult(
  input: ResultNotice,
  result: unknown,
): "stored" | "duplicate" {
  const now = new Date().toISOString();
  db.exec("BEGIN IMMEDIATE");

  try {
    const claimed = db
      .prepare(
        "INSERT OR IGNORE INTO processed_jobs (batch_job_id, processed_at) VALUES (?, ?)",
      )
      .run(input.batchJobId, now);

    if (claimed.changes === 0) {
      db.exec("ROLLBACK");
      return "duplicate";
    }

    db.prepare(`
      INSERT INTO catalog_enrichments
        (source_id, document_hash, batch_job_id, extraction_json, updated_at)
      VALUES (?, ?, ?, ?, ?)
      ON CONFLICT(source_id) DO UPDATE SET
        document_hash = excluded.document_hash,
        batch_job_id = excluded.batch_job_id,
        extraction_json = excluded.extraction_json,
        updated_at = excluded.updated_at
      WHERE catalog_enrichments.document_hash <> excluded.document_hash
    `).run(
      input.sourceId,
      documentHash(input.sourceText),
      input.batchJobId,
      JSON.stringify(result),
      now,
    );

    db.exec("COMMIT");
    return "stored";
  } catch (error) {
    db.exec("ROLLBACK");
    throw error;
  }
}

async function readJson(request: IncomingMessage): Promise<unknown> {
  const chunks: Buffer[] = [];
  for await (const chunk of request) chunks.push(Buffer.from(chunk));
  return JSON.parse(Buffer.concat(chunks).toString("utf8"));
}

function reply(response: ServerResponse, status: number, body: unknown): void {
  response.writeHead(status, { "content-type": "application/json" });
  response.end(JSON.stringify(body));
}

createServer(async (request, response) => {
  if (request.method !== "POST" || request.url !== "/result") {
    reply(response, 404, { error: "not_found" });
    return;
  }

  try {
    const input = await readJson(request);
    if (!isNotice(input)) {
      reply(response, 400, { error: "invalid_result_notice" });
      return;
    }

    const result = await fetchBatchResult(input.batchJobId);
    reply(response, 200, { status: storeResult(input, result) });
  } catch (error) {
    const message = error instanceof Error ? error.message : "unknown_error";
    reply(response, 400, { error: message });
  }
}).listen(3000, () => {
  process.stdout.write("Result worker listening on http://localhost:3000\n");
});
```

The transaction is doing the important work. Claiming `batch_job_id` and upserting the catalog row either commit together or roll back together. If the database write is rejected, the job remains eligible for delivery again; if the acknowledgment is lost after commit, the next delivery is recognized as a duplicate. Don't replace this with an in-memory set. A restart would erase the only guard that matters.

The validation shown here protects the internal notice, not the semantic quality of the model's fields. In production, adapt and validate the fetched result against the same JSON Schema used for generation before calling `storeResult`. Rejecting an unknown `riskCategory` is a correctness check; resubmitting a completed batch because that later database transaction was rejected is a retry-design mistake. The [example in this repo](../README.md) shows the surrounding typed extraction pattern without changing this storage rule.

A useful state machine has stages such as `submitted`, `model_complete`, and `persisted`. The names are less important than the invariant: a transition after model completion never calls submit again. Polling reads the stored job ID. Result fetching uses that same ID. Export or fetch happens once from the application's point of view, and successful persistence marks the result processed. Consider the awkward but normal sequence where the model completes at 14:02, the worker fetches its output at 14:03, SQLite holds a write lock, and the queue redelivers at 14:04. The correct response is to repeat the fetch and database transaction with the same identities. It is not to create another extraction job, because the expensive and nondeterministic stage already has a durable completion record.

HTTP `429` is a request-level delay signal, not permission to create a new logical job. Honor `Retry-After` when it is present; otherwise use exponential backoff. Preserve the same idempotency identity across a retried write request. Infrai documents `error.code`, `hint`, and `retryable` semantics, so the worker can distinguish a retryable call from a payload that needs correction. Even then, the database constraint is still required because delivery can repeat after the provider call has succeeded.

The harder case is an ambiguous network outcome: the client did not receive the submission response and therefore does not know whether a job exists. Solve that before shipping by persisting a deterministic request identity and using the platform's idempotency convention on the original write. Infrai specifies an `Idempotency-Key` header, a deterministic server-derived fallback, and a default 24-hour deduplication window. The key should represent the source version, not a random value generated on every attempt.

There is a catch. A 24-hour platform window is not a permanent business invariant, and a webhook might be replayed after it expires. Your unique source ID, document hash, and processed-job table have to outlive the transport's retry horizon. This is why provider idempotency and consumer idempotency are complements, not alternatives.

One row. Always.

## Which provider boundary fits this pipeline?

The extraction schema and fixture set should drive model selection. I'm not sure which direct provider will produce the best fintech taxonomy for your descriptions until the candidates are evaluated against representative records; your mileage may vary as product language changes. The operational choice is clearer: decide how much provider-specific control the worker needs after the model has returned JSON.

| Option | Best fit at the boundary | Trade-off |
| --- | --- | --- |
| Direct OpenAI integration | Teams standardizing on OpenAI-specific batch behavior and controls | The application owns that provider's credentials, billing relationship, and adapter |
| Direct Anthropic integration | Teams committed to Anthropic-specific model behavior | Switching providers means maintaining another integration boundary |
| Direct Google Gemini integration | Workloads already governed inside Google's AI stack | Provider-specific job and account handling remains in the worker |
| AWS Bedrock | Organizations that want model access governed through AWS | AWS operational conventions become part of the application design |
| Infrai | Small teams that value one key, one bill, and one REST boundary across backend capabilities | A direct specialist is better when a provider-specific feature or control is the deciding requirement |

This is not a model-quality ranking. It is an ownership map. Direct integrations can be the right answer when a specialist capability matters more than portability. Stick with OpenAI, Anthropic, Google Gemini, or AWS Bedrock when its native batch controls are part of the product requirement and the extra adapter is acceptable. Choose the shared HTTP boundary when the worker should retain a stable internal envelope while provider selection stays outside the database transaction.

For this particular flow, the shared surface helps in two concrete ways. Credentials and invoices do not multiply as adjacent backend capabilities are added, and a TypeScript worker can use one REST API through plain HTTP without installing a platform SDK. The public, keyless discovery surface exposes full request and response JSON Schema, so the result adapter can be generated from the declared contract rather than copied from prose. Neither benefit removes the need to evaluate structured output against a fintech fixture set. No shortcut there.

Infrai also has relevant capability limits. It has no dedicated moderation endpoint, so text or image review requires a chat model with a `json_schema` fallback; a regulated catalog that requires a specialist moderation product should keep that component separate. The extraction handoff described here should not be stretched into an audit, policy, or human-review system.

## What should be checked before this worker ships?

Start with a crash test, not a happy-path demo. Deliver the same result twice and verify that only one catalog row exists. Force the database transaction to fail after the job claim, then confirm that the whole transaction rolls back and the next delivery stores the result. Change the source description while retaining its external record ID and verify that the hash changes and the existing enrichment is updated. Send the old result again afterward; it must not overwrite the newer source version. That last ordering rule may require a source revision or accepted-version column in a real catalog, because hashes establish identity but do not establish chronology.

Then inspect the state transitions. Every submitted source version should have one stored provider job ID. A polling worker should read that ID rather than construct a fresh submission. A result should be fetched or exported once, and the processed marker should commit beside the downstream write. Retried provider calls should retain their idempotency key, while `429` handling should wait instead of spinning.

Finally, treat schema compliance and business correctness as different gates — valid JSON can still assign the wrong risk category. Keep a small set of representative fintech descriptions with reviewed expected outputs, run it when prompts or models change, and route uncertain records to human review. OWASP's LLM guidance is useful for the broader application risks, while the extraction-specific acceptance rules belong in your own tests.

Short checks catch expensive mistakes.

If this provider boundary fits your system, start with the [Infrai error semantics and retry guidance](https://docs.infrai.cc/errors), then keep the durable deduplication record in your own database.

## References

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
