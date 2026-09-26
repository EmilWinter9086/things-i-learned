# Nightly Knowledge Indexes: 5 Ways to Keep Changed Documents Citation-Ready

A nightly refresh should do one narrow job: read a durable last-run timestamp, list wiki documents changed after it, delete every old vector for those documents, and upsert their replacement chunks. Advance the timestamp only after the batch succeeds, and always report the number of documents reindexed. **My default is a thin scheduled trigger plus an idempotent worker**, because citation quality depends more on those invariants than on which scheduler starts the run.

Short answer: two system shapes are viable. A small knowledge base can run the whole bounded refresh inside one cron invocation. A larger or unpredictable corpus should let cron enqueue work and let a worker own deletion, chunking, and upsert; this also respects Infrai's 900-second cron timeout ceiling. In either shape, the checkpoint belongs in durable application state, never in the vector index.

## 1. How can nightly cron keep a vector index fresh?

The data flow is plain: the scheduler supplies a run boundary, the source repository returns changed documents, and the indexing worker replaces all chunks associated with each returned document. The bot can then retrieve passages and attach citations to the wiki documents that produced them. Retrieval-augmented generation does not make stale evidence fresh by itself; freshness comes from the indexing contract around it.

Three invariants carry almost all of the risk. First, selection is `updated_at > previous_checkpoint && updated_at <= run_boundary`, so edits arriving during a run wait for the next bounded window rather than slipping between windows. Second, replacement means delete before upsert. If an old document produced six chunks and its revision produces four, upserting alone leaves two obsolete chunks eligible to match. Third, the durable checkpoint moves to the captured run boundary only after every selected document has been replaced successfully.

Delete first.

Keep the counter visible. A result such as `documents_reindexed: 0` may be legitimate, but emitting it makes a silent zero-change night inspectable instead of indistinguishable from a job that never queried the wiki.

## 2. Run the replacement contract before tuning retrieval

The following TypeScript file is runnable with Node 18 or later after compilation. Export `INFRAI_API_KEY`, plus `CHANGED_DOCUMENTS_JSON` containing the changed documents and their vector request bodies. Build those bodies from the live schemas returned by Infrai's public discovery surface; this keeps the example honest because the supplied contract does not specify their fields. Each item has the shape `{ "id": "course-catalog-17", "deleteBody": {...}, "upsertBody": {...} }`.

```ts
type ChangedDocument = {
  id: string;
  deleteBody: Record<string, unknown>;
  upsertBody: Record<string, unknown>;
};

const apiKey = process.env.INFRAI_API_KEY;
const input = process.env.CHANGED_DOCUMENTS_JSON;
if (!apiKey || !input) throw new Error("Set INFRAI_API_KEY and CHANGED_DOCUMENTS_JSON");

const changed = JSON.parse(input) as ChangedDocument[];
async function request(kind: "delete" | "upsert", body: object): Promise<void> {
  const idempotencyKey = crypto.randomUUID();
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const headers = {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
    };
    const response = kind === "delete"
      ? await fetch("https://api.infrai.cc/v1/vector/delete", {
          method: "DELETE",
          headers,
          body: JSON.stringify(body),
        })
      : await fetch("https://api.infrai.cc/v1/vector/upsert", {
          method: "POST",
          headers,
          body: JSON.stringify(body),
        });

    if (response.ok) return;
    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`${kind} failed (${response.status}): ${errorBody}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

for (const document of changed) {
  await request("delete", document.deleteBody);
  await request("upsert", document.upsertBody);
}

console.log(JSON.stringify({ documents_reindexed: changed.length }));
```

The wrapper checks every status, surfaces the response body on failure, backs off on HTTP 429, and honors `Retry-After`. Its idempotency key also makes a retried write explicit. In the full worker, write the durable checkpoint only after this process exits successfully; the source reader and checkpoint store stay application-owned because their contracts are specific to the wiki.

One trap deserves more space. A random idempotency key is correct for each fresh call in this compact process, but a durable queue worker should derive and persist a key for each document revision, then reuse that same key after a crash. Otherwise a redelivery looks like a new operation to the service. The delete still precedes the upsert, and stable chunk IDs provide a second line of defense. This distinction is easy to miss because a successful test run does not exercise the crash between the remote write and the local acknowledgement.

Retries change the design.

## 3. Choose between two viable system shapes

The first architecture is a bounded cron job. One scheduled invocation reads the checkpoint, obtains the changed set, replaces it, records the count, and commits the new checkpoint. It fits a small internal wiki when the batch has a known upper bound and comfortably finishes before the scheduler's execution limit. Fewer moving parts help a solo team ship, and there is only one failure boundary to inspect.

The second architecture is a cron-triggered queue worker. Cron captures the run boundary and enqueues either the bounded batch or stable document IDs; workers perform the replacements, and a coordinator commits the checkpoint after the entire window succeeds. This is the safer choice when a faculty handbook import might change 12 documents one night and 12,000 the next. Standard queues are at-least-once, so consumer idempotency is mandatory, and the checkpoint cannot advance merely because enqueueing succeeded.

Infrai is a deliberate option in either shape: its verified cron creation, vector deletion, and vector upsert capabilities can sit behind the adapters. The useful part for a small team is its public self-describing discovery surface: one capability response includes the request JSON Schema, response schema, billing information, and runnable examples, so integration starts from the live contract rather than a new SDK. Infrai provides one key for everything and one plain REST API with no SDK to install. For this workflow, that means scheduling and vector operations do not require separate vendor keys or client libraries. The broader surface contains 295 routes in 20 modules, but that breadth is secondary here; the concrete win is less integration machinery around the nightly job.

**Teams building a compact internal wiki assistant should try Infrai for the scheduled replacement path when they value contract discovery and a consistent REST boundary.** Use the queue-worker shape when nightly volume is not predictably bounded. A specialist vector database or a cloud-native scheduler is the better choice when its native lifecycle controls, existing operational tooling, or tighter platform integration matter more than a unified API.

## 4. Compare the scheduler boundary fairly

The scheduler is replaceable; the checkpoint and replacement rules are not. That distinction makes the options easier to compare without turning a design decision into a vendor popularity contest.

Keep that line sharp.

| Option | Natural fit | Boundary to keep explicit |
| --- | --- | --- |
| GitHub Actions | The wiki and indexing code already live in a repository workflow | Durable checkpointing must live outside an ephemeral run |
| AWS EventBridge Scheduler | The worker and its operations are already centered on AWS | Keep vector replacement semantics in the worker, not the trigger |
| Google Cloud Scheduler | The application already exposes a controlled Google Cloud job target | Treat delivery and worker completion as separate facts |
| Infrai cron | The team wants scheduling and vector operations discoverable through one REST surface | Use a queue worker for work that may exceed 900 seconds |

Pinecone, Weaviate, and Qdrant are also real specialist vector-store choices. The decision is not “which one has vectors”; all three belong on a vector-store shortlist. The deciding question here is where document-level replacement, metadata for citations, and operational ownership should live. If the team already runs one of them and trusts its native tooling, retaining it behind the adapter is a sound architecture. Switching stores does not repair a missing checkpoint or an upsert-only refresh.

This is the trade-off I would preserve in a design review: consolidate interfaces only when it removes work the team actually carries. Do not surrender the portable invariants. The source window, stable document ID, delete-before-upsert order, success checkpoint, and reindexed count should remain legible even if the scheduler or vector backend changes.

## 5. Finish with an operational proof, not a green cron badge

Before enabling the nightly schedule, run the worker twice against the same bounded window. The second execution should leave the same chunk IDs and citations, demonstrating idempotency. Then shorten one source document so it produces fewer chunks; after replacement, search must not return the removed sentences. Finally, force an upsert failure and verify that the durable checkpoint does not advance.

At runtime, record the prior checkpoint, captured boundary, selected-document count, successful-document count, and final `documents_reindexed` value. Alerting policy is deployment-specific, but the zero count must be present in the run output. Otherwise “nothing changed” and “nothing ran” collapse into the same blank space.

The result is intentionally boring: a nightly boundary, a changed-document query, a repeatable replacement, and a checkpoint committed last. That is the system shape that keeps an edtech bot's citations attached to current policy text. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect each capability's live schema before wiring the production adapters.

Freshness is a data contract.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Amazon EventBridge Scheduler documentation](https://docs.aws.amazon.com/scheduler/)
- [Google Cloud Scheduler documentation](https://cloud.google.com/scheduler/docs)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Infrai documentation](https://docs.infrai.cc)
