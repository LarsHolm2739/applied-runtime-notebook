# Nightly Reconciliation Under a Rate Limit: Cron, Queues, and Node.js Webhooks

Short answer: let a cron trigger enqueue deterministic jobs every minute, then let a separate Node.js worker process them at the payment provider's allowed rate; the public webhook endpoint should validate, enqueue, and return without running reconciliation inline.

A nightly payment reconciliation has one hard constraint: the payment provider, not the cron clock, sets the processing speed. The design follows from that constraint. Scheduling creates work, admission deduplicates it, and a worker spends the limited request budget.

## How should a Node.js cron trigger queue rate-limited webhook processing every minute?

Treat the cron expression as a wake-up signal, not a throughput promise. On each invocation, calculate a deterministic reconciliation window, create a stable job ID, and enqueue that job if it isn't already present. The worker owns the provider limit. The webhook owns only request validation and admission.

The contract is small enough to state in a table:

| Component | Accepts | Must guarantee |
| --- | --- | --- |
| Minute trigger | A UTC tick | The same tick creates the same discovery ID |
| Public webhook | Signed raw bytes | A valid event is admitted once and gets `202` |
| Queue | A stable job ID | Reserved work returns after an unfinished lease |
| Worker | One reserved job | All replicas share the provider request budget |

That's enough.

That separation matters during a nightly customer-support workflow. Suppose the 02:14 UTC tick is delivered twice while a payment event for the same merchant arrives at the public endpoint. The scheduler retries should collapse to one discovery job, while the webhook event should retain its provider event ID and become one admission record. Both jobs may eventually point at the same merchant-window pair, so the reconciliation write also needs a stable business idempotency key. Without those identities at all three boundaries, a `202` only proves that one HTTP request was accepted; it says nothing about duplicate discovery or duplicate payment effects. With them, the support trail is mechanical: search the event ID, follow the merchant-window job, and inspect the final reconciliation key.

The key word is deterministic. A retry of the 02:14 scheduler tick should produce the same job ID as the original 02:14 tick. The queue's `enqueueOnce` operation can then reject the duplicate without treating it as an error. Don't derive that ID from the process start time or a random UUID. Those values make every retry look new.

Keep the scheduled unit small. For example, enqueue one merchant-window pair rather than one job for the entire nightly population. Small jobs can be retried independently and distributed later, but making them microscopic increases queue writes and coordination cost. The right grain is the smallest unit that can fail without forcing unrelated customer records through the payment API again.

## The smallest working boundary

The following TypeScript sketch uses generic interfaces on purpose. A database table with unique job IDs can implement `Queue`; so can a managed queue. The scheduling and worker code shouldn't care.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";
import { createHmac, timingSafeEqual } from "node:crypto";

type ReconcileJob = {
  id: string;
  merchantId: string;
  windowStart: string;
  attempt: number;
};

interface Queue {
  enqueueOnce(job: ReconcileJob): Promise<boolean>;
  reserve(): Promise<ReconcileJob | undefined>;
  complete(id: string): Promise<void>;
  retry(id: string, nextAttempt: number): Promise<void>;
  deadLetter(id: string, reason: string): Promise<void>;
}

interface PaymentClient {
  reconcile(merchantId: string, windowStart: string): Promise<void>;
}

export async function onMinute(queue: Queue, tick: Date): Promise<void> {
  const windowStart = tick.toISOString().slice(0, 16) + ":00.000Z";
  const merchantId = "support-prod";

  await queue.enqueueOnce({
    id: `reconcile:${merchantId}:${windowStart}`,
    merchantId,
    windowStart,
    attempt: 0,
  });
}

const pause = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function runWorker(
  queue: Queue,
  payments: PaymentClient,
  requestsPerMinute: number,
): Promise<void> {
  if (!Number.isInteger(requestsPerMinute) || requestsPerMinute < 1) {
    throw new Error("requestsPerMinute must be a positive integer");
  }

  const interval = Math.ceil(60_000 / requestsPerMinute);

  for (;;) {
    const job = await queue.reserve();
    if (!job) {
      await pause(250);
      continue;
    }

    try {
      await payments.reconcile(job.merchantId, job.windowStart);
      await queue.complete(job.id);
    } catch (error) {
      const nextAttempt = job.attempt + 1;
      if (nextAttempt >= 5) {
        await queue.deadLetter(job.id, String(error));
      } else {
        await queue.retry(job.id, nextAttempt);
      }
    }

    await pause(interval);
  }
}

function signatureIsValid(body: Buffer, supplied: string, secret: string): boolean {
  const expected = createHmac("sha256", secret).update(body).digest("hex");
  const left = Buffer.from(supplied, "hex");
  const right = Buffer.from(expected, "hex");
  return left.length === right.length && timingSafeEqual(left, right);
}

export function webhookServer(queue: Queue, secret: string) {
  return createServer(async (req: IncomingMessage, res: ServerResponse) => {
    if (req.method !== "POST" || req.url !== "/webhooks/reconcile") {
      res.writeHead(404).end();
      return;
    }

    const chunks: Buffer[] = [];
    for await (const chunk of req) chunks.push(Buffer.from(chunk));
    const body = Buffer.concat(chunks);
    const signature = String(req.headers["x-webhook-signature"] ?? "");

    if (!signatureIsValid(body, signature, secret)) {
      res.writeHead(401).end();
      return;
    }

    const event = JSON.parse(body.toString()) as {
      id: string;
      merchantId: string;
      windowStart: string;
    };

    await queue.enqueueOnce({
      id: `webhook:${event.id}`,
      merchantId: event.merchantId,
      windowStart: event.windowStart,
      attempt: 0,
    });

    res.writeHead(202).end();
  });
}
```

The example is deliberately missing a concrete queue adapter, HTTP body limit, schema validator, deployment wrapper, and payment SDK. Those are environment choices, and pretending otherwise would make the sample look complete while hiding its actual contract. In production, reject an oversized body before buffering it, validate parsed fields, store the provider's raw event ID for deduplication, and load secrets from the runtime's secret store. Authentication must be checked against the exact raw bytes the sender signed.

There is another sharp edge. The fixed delay limits one worker, not the whole deployment. Start three replicas and the aggregate request rate can become three times the configured value. For the smallest deployment, run one worker and alert if queue age climbs. Once horizontal workers are necessary, replace the local delay with a shared token bucket or lease-based rate limiter. That's the moment to pay the coordination cost, not during week one.

## Prove the admission contract before deployment

A successful enqueue is not successful reconciliation. Track at least job age, attempt count, completion time, and dead-letter count. Logs need the stable job ID, merchant ID, and reconciliation window so a support ticket can be traced across the scheduler, webhook, queue, and worker without matching timestamps by eye.

Retries need classification. A temporary provider throttle can return to the queue with backoff and jitter; a `401` or malformed account data needs operator attention. The sample caps attempts at five to make the control flow concrete, but five isn't a universal policy. Set the real cap from the provider's documented error semantics and the business deadline, then test it with a fake client that returns each relevant status in sequence. I'm not sure any fixed retry count is defensible without those two inputs.

Dead-letter queues are useful because they isolate repeatedly unsuccessful messages for later inspection, and the AWS SQS documentation also warns that moving messages can affect ordering. That is an architectural trade-off, not a reason to skip dead-letter handling. If strict order changes the meaning of payment adjustments, preserve ordering in the primary design and define a deliberate replay procedure. Otherwise, a poisoned job can block useful work or be silently discarded. Neither outcome belongs in a reconciliation system.

Test time itself. Invoke the same cron tick twice and assert one queued job. Deliver the same webhook event twice and assert one queued job. Stop the worker after the provider call but before `complete`, then confirm that a retry doesn't duplicate the business effect; this requires the downstream operation to accept an idempotency key or the application to record the result transactionally. Finally, advance a fake clock and verify that the global request budget holds. Real sleeps make this test slow and vague.

No heroics. A queue depth chart and a replay command are worth more than clever retry recursion at 03:00.

## Scale only after measuring queue age

The first upgrade would be admission control across all workers. Use a shared limiter, reserve jobs with a visibility timeout or lease, and renew that lease only while processing is alive. The second would be fair scheduling by merchant so one large account can't occupy every token while smaller support queues wait. The third would split discovery from execution: one scheduled job finds merchant-window work, while many bounded jobs perform reconciliation.

Cron overlap needs an explicit policy too. Kubernetes documents that CronJob scheduling is approximate, that concurrent runs can be controlled with a concurrency policy, and that jobs should be idempotent. Even outside Kubernetes, those are sound design questions: can two discovery runs overlap, what happens after a missed tick, and does replaying a tick create the same jobs? Answer them in application state rather than assuming the scheduler fires exactly once.

The catch is latency. A single rate-limited worker is inexpensive to understand and operate, but queue wait grows when arrivals exceed its drain rate. It is not suitable when a customer-support agent needs reconciliation before ending a live conversation, or when the nightly batch cannot finish inside its business window. In that case, negotiate a higher provider quota, partition work across independently limited accounts when the provider permits it, or move truly urgent jobs into a separately budgeted lane. Stick with direct synchronous processing only when the provider limit, request volume, and failure semantics are trivial enough that queueing adds more operational risk than it removes.

For a one-person SaaS, the revenue-per-hour decision rule stays blunt: outsource undifferentiated queue durability when operating it would consume feature weeks, but keep the interfaces portable and the rate policy in application code. Ship weekly. Add shared coordination when measured queue age demands it, and don't confuse a more elaborate scheduler with more payment throughput.

## References

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
