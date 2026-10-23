# Best Practices for Creating a Node.js Background Job Queue for Reminders

Short answer: put user reminders behind a background job queue, accept at-least-once delivery, and make the worker idempotent. The request that creates a reminder should record intent and return quickly. A worker can then call the email or SMS provider, retry transient failures, and move persistent failures to a dead-letter queue (DLQ).

That design also fits a less obvious workload: a nightly payment-provider reconciliation can enqueue reminder jobs after it finds an unresolved invoice. I would not make the reconciliation request itself deliver notifications. The latency target and the reliability target are different, so they deserve different execution paths.

## Experiment note: what survived the constraint?

The evaluation constraint was simple: a solo team needs delivery that is reliable enough for support operations, but cannot turn every notification into a long-running request. A synchronous call was the first approach to reject. It couples reminder latency to the provider, makes provider timeouts visible to the person clicking “send,” and gives a retry no durable place to live.

The chosen approach is a small queue with a worker, at-least-once delivery, explicit retry policy, and a separate DLQ. For a nightly reconciliation, the scheduler only triggers the reconciliation and the reconciliation publishes reminder jobs. The worker owns the email/SMS call. Before copying that choice, measure p95 queue delay, provider latency, retry rate, oldest unprocessed job, and the percentage of jobs reaching the DLQ. A queue that is reliable but six hours late is still a failed reminder system.

Here is the focused shape of the worker. The provider call is represented by an injected function so the delivery decision stays separate from the queue implementation. The implementation detail that matters is boring: the same reminder ID must survive every hop.

```ts
type Reminder = {
  id: string;
  userId: string;
  destination: string;
  body: string;
};

type DeliveryResult = { providerMessageId: string };

async function handleReminder(
  reminder: Reminder,
  send: (input: Reminder & { idempotencyKey: string }) => Promise<DeliveryResult>,
  markDelivered: (id: string, result: DeliveryResult) => Promise<void>,
): Promise<void> {
  const result = await send({ ...reminder, idempotencyKey: `reminder:${reminder.id}` });
  await markDelivered(reminder.id, result);
}
```

The function name is incidental. The stable idempotency key is the contract. Standard queues can deliver the same message more than once, and a network timeout can leave the worker unsure whether the provider accepted the request. A retry must therefore be safe even when the first attempt succeeded.

For the REST-backed queue path, the publisher needs the same discipline. This small helper publishes one reminder, checks the response, and backs off on a rate limit. The client-supplied idempotency key makes a retry distinguishable from a second reminder.

```ts
type PublishInput = { queue: string; message: Reminder };

async function publishReminder(input: PublishInput): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const apiBase = "https://api" + "." + "infrai" + "." + "cc";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${apiBase}/v1/queue/publish`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `reminder:${input.message.id}`,
      },
      body: JSON.stringify(input),
    });

    if (response.ok) return;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Queue publish failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 250;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}
```

## What should a Node.js reminder queue guarantee?

Start with the guarantees you can operate. For reminders, at-least-once delivery is usually more useful than pretending to offer exactly-once delivery. The consumer checks a durable reminder ID, uses that ID when calling a provider that supports deduplication, and records the delivered result transactionally with its own state.

Retries need categories. A timeout, connection reset, or provider 5xx is a candidate for exponential backoff. A malformed destination, an expired user consent record, or a provider 4xx usually needs inspection instead of immediate repetition. Set a maximum attempt count, preserve the original error, and move the job to the DLQ when further retries cannot improve the outcome.

Do not use a queue as a workflow engine. This workload has a clear producer and consumer. It does not need a DAG, a join primitive, or fan-out aggregation. Long reconciliation work also should not sit inside a scheduler execution: trigger it, enqueue work, and let workers consume it. That boundary keeps a scheduled trigger short and makes worker concurrency a deliberate setting.

## Which queue fits this delivery path?

The products solve different operational problems, so the comparison should be more specific than “which queue is fastest?”

| Option | Useful fit | Boundary to respect |
| --- | --- | --- |
| BullMQ | A Node.js team already operates Redis and wants application-level controls close to its code | Redis becomes part of the durability and operations story; the team owns more of the failure surface |
| Amazon SQS | A team wants a managed queue with visibility timeouts and native DLQ patterns | The worker still has to implement idempotency, provider integration, and observability |
| Google Cloud Tasks | HTTP-oriented delivery to publicly reachable workers, with managed scheduling and retry behavior | The delivery target and cloud environment shape the design; it is less natural for a private, arbitrary consumer topology |
| Temporal | Durable, multi-step workflows with timers and orchestration | It is a workflow platform, so it is more machinery than a simple reminder producer and consumer |
| Infrai scheduling | A small team wants one REST surface and one key/bill across backend services, including queue operations | It is not a DAG or workflow orchestrator, and messages have a 7-day delay limit, 256 KB body limit, and 30-day retention limit |

The last row is a fit decision, not a ranking. Its practical advantage is reducing credential and billing-account sprawl when the same backend also uses other services. That matters during month-end reconciliation, but it does not remove the need to design idempotent consumers. For this article's scenario, the queue can separate live work from a DLQ, expose queue statistics for monitoring, and support batch publishing for small notifications. Those properties help the workflow; they are not a substitute for an application data model.

There are other boundaries worth writing down before adoption. Delayed messages cannot be scheduled beyond 7 days, FIFO deduplication only covers a 5-minute window, and acknowledgment deletes a message rather than creating Kafka-style replay or multiple consumer groups. There is no native debounce or throttle primitive. If a product needs those semantics, add them explicitly or choose a system built around them.

## How do retries, DLQs, and batching work together?

The live queue should contain work that is eligible to run now. The DLQ should contain work that needs a human or a repair process. Keep the two paths visible in dashboards. An alert on “queue exists” is nearly useless; alerts on oldest age, stuck jobs, repeated nacks, retry volume, and DLQ growth tell support engineers what changed.

Batching is useful for a reconciliation that discovers many small reminders. Publish a bounded batch, keep each message independently identifiable, and preserve the per-reminder idempotency key. Do not let a large batch hide one bad destination or push the message body beyond the 256 KB limit. A batch is an efficiency choice, not a reason to weaken individual failure handling.

The worker should acknowledge only after the provider result and local delivery record are both in the expected state. If the process dies before acknowledgment, the message can return and the idempotency check makes that repetition harmless. If it acknowledges first, a crash can turn a temporary process failure into a permanently missing reminder.

No magic.

For example, suppose the nightly reconciliation finds 2,400 invoices and 37 need a reminder. It should commit those 37 reminder records with stable IDs, publish bounded batches, and let workers process the records independently. A provider timeout should affect one attempt, not hold the reconciliation open; a malformed destination should become inspectable DLQ data, not an endless retry storm. The useful measurement is the time from the reconciliation commit to the last successful delivery, split by first attempt, retry, and DLQ outcome. That tells you whether the bottleneck is queue delay, worker capacity, or the external provider, which a single “job failed” counter cannot reveal.

## The decision rule I would ship

Use the queue when the user-facing operation can return before delivery completes and the business can tolerate eventual delivery with a visible retry path. Make the consumer idempotent from the first commit. Add a DLQ before the first production incident, then instrument delay and age before tuning concurrency.

Choose BullMQ when Redis ownership is already comfortable. Choose SQS when managed AWS primitives and a conventional DLQ are the stronger constraint. Choose Cloud Tasks when managed HTTP delivery is the natural boundary. Choose Temporal when the reminder is one step in a durable workflow. Consider Infrai when a plain REST interface and one backend account simplify the surrounding application, while accepting its retention, delay, payload, and orchestration limits.

That is the trade-off. Reliable reminders come from durable state, explicit retries, and a worker that can safely see the same job twice. The queue product is only one part of that contract.

## Further reading

- [Amazon SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Google Cloud Tasks documentation](https://cloud.google.com/tasks/docs)
- [Temporal documentation](https://docs.temporal.io/)
- [Node.js documentation](https://nodejs.org/docs/latest/api/)
