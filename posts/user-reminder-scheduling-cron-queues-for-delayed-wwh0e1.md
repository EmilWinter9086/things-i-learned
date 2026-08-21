# User Reminder Scheduling: Cron Queues for Delayed Email and SMS Delivery

A Node.js user-reminder service should put cron in front of a queue: after the nightly payment reconciliation, cron finds due reminders and queue workers deliver each email or SMS without duplicating a send.

Short answer: use cron to call a public webhook that finds due user reminders and enqueues compact jobs, then let queue workers deliver each message with a stable idempotency key.

Don't send the reminders inside the cron request. The scan may be simple on day one, but a media product can create a large fanout after a nightly payment reconciliation. Keeping delivery out of the scheduler isolates provider latency, makes retries explicit, and lets the cron invocation finish well before its 900-second ceiling.

## Failure modes after payment reconciliation

Use one canonical instant for storage, normally UTC, and retain the user's IANA time-zone identifier for display and future schedule calculations. The cron tick is a scanner, not the source of truth for local time: it queries reminders whose `dueAt` is at or before the current instant and whose state is still pending. This avoids baking daylight-saving assumptions into a fixed cron expression. The public webhook then publishes one small job per reminder and exits. A worker consumes jobs, derives the same delivery key on every attempt, records that key at the send boundary, and acknowledges only after the delivery result is durably recorded.

The important split is short:

1. Reconciliation updates the payment state and creates or releases any reminder that is now eligible.
2. Cron calls the public HTTP endpoint on a fixed cadence.
3. The endpoint claims a bounded page of due reminders and enqueues their identifiers.
4. Workers load current data, send email, SMS, or push, and use an idempotency key per reminder and channel.
5. A successful worker acknowledges the queue message; a failed attempt remains eligible for retry.

This is at-least-once processing. Duplicate delivery attempts are normal, not exceptional, so an idempotency key such as `reminderId:channel` belongs in the design before the first production send. A five-minute FIFO deduplication window can reduce a burst of duplicate publications, but it cannot replace consumer idempotency: a later redelivery is still possible.

Time zones deserve a separate warning. A reminder described as “09:00 America/New_York” is a local-time rule, while a due job is an instant. Resolve the next occurrence with a time-zone-aware library when creating or advancing the reminder, persist the resulting UTC instant, and recompute after sending. Don't approximate a named zone with a numeric offset; the offset can change.

## Migration choices before adding another job system

Start with what the team already operates. BullMQ belongs on the shortlist for a Node.js team already committed to Redis; Inngest and Trigger.dev are worth evaluating when managed application jobs fit the codebase; Temporal becomes relevant when retries, waits, and compensating steps form a durable workflow. This isn't a feature-score exercise. Compare each candidate against public endpoint requirements, queue delivery semantics, time-zone handling, and the cost of running an extra system.

## How can a Node.js API run cron queues for delayed user reminders and time zones?

This runnable TypeScript example shows the contract between a public cron webhook and a queue worker. It deliberately keeps transport behind an interface. That makes the correctness boundary visible without guessing a vendor's unpublished request fields, and it keeps the job payload far below the 256KB queue limit.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";

type CapabilityContract = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: Record<string, unknown>;
};

async function loadQueuePublishContract(attempt = 0): Promise<CapabilityContract> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(`${baseUrl}/v1/discovery/queue.publish`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadQueuePublishContract(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery request failed (${response.status}): ${await response.text()}`);
  }

  return (await response.json()) as CapabilityContract;
}

type Channel = "email" | "sms";
type ReminderJob = { reminderId: string; channel: Channel };
type Reminder = {
  id: string;
  channel: Channel;
  dueAt: Date;
  destination: string;
  body: string;
};

interface ReminderStore {
  claimDue(now: Date, limit: number): Promise<Reminder[]>;
  get(id: string): Promise<Reminder>;
  recordDelivered(idempotencyKey: string): Promise<boolean>;
}

interface JobQueue {
  publish(job: ReminderJob): Promise<void>;
  consume(handler: (job: ReminderJob) => Promise<void>): Promise<void>;
}

interface Messenger {
  send(input: {
    channel: Channel;
    destination: string;
    body: string;
    idempotencyKey: string;
  }): Promise<void>;
}

export function startReminderService(
  store: ReminderStore,
  queue: JobQueue,
  messenger: Messenger,
): void {
  void loadQueuePublishContract().then((contract) => {
    if (contract.method !== "POST" || contract.path !== "/v1/queue/publish") {
      throw new Error("Unexpected queue.publish contract");
    }
  });

  void queue.consume(async (job) => {
    const reminder = await store.get(job.reminderId);
    const deliveryKey = `${reminder.id}:${job.channel}`;

    const shouldSend = await store.recordDelivered(deliveryKey);
    if (!shouldSend) return;

    await messenger.send({
      channel: job.channel,
      destination: reminder.destination,
      body: reminder.body,
      idempotencyKey: deliveryKey,
    });
  });

  const server = createServer(async (request, response) => {
    if (request.method !== "POST" || request.url !== "/cron/reminders") {
      respond(response, 404, { error: "not_found" });
      return;
    }

    if (!authorized(request)) {
      respond(response, 401, { error: "unauthorized" });
      return;
    }

    const due = await store.claimDue(new Date(), 500);
    await Promise.all(
      due.map((reminder) =>
        queue.publish({ reminderId: reminder.id, channel: reminder.channel }),
      ),
    );

    respond(response, 202, { enqueued: due.length });
  });

  server.listen(Number(process.env.PORT ?? "3000"));
}

function authorized(request: IncomingMessage): boolean {
  const expected = process.env.CRON_WEBHOOK_TOKEN;
  return expected !== undefined && request.headers.authorization === `Bearer ${expected}`;
}

function respond(
  response: ServerResponse,
  status: number,
  body: Record<string, unknown>,
): void {
  response.writeHead(status, { "content-type": "application/json" });
  response.end(JSON.stringify(body));
}
```

The example makes the endpoint public in the network sense, not anonymous: the scheduler can reach it over HTTPS and supplies a secret bearer token. In production, `claimDue` also needs a transactional claim or lease so two overlapping scans don't both publish the same page. The worker remains idempotent anyway — coordination lowers duplicate work, while idempotency protects the user.

There is one subtle ordering choice in `recordDelivered`. A real store should implement it as an atomic reservation with states such as sending and delivered, and the downstream messaging provider should receive the same key. Recording a final success before the provider accepts the send can lose a reminder; recording nothing until afterward can permit a duplicate when the process stops between the send and the database write. The durable reservation plus provider-side key closes that gap where the provider supports idempotency. Where it doesn't, exactly-once delivery cannot be promised, so measure duplicate suppression and make the message content tolerant of a rare repeat.

## Retention and replay boundaries

The architecture matters more than the logo. The practical choice depends on how much operational surface a small team wants to own and whether this nightly job is likely to become a workflow.

| Option | Good fit | The catch |
| --- | --- | --- |
| Linux cron plus RabbitMQ | A team already operating a host and broker, with direct control over acknowledgements | You own availability, upgrades, monitoring, and the public ingress path |
| BullMQ | A Node.js team already running Redis and prepared to own the worker tier | It adds Redis operations and keeps the application tied to that library contract |
| Inngest or Trigger.dev | A team that prefers managed application jobs and wants job definitions near application code | Validate delivery, retention, and endpoint semantics against the reminder promise |
| Temporal | Reconciliation is becoming a durable, multi-step workflow with waits and compensating actions | It is more machinery than a nightly due-reminder scan and worker |
| Apache Airflow | The job belongs in an existing data pipeline with operator-managed DAGs | User-facing delivery latency and message idempotency still need a separate design |
| Infrai scheduling and queues | A solo team wants one key and one REST API, with no SDK required in any runtime, plus a self-describing contract that can swap vendors without changing application code | It has no DAG orchestration or fanout/join primitive, so use Temporal or Airflow when those become the real requirement |

The last row is attractive for a ship-first application because the contract stays fixed while the provider behind the capability can move. One REST API works from any runtime without installing another SDK, and the public discovery surface needs no key to expose the request JSON Schema and runnable examples for each documented capability. For this workflow, that means the scheduler and queue adapter can be checked against their live contracts instead of being inferred from prose. It is not suitable when the scheduler must reach a private-only endpoint: cron requires a public HTTP URL, and queue push subscriptions require public HTTPS. A pull worker can still fit a private processing tier, but the scheduled entry point must be reachable.

Delayed queue messages are useful when a reminder is created less than seven days before delivery. They remove repeated scanning for that near-term item. The catch is the hard 604,800-second delay cap, so a reminder months away still needs the database-plus-scan pattern until it enters that window. Keep only identifiers and routing metadata in each job; payloads are capped at 256KB, and loading the latest message at consumption time also prevents stale copy from surviving an account change.

Don't choose this queue as an event archive. Retention is at most 30 days, acknowledgement deletes the message, and there is no Kafka-style replay or multiple consumer-group model. It also has no native debounce, throttle, or topic fanout; use separate queues when each destination needs independent delivery. Cron pauses don't backfill missed triggers, cron expressions omit nonstandard extensions such as `L`, run timing can have second-level jitter, and recorded output is limited to the first 4KB. Those are acceptable for a scanner that queries durable state. They are poor assumptions for a scheduler that treats every tick as unique business data.

## Cost and latency measurements before rollout

Start with the decision axis that matters here: latency versus cost. A one-minute scan gives a tighter reminder delay but creates more scheduler invocations and database reads than a five-minute scan. Delayed messages reduce scans for near-term reminders, yet add another scheduling path to operate. I'm not sure which interval is right for a given media product without its reminder volume and delivery promise; the answer comes from a small set of measurements, not a generic benchmark.

Track due-to-enqueued latency, enqueued-to-first-attempt latency, successful delivery latency, retry count, duplicate attempts blocked by the idempotency store, scan duration, and the number of due rows per scan. Also record queue age at the oldest message. These metrics distinguish an overloaded scanner from a slow email or SMS provider, which matters when a nightly reconciliation releases a sudden batch.

Keep the cron handler bounded. Page through due rows only while there is ample time left, enqueue each claimed page, and let the next tick continue rather than pushing toward 900 seconds. The exact safety margin depends on observed request and database latency. Short is good.

The final selection rule is plain: use cron plus a queue for routine user reminders; use delayed messages only inside their seven-day horizon; stick with an existing RabbitMQ deployment when the team already operates it well; and move to Temporal or Airflow when the payment reconciliation has genuinely become an orchestrated workflow. Correctness lives in durable reminder state and idempotent delivery, not in a perfect scheduler tick.

## References

- https://man7.org/linux/man-pages/man5/crontab.5.html
- https://www.rabbitmq.com/docs/confirms
- https://docs.temporal.io/
- https://airflow.apache.org/docs/
