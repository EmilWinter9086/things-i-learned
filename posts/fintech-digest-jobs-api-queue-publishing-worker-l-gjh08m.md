# Fintech Digest Jobs — API Queue Publishing, Worker Leases, and Idempotency Keys

Short answer: accept the weekly-digest request, commit its idempotency record and queued job in one short database transaction, return `202 Accepted`, then let leased workers build the digest and publish through a separately deduplicated outbox. This keeps API latency independent of model and data-fetch time without paying for a dedicated queue service before the workload needs one.

For a fintech digest, speed and cost pull in opposite directions. Customers should not hold an HTTP connection while transaction summaries, disclosures, and personalized text are assembled. Yet a solo team may not want another broker, control plane, and bill for one weekly workload. A PostgreSQL work queue is a defensible starting point when the database already exists, provided the design treats retries and duplicate publication as normal events.

The data flow is plain: an Express handler validates a stable business key, inserts the request and job atomically, and responds. A consumer claims one due row with a time-bounded lease. It performs slow work after releasing the row lock, writes a publication record under a unique constraint, and marks the job complete. A separate sender can drain that publication table using the same claim pattern. The boundary matters more than the library.

Commit first.

## The transaction boundary comes before the worker

The API should publish by committing durable state, not by calling an in-process function after sending the response. That latter pattern looks fast in a demo but loses work when the process exits between the response and the function call. Publishing before the request record commits is no better: a worker can observe a job whose business state does not exist yet. One transaction removes both gaps.

Use one idempotency key for one customer and one digest period. Keep a canonical hash of the accepted inputs beside it. A replay with the same key and same hash returns the original request; reuse with different inputs returns `409 Conflict`. The database also needs a unique business constraint on `(customer_id, week_start)`, because clients can accidentally mint two keys for the same weekly digest.

A job lease is deliberately temporary. The claim transaction changes `ready` to `running`, sets `lease_until`, and commits quickly. If the worker stops before completion, a reaper makes the row eligible after the lease expires. This is at-least-once processing. Good. Pretending the queue alone gives exactly-once effects is the expensive mistake.

Retries are normal.

## Run the request-to-lease path

The focused example below uses Express and `pg`, but its contract is portable: short transactions for enqueue and claim, slow work outside locks, and uniqueness at the publication boundary. The schema can be installed through the same PostgreSQL client during development; production migrations should run separately from application startup.

```ts
import express from 'express';
import { createHash } from 'node:crypto';
import { Pool, PoolClient } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const app = express();
app.use(express.json());

const schema = `
  CREATE TABLE IF NOT EXISTS digest_requests (
    id bigserial PRIMARY KEY,
    customer_id text NOT NULL,
    week_start date NOT NULL,
    idempotency_key text NOT NULL UNIQUE,
    request_hash text NOT NULL,
    status text NOT NULL DEFAULT 'queued',
    created_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (customer_id, week_start)
  );

  CREATE TABLE IF NOT EXISTS digest_jobs (
    id bigserial PRIMARY KEY,
    request_id bigint NOT NULL UNIQUE REFERENCES digest_requests(id),
    status text NOT NULL DEFAULT 'ready',
    available_at timestamptz NOT NULL DEFAULT now(),
    lease_until timestamptz,
    attempts integer NOT NULL DEFAULT 0
  );

  CREATE TABLE IF NOT EXISTS digest_publications (
    request_id bigint PRIMARY KEY REFERENCES digest_requests(id),
    payload jsonb NOT NULL,
    published_at timestamptz
  );
`;

type DigestInput = { customerId: string; weekStart: string };
type ClaimedJob = { id: string; request_id: string };

function canonicalHash(input: DigestInput): string {
  return createHash('sha256')
    .update(`${input.customerId}\n${input.weekStart}`)
    .digest('hex');
}

async function inTransaction<T>(fn: (client: PoolClient) => Promise<T>): Promise<T> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const value = await fn(client);
    await client.query('COMMIT');
    return value;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

app.post('/weekly-digests', async (req, res) => {
  const input = req.body as Partial<DigestInput>;
  const idempotencyKey = req.header('Idempotency-Key');
  if (!idempotencyKey || !input.customerId || !input.weekStart) {
    res.status(400).json({ error: 'missing required digest input' });
    return;
  }

  const digestInput: DigestInput = {
    customerId: input.customerId,
    weekStart: input.weekStart
  };
  const requestHash = canonicalHash(digestInput);

  const result = await inTransaction(async (client) => {
    const inserted = await client.query<{ id: string }>(`
      INSERT INTO digest_requests
        (customer_id, week_start, idempotency_key, request_hash)
      VALUES ($1, $2, $3, $4)
      ON CONFLICT DO NOTHING
      RETURNING id
    `, [digestInput.customerId, digestInput.weekStart, idempotencyKey, requestHash]);

    if (inserted.rowCount === 1) {
      const requestId = inserted.rows[0].id;
      await client.query(`
        INSERT INTO digest_jobs (request_id) VALUES ($1)
      `, [requestId]);
      return { requestId, replay: false };
    }

    const existing = await client.query<{ id: string; request_hash: string }>(`
      SELECT id, request_hash
      FROM digest_requests
      WHERE idempotency_key = $1
         OR (customer_id = $2 AND week_start = $3)
      FOR UPDATE
    `, [idempotencyKey, digestInput.customerId, digestInput.weekStart]);

    if (existing.rowCount !== 1 || existing.rows[0].request_hash !== requestHash) {
      return { conflict: true } as const;
    }
    return { requestId: existing.rows[0].id, replay: true };
  });

  if ('conflict' in result) {
    res.status(409).json({ error: 'idempotency key or digest period was reused' });
    return;
  }
  res.status(202).json(result);
});

async function claimJob(): Promise<ClaimedJob | undefined> {
  return inTransaction(async (client) => {
    const claimed = await client.query<ClaimedJob>(`
      WITH candidate AS (
        SELECT id
        FROM digest_jobs
        WHERE status = 'ready' AND available_at <= now()
        ORDER BY available_at, id
        FOR UPDATE SKIP LOCKED
        LIMIT 1
      )
      UPDATE digest_jobs AS job
      SET status = 'running',
          lease_until = now() + interval '5 minutes',
          attempts = attempts + 1
      FROM candidate
      WHERE job.id = candidate.id
      RETURNING job.id, job.request_id
    `);
    return claimed.rows[0];
  });
}

async function buildDigest(requestId: string): Promise<Record<string, unknown>> {
  // Fetch approved ledger aggregates and generate the customer-safe summary here.
  return { requestId, kind: 'weekly-digest' };
}

async function consumeOne(): Promise<boolean> {
  const job = await claimJob();
  if (!job) return false;

  const payload = await buildDigest(job.request_id);
  await inTransaction(async (client) => {
    await client.query(`
      INSERT INTO digest_publications (request_id, payload)
      VALUES ($1, $2)
      ON CONFLICT (request_id) DO NOTHING
    `, [job.request_id, payload]);
    await client.query(`
      UPDATE digest_requests SET status = 'prepared' WHERE id = $1;
    `, [job.request_id]);
    await client.query(`
      UPDATE digest_jobs
      SET status = 'complete', lease_until = NULL
      WHERE id = $1
    `, [job.id]);
  });
  return true;
}

async function releaseExpiredLeases(): Promise<void> {
  await pool.query(`
    UPDATE digest_jobs
    SET status = 'ready', lease_until = NULL, available_at = now()
    WHERE status = 'running' AND lease_until < now()
  `);
}

async function workerLoop(): Promise<void> {
  for (;;) {
    const worked = await consumeOne();
    if (!worked) await new Promise((resolve) => setTimeout(resolve, 500));
  }
}

void schema;
void releaseExpiredLeases;
void workerLoop;
app.listen(3000);
```

Two details are intentionally outside the compact loop. First, production code needs bounded retry delays and a terminal review state after a chosen attempt limit; retrying a permanently invalid customer record forever wastes compute. Second, the sender that drains `digest_publications` must pass `request_id` as its downstream idempotency key when the destination supports one. If the destination cannot deduplicate and cannot participate in the database transaction, exactly-once delivery cannot be guaranteed. Record the send attempt and reconcile ambiguous outcomes instead of claiming certainty. There is also a race worth naming. A five-minute lease is safe only if normal digest construction completes comfortably inside it, or if workers extend leases while they still own them. I'm not sure a universal lease duration exists, because data volume and model latency vary. Measure the job p99, set the initial lease above it with margin, and alert on expired leases; those observations resolve the choice for this workload.

## The duplicate starts before the queue

Transport identifiers are too narrow. An HTTP retry may carry a new connection ID, and a redelivered job may receive a new broker delivery tag, while the requested outcome remains one digest for customer `cust_4821` for week `2026-08-17`. The durable identity is that business tuple. The client-provided key is useful for replaying the same response, but the unique tuple is the backstop when a caller generates a fresh key after a timeout. This distinction prevents three different duplicates. The request table rejects two logical requests for the same period. The jobs table allows one queue row per request. The publication table allows one prepared output per request. None of those constraints proves that an external email or push endpoint acted once, so the final dispatcher still needs a destination idempotency contract or a reconciliation process. Short version: deduplicate at every irreversible boundary. Keep the original input hash. Without it, a reused key can silently return an older request even though the caller changed the week or customer. PostgreSQL reports unique-constraint violations with SQLSTATE `23505`, but treating that code alone as a replay is insufficient; fetch the existing row and compare the canonical hash before returning `202`. A mismatch is a conflict, not success. Model calls add another wrinkle. Retrying generation may produce different wording even with the same inputs, and token use grows with every attempt. Store the accepted prepared payload before publication. If a worker loses its lease after that commit, the next attempt should read the existing publication rather than invoke the model again. That one constraint protects both consistency and cost.

## Measure two latency clocks before adding infrastructure

Start with the database queue when weekly volume is moderate, PostgreSQL is already an operational dependency, and a few seconds of dispatch delay is acceptable. It gives the API a fast durability boundary with no extra network hop to a broker.

**Limitation:** this design is not suitable for queue traffic that competes with latency-sensitive ledger queries, backlogs that require an independent failure domain, or fan-out that depends on broker-native routing across many consumers. Use a dedicated message broker in those cases, while preserving the enqueue, lease, idempotency, and outbox contracts.

The cost test should include more than a service invoice. Count database write amplification, polling queries while idle, retained job rows, on-call work, and the engineering time needed to build fair scheduling. Polling every 500 milliseconds is easy to understand and intentionally visible in the example, but at many worker replicas it creates avoidable reads. Adaptive backoff or database notifications can reduce idle traffic; a managed broker can become the cheaper system once operational labor and contention dominate. Your mileage may vary, so decide from queue depth, oldest-job age, claim latency, completion latency, attempts per job, expired leases, and tokens per completed digest.

Priority deserves restraint. A high-priority queue for compliance notices and a normal queue for weekly digests may be justified, but adding many priority levels makes starvation and capacity planning harder. RabbitMQ's documentation explicitly notes that priority queues have CPU and memory costs and recommends keeping the number of priorities small. A pair of separate lanes is often easier to reason about than an elaborate numeric scale.

Watch both.

Latency has two clocks. The API clock ends after the enqueue transaction, while the customer clock ends after publication. Track both. A fast `202` can hide a six-hour backlog, and a cheap worker pool can miss the promised weekly window. For this fintech job, scale workers from oldest-job age before raw queue depth: one unusually expensive digest can matter more than hundreds of cheap ones. No single threshold is universal; the delivery promise and measured service times should set it.

## How can an API request queue keep a consumer worker safe through restarts?

Deployment starts with compatibility. Add tables and nullable columns before code depends on them, deploy consumers that understand both old and new payload versions, then switch publishers. During shutdown, stop claiming work, allow active jobs to finish within a deadline, and leave unfinished leases to expire. Never hold a database transaction open across ledger reads, model inference, or network publication — locks turn variable external latency into database contention.

Stop cleanly.

Testing should force the awkward boundaries. Send the same idempotency key twice and expect one request ID. Send different inputs with that key and expect `409`. Stop a worker after claim, expire the lease, and verify another worker can finish. Stop it after publication insertion but before job completion, then verify the unique publication row prevents regeneration. Run two claimers concurrently and assert they receive different job IDs. These are deterministic integration tests against PostgreSQL, not timing guesses. Observability should join every event with `request_id`, `job_id`, attempt number, and payload version while excluding account balances and generated digest text from logs. Alert on age, not merely errors: a queue can be perfectly quiet because consumers stopped claiming. Retain counts for conflicts, retries, lease expiry, preparation duration, publish duration, and token consumption. The weekly schedule also needs a completeness query that compares eligible active customers with durable request rows for the period; enqueue metrics alone cannot reveal customers the scheduler never selected.

Before shipping, walk one digest from scheduler selection to the durable request, leased job, prepared publication, and downstream receipt. Confirm that every transition is queryable, every retry preserves the business identity, and every slow operation occurs outside an open transaction. Then rehearse a worker restart and a delayed downstream acknowledgment. The system is ready when those ordinary failures lead to bounded repeat work and an auditable final state, not when the happy-path demo returns quickly.

## References

- https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- https://www.postgresql.org/docs/current/errcodes-appendix.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://www.rabbitmq.com/docs/priority
- https://www.rfc-editor.org/rfc/rfc9110.html#name-202-accepted
