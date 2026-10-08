# Cron vs Message Queues: Delayed Webhook Retries with Idempotent Processing

Short answer: choose a message queue for delayed webhook delivery and retries; use cron for fixed recurring schedules, not for a different deadline attached to every event.

| Choice | Best fit | Main trade-off |
| --- | --- | --- |
| Cron | One fixed recurring schedule | Simple trigger, but a poor model for per-event delay and retry state |
| Managed message queue | Many independent deadlines and retry attempts | Less application machinery, with explicit delivery limits and at-least-once handling |
| RabbitMQ or BullMQ | Teams that want broker control or already run a Node.js job stack | More infrastructure ownership |
| Trigger.dev | Teams that want an application-oriented task platform | A broader abstraction than a plain queue |
| Temporal or Airflow | Multi-step workflow orchestration | More machinery than a single delayed webhook needs |

For a one-person logistics SaaS, I would put each renewal reminder on a queue with its own due time, then acknowledge it only after the destination accepts the webhook. That choice optimizes the scarce resource: engineering attention. The dispatch might be a few seconds late, but retry state isn't hidden in a database scan, and shipping the next revenue-producing feature stays plausible this week.

## Should a SaaS use cron or a message queue for delayed webhook delivery retries?

A renewal reminder is attached to a business event: shipment account `acct_4821` renews at a particular deadline, perhaps three days after an invoice enters review. Another account has another deadline. A cron expression describes a recurring calendar schedule; it doesn't naturally describe thousands of unrelated future timestamps. Making cron pretend otherwise usually means running a frequent sweep over a table, claiming due rows, recording attempts, and preventing two overlapping sweeps from sending the same webhook.

That sweep can be valid at small scale. It is also a queue implemented inside the application database. The code must own locking, recovery, backoff, observability, and cleanup, so the apparent simplicity disappears right where failures begin. A managed queue already maps retry attempts to delivery state: a consumer acknowledges success and negatively acknowledges work that should be retried. Standard queues still provide at-least-once delivery, so idempotency remains application work. There is no honest checkbox that removes it.

Cron has two more mismatches here. A run can last at most 900 seconds and invokes a public HTTP URL, which makes it a trigger rather than a long-running worker. Paused schedules don't backfill missed runs, and execution has seconds-level jitter. Those properties are tolerable for a nightly reconciliation job; they are awkward for a reminder whose deadline belongs to one customer record.

Wrong abstraction.

Keep cron for genuinely periodic work. A daily report, a weekly cleanup, or a regular scan that publishes bounded jobs to a queue is easy to reason about. For anything longer than 900 seconds, let cron enqueue work and let workers consume it.

## Latency and cost are coupled

The word "cheapest" is slippery for a solo SaaS. Direct infrastructure spend matters, but so do the hours spent maintaining a polling loop. I use revenue per hour as the deciding lens: if a generic service removes retry bookkeeping and lets me ship weekly, a slightly higher request bill can still be the cheaper system. Your mileage may vary when traffic is stable and an existing database sweep already has good locking and metrics.

Latency makes the trade concrete. A once-per-minute cron sweep adds up to roughly one sweep interval before dispatch even starts; shortening the interval increases empty work and contention. A delayed message can become eligible near its own due time without scanning unrelated records. Neither mechanism promises zero jitter, so the product should express a deadline tolerance instead of implying an exact instant. For renewal reminders, seconds are usually acceptable. For a tightly synchronized auction close, they may not be.

Infrai is one managed queue option whose **one key covers 295 routes across 20 modules through a plain REST API**, reducing credential and SDK maintenance for a solo operator. The application contract can also stay put when the provider behind the capability changes. That can reduce integration churn for a small team, but it should not override the queue's actual constraints.

Those constraints are specific. Delayed messages top out at seven days, message bodies at 256 KB, and retention at 30 days. An acknowledged message is deleted. FIFO deduplication covers only a five-minute window, while standard queues are at least once. There is no native debounce or throttle, and no topic that fans one event out to several independent consumers. If billing, customer success, and audit processing each need the same event, use three queues or choose a system with the fan-out model you need.

That last point matters. A logistics renewal payload should carry a small reference and an immutable delivery ID, not a large account snapshot. If the business deadline is more than seven days away, store the deadline durably and use a bounded staging step closer to delivery; don't submit an out-of-range delay and hope. This is the sort of boundary that belongs in the design before the first customer depends on it.

## Make the consumer idempotent before tuning retries

At-least-once delivery means the same reminder can reach a consumer more than once. Consider a reminder due at `2026-09-10T09:00:00Z`. The first worker claims `renewal_acct_4821_2026-09-10`, sends the request, and loses its connection before it can acknowledge the queue message. The destination may already have accepted the reminder. When a second delivery arrives, a newly generated ID would make the retry look like fresh work and the customer could receive a duplicate. A stable ID changes the outcome: the consumer can recognize completed work locally, while the destination sees the same idempotency key on every attempt. The consumer should claim that ID in durable storage, send the webhook, and mark completion only after a successful response. A `429` should respect `Retry-After`; a permanent `4xx` needs an explicit product policy rather than endless retries. None of this depends on a lucky timing window.

Duplicates happen.

The compact TypeScript example below isolates that rule. `DeliveryStore` must be backed by a durable database with an atomic claim in production. The included memory store makes the file runnable and exposes the control flow without pretending that process memory is durable.

```ts
type DeliveryTask = {
  deliveryId: string;
  accountId: string;
  deadline: string;
  targetUrl: string;
};

interface DeliveryStore {
  claim(deliveryId: string): Promise<"claimed" | "done" | "busy">;
  complete(deliveryId: string): Promise<void>;
  release(deliveryId: string): Promise<void>;
}

class MemoryDeliveryStore implements DeliveryStore {
  private states = new Map<string, "busy" | "done">();

  async claim(deliveryId: string): Promise<"claimed" | "done" | "busy"> {
    const current = this.states.get(deliveryId);
    if (current) return current;
    this.states.set(deliveryId, "busy");
    return "claimed";
  }

  async complete(deliveryId: string): Promise<void> {
    this.states.set(deliveryId, "done");
  }

  async release(deliveryId: string): Promise<void> {
    if (this.states.get(deliveryId) === "busy") this.states.delete(deliveryId);
  }
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelayMs(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

async function publishReminder(
  deliveryId: string,
  publishBody: unknown,
): Promise<unknown> {
  const apiBaseUrl = process.env.INFRAI_API_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiBaseUrl || !apiKey) {
    throw new Error("INFRAI_API_BASE_URL and INFRAI_API_KEY are required");
  }

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBaseUrl}/v1/queue/publish`, {
      method: "POST",
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
        "idempotency-key": deliveryId,
      },
      body: JSON.stringify(publishBody),
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`queue publish failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("queue publish retry limit reached");
}

async function deliver(task: DeliveryTask, store: DeliveryStore): Promise<void> {
  const claim = await store.claim(task.deliveryId);
  if (claim === "done") return;
  if (claim === "busy") throw new Error("delivery already in progress");

  try {
    const response = await fetch(task.targetUrl, {
      method: "POST",
      headers: {
        "content-type": "application/json",
        "idempotency-key": task.deliveryId,
      },
      body: JSON.stringify({
        type: "renewal.reminder",
        deliveryId: task.deliveryId,
        accountId: task.accountId,
        deadline: task.deadline,
      }),
    });

    if (!response.ok) {
      const retryAfter = response.headers.get("retry-after");
      throw new Error(
        `webhook rejected with ${response.status}; retry-after=${retryAfter ?? "unset"}`,
      );
    }

    await store.complete(task.deliveryId);
  } catch (error) {
    await store.release(task.deliveryId);
    throw error;
  }
}

const deliveryTaskJson = process.env.DELIVERY_TASK_JSON;
const queuePublishBodyJson = process.env.INFRAI_QUEUE_PUBLISH_BODY;
if (!deliveryTaskJson || !queuePublishBodyJson) {
  throw new Error("DELIVERY_TASK_JSON and INFRAI_QUEUE_PUBLISH_BODY are required");
}

const task: DeliveryTask = JSON.parse(deliveryTaskJson);
const publishBody: unknown = JSON.parse(queuePublishBodyJson);
await publishReminder(task.deliveryId, publishBody);

// Run this line in the queue consumer, then ack only after it resolves.
await deliver(task, new MemoryDeliveryStore());
```

Build `INFRAI_QUEUE_PUBLISH_BODY` against the public `queue.publish` discovery schema rather than copying a stale field list into application code. The queue adapter then has a narrow job: decode the message, call `deliver`, acknowledge after it resolves, and negatively acknowledge when it throws. Backoff policy stays in queue configuration. HMAC signing should also wrap the outbound body so the recipient can authenticate it; RFC 2104 defines the keyed-hash construction, while key rotation and timestamp replay checks remain application policy.

Don't generate a new delivery ID during a retry. That single mistake turns an idempotent handler into duplicate reminders even when every other part is correct.

## When is the runner-up actually better?

Stick with cron when the schedule itself is the source of truth: "run reconciliation every night" is a cron problem. It is also reasonable when a low-volume database sweep already exists, the added dispatch latency is acceptable, and the team can prove atomic claiming and recovery. Cron remains a bad substitute for an unbounded worker, so publish work to a queue when the job could exceed its 900-second execution limit.

RabbitMQ is the better fit when priority queues and direct broker control matter enough to justify operating or procuring that broker. BullMQ deserves the shortlist when the Node.js application already uses its job model and accepts that operational footprint; Trigger.dev is worth evaluating when application-oriented task tooling is preferable to a bare queue contract. Kafka is a better direction when replay and multiple consumer groups are requirements; the managed queue described here retains messages for at most 30 days and deletes them on acknowledgment, so it isn't a Kafka-style event log. Choose Temporal or Airflow when the task is really a multi-step workflow with orchestration, dependencies, or join semantics. A simple delayed queue has no DAG or fan-out/join primitive.

There is one more hard boundary: push subscription targets must be public HTTPS endpoints. An internal-only consumer won't receive those pushes. Use a pull consumer that can reach the queue, or select infrastructure that fits the private network. I'm not sure which option wins without knowing the deployment boundary, because that fact changes the architecture more than a feature checklist does.

My default remains narrow: queue one small renewal-reminder message per business deadline, persist a stable delivery ID, acknowledge only after success, and keep the destination idempotent. Use cron around the edges for recurring triggers. Outsource the undifferentiated delivery machinery, then get back to the product.

## References

- RFC 2104, HMAC: https://www.rfc-editor.org/rfc/rfc2104
- RabbitMQ priority queues: https://www.rabbitmq.com/docs/priority
