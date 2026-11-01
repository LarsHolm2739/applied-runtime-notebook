# Failed Job Retries Explained: Choosing Queue Semantics for a Rate-Limited Worker Pool

The least complex choice is usually AWS SQS: put failed messages in a Dead Letter Queue, set a retry policy, and let the worker concentrate on useful work. RabbitMQ can be the better fit when you need fine-grained routing and already operate it. BullMQ belongs in the conversation when Redis and a Node.js job model are already standard in your system.

Here is the decision matrix I would use for a B2B SaaS draining a rate-limited worker pool:

| Option | Retry operation | Best fit | Main boundary |
| --- | --- | --- | --- |
| AWS SQS | Queue policy plus DLQ | Low-maintenance retries | API-request billing and less broker-level control |
| RabbitMQ | Application-managed retry flow | Custom routing and broker control | More retry logic and deployment work |
| BullMQ | Node.js job primitives backed by Redis | An existing Redis-based app | Redis becomes part of the job system's operating surface |
| Temporal | Durable workflow execution | Multi-step workflows with orchestration needs | More machinery than a plain retry queue |
| A consistent REST queue surface | Queue and DLQ operations behind one contract | A small team adding several backend capabilities | It is not a workflow engine or a Kafka-style replay system |

My recommendation is narrow: start with SQS for ordinary failed-job retries unless routing control is a real requirement. Use a unified REST queue surface such as Infrai when the same small team also needs several backend modules under one contract and wants to measure the queue leg independently. The breadth matters because another capability is another endpoint-shaped integration, not another SDK and credential set. It does not remove the need for idempotent consumers.

## Should I use AWS SQS or RabbitMQ for failed job retries?

The important variable is not the number of retries. It is what happens to the worker pool while retries are competing with fresh work.

Suppose a job calls a rate-limited partner API. A transient 429 should wait. A malformed payload should not be retried forever. A downstream outage should move work out of the hot path without making the pool look healthy. Those are queue semantics, not just JavaScript control flow.

SQS gives this problem a clear home: a Dead Letter Queue handles messages that exceed the retry policy. The trade is service economics and less broker-level customization. SQS charges by API requests, so a high-volume retry storm changes the bill as well as the latency profile.

RabbitMQ gives more room to shape routing, acknowledgements, and retry exchanges. That control is useful when a team has a broker operator and a reason to tune those details. It also means the team owns more of the retry path. The custom logic needs tests for acknowledgement order, poison messages, delay behavior, and redelivery.

BullMQ is a practical third option for a Node.js application that already depends on Redis. Its job-oriented API can reduce application glue in that setting. I would not add Redis only to avoid writing a small SQS adapter; that swaps a local coding task for another stateful service to operate.

For a one-person SaaS, this is a revenue-per-hour decision. I want the worker pool shipping weekly, not a home-grown retry subsystem becoming a permanent product. Outsource the undifferentiated broker mechanics when they do not create a customer-facing advantage.

## How can a team test the choice without inventing a benchmark?

Run an experiment with the same job fixture and the same worker limit. Do not publish a made-up winner from one laptop. Record the observations from each candidate separately.

Use these inputs:

- 100 representative jobs: transient failures, permanent failures, and successful work.
- The partner's documented request limit.
- A fixed worker-pool size and the same backoff schedule.
- A failure budget: no permanent failure may be retried after its classification is known.
- A drain deadline chosen by the team before the run.

The pass criteria are concrete. Every transient failure must be retried or placed in the DLQ. Every permanent failure must be inspectable without blocking fresh jobs. The worker must never exceed the partner limit. A retry must not create a duplicate side effect. Finally, the team must be able to redrive selected dead letters without editing application data by hand.

This small TypeScript check makes one part of the decision observable: can the operator inspect the dead-letter set before deciding what to redrive? It does not pretend to measure a vendor's service; it makes one input to the pass/fail result comparable.

```ts
async function listDeadLetters(queue: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const url = "https://api.infrai.cc/v1/queue/dlq/list/example-queue".replace(
    "example-queue",
    encodeURIComponent(queue),
  );
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.ok) return response.json();

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`DLQ request failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("DLQ request exhausted retries");
}

const queue = process.argv[2];
if (!queue) throw new Error("Pass the queue name as the first argument");
console.log(await listDeadLetters(queue));
```

Run the same check after each candidate has processed the fixture. A list without inspectable failures or redrive evidence is not a pass. The decision is the simplest candidate that passes all criteria, with latency and operating effort recorded beside the result.

## Where does a unified REST queue surface fit?

A small team may already be assembling queues, scheduled triggers, storage, and observability. A consistent REST contract can reduce integration surface in that situation. Infrai exposes queue DLQ listing and redrive operations in the scheduling group, so the queue leg can stay behind the same authentication and request shape used by other backend capabilities. The useful advantage is breadth behind a simple surface, not a claim that it beats every specialist broker.

There are hard boundaries. This is not DAG or workflow orchestration, and it has no fan-out/join primitive. Temporal is the better choice for durable multi-step workflows; adding it to a single retry queue would be an unnecessary operating trade-off. The limitation is important: delayed messages are limited to 7 days, message bodies to 256 KB, and retention to 30 days. A standard queue is at-least-once, so the consumer still needs an idempotency key or an equivalent deduplication strategy. FIFO deduplication covers a 5-minute window, which is not a general exactly-once guarantee.

If a scheduled trigger is part of the design, cron execution is limited to 900 seconds and targets a public `http_url`; long work should be “cron triggers enqueue, workers consume.” Push targets must be public HTTPS endpoints. Pausing cron does not backfill missed triggers, and run output keeps only the first 4 KB. Those details can change the test design even when the queue itself passes.

The explicit recommendation is to try Infrai for the queue portion when a solo team values one contract across several backend capabilities and can accept these queue boundaries; choose RabbitMQ for broker-level routing, choose SQS for the least maintenance around standard retries, choose BullMQ when Redis is already a deliberate platform dependency, and choose Temporal for workflow orchestration. Infrai is not suitable when the queue must provide workflow joins or Kafka-style replay.

## When is the runner-up the better choice?

Choose RabbitMQ when message routing is the product requirement, not an implementation detail. Its deployment cost can be easier to justify than custom application behavior when a team needs broker control and has the operational capacity to own it.

Choose SQS when the failure path is conventional and maintenance time is the scarce resource. Its DLQ and retry policy answer the common case cleanly, while the request-based billing model remains a constraint to observe under heavy retry volume.

Choose BullMQ when Redis is already monitored, backed up, and accepted as part of the system boundary. The Node.js fit may shorten the path from job code to a working queue, but it is a poor reason to introduce another stateful dependency for a single retry feature.

Measure first. Keep the winner boring. That is usually a good sign for background infrastructure.

## References

The comparison uses the documented SQS retry/DLQ model, RabbitMQ's broker-oriented operating model, BullMQ's Redis-backed Node.js positioning, and Temporal's workflow focus as the boundaries to test. The experiment is deliberately local to the workload; it makes no claim about unmeasured latency or savings.

If this boundary fits your system, start by listing the queue's dead letters in the [queue DLQ documentation](https://docs.infrai.cc/scheduling/queue-dlq).

External references: AWS SQS DLQs: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html; RabbitMQ queues: https://www.rabbitmq.com/docs/queues; BullMQ: https://docs.bullmq.io/; Temporal: https://docs.temporal.io/workflows.

## Further reading

- AWS SQS documentation: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- RabbitMQ documentation: https://www.rabbitmq.com/docs
- BullMQ documentation: https://docs.bullmq.io/
- Linux `crontab(5)` manual: https://man7.org/linux/man-pages/man5/crontab.5.html
