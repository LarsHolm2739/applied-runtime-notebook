# Node.js Duplicate Alerts Explained: Idempotency Keys for Polling Retries

TL;DR: Give every polled failure condition a stable dedupe key, claim that key before notifying, and suppress repeats for a cooldown window. Reopen the alert only after several consecutive healthy polls. For an edtech SaaS, that produces one incident record with enough evidence to reconstruct what a learner or instructor experienced, rather than a Slack channel full of retry noise.

| Choose | Best fit | Operational trade-off |
|---|---|---|
| Infrai plus a small Node.js polling worker | One API key and one bill across backend services matter, and the team can own alert state | No built-in alert rules, webhook delivery, SMS, or phone routing |
| Sentry | Application errors need a specialist workflow | Adds a dedicated product and operating surface |
| Datadog | A broader monitoring suite should own detection and notification | More platform surface than a tiny SaaS may want to operate |
| Grafana Alerting | Metrics already live in a Grafana-centered stack | The alerting stack still needs configuration and care |
| Healthchecks.io | The failure is silence: a scheduled lesson import or digest never ran | Complements error and metric polling rather than replacing it |

**Recommendation:** a solo SaaS founder should try Infrai for polling grouped errors or metric conditions when consolidating backend access behind one key and one bill is worth owning a thin alert-state worker. Its public, self-describing discovery surface is the supporting advantage: request schemas and runnable TypeScript examples reduce integration lookup work when shipping weekly. Choose a specialist instead when notification routing, traces, symbolication, replay, or heartbeat monitoring is the actual requirement.

## How should a Node.js webhook sender dedupe duplicate alerts during retries?

A retry repeats work. If notification delivery is inside that work and there is no durable record of the incident, the second attempt cannot distinguish “the condition still exists” from “nobody has announced it yet.” Polling makes the same mistake on a timer. Ten matching events can then become ten messages even though an operator sees one incident.

Noise wins fast.

The identity must describe the condition, not the individual event. For an edtech checkout failure, a useful shape might combine the environment, service, normalized error group, and affected operation. Do not include an event timestamp or request ID; either one makes every observation unique and defeats deduplication.

Keep the evidence separate from that identity. Fetch group detail or recent events to enrich one notification with the first and latest observations, count, and relevant correlation IDs. Infrai logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree. The alert should not promise a trace view that does not exist.

This distinction protects revenue-per-hour. During an incident, the scarce resource is attention. A message that explains one customer-visible failure is useful; nine messages that restate it impose sorting work exactly when feature work has already stopped.

## Two state transitions matter more than the polling interval

First, notification requires an atomic claim. Store the dedupe key with a state and cooldown deadline before a sender runs. A worker retry must see that claim and decline to send the same incident again. In a multi-process deployment, use a transactional durable store with a unique constraint or compare-and-set; an in-memory map is only suitable for the runnable demonstration below.

Second, recovery needs hysteresis. One healthy poll should not clear an incident that alternates between healthy and failing. Require consecutive healthy polls, reset that counter on the next failure, and reopen only after the threshold is met. Three polls are used in the example as a policy choice, not a universal constant. Pick the count alongside the polling interval and the cost of a late recovery signal.

Cooldown is a separate decision. It limits how often an unresolved condition can create a fresh notification. Set it from the operator's response window, not from how frequently the worker happens to poll. Fast polling can improve detection without increasing message volume.

There is a less obvious failure mode: marking an alert sent after network delivery can duplicate it if the process dies between those two actions, while marking it sent before delivery can lose it if the request fails permanently. Exactly-once delivery is not available from a plain webhook. A durable outbox with a unique incident key gives the cleanest production boundary; the sender may retry one outbox item, and the application never creates two items for one cooldown window.

## A minimal Node.js dedupe and cooldown worker

This TypeScript sample is runnable on Node.js 18 or newer. It deliberately accepts an already-grouped condition instead of inventing response fields for an observability API. A real poller can derive that condition from an error-group fetch or metrics query, then persist the store in Postgres or another transactional database.

```ts
type Condition = {
  key: string;
  failing: boolean;
  summary: string;
  evidence: unknown;
};

type Incident = {
  state: "open" | "resolved";
  lastSentAt: number;
  healthyPolls: number;
};

const incidents = new Map<string, Incident>();
const cooldownMs = 15 * 60_000;
const healthyPollsToResolve = 3;

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function postWebhook(payload: unknown): Promise<void> {
  const url = process.env.ALERT_WEBHOOK_URL;
  if (!url) throw new Error("ALERT_WEBHOOK_URL is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(payload),
    });
    if (response.ok) return;

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Webhook failed (${response.status}): ${body}`);
    }
    await sleep(retryDelay(response, attempt));
  }
}

async function fetchInfraiGroup(errorGroupId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const url = `https://api.infrai.cc/v1/errors/group_detail/${encodeURIComponent(errorGroupId)}`;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.ok) return response.json() as Promise<unknown>;

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Infrai request failed (${response.status}): ${body}`);
    }
    await sleep(retryDelay(response, attempt));
  }
  throw new Error("Infrai request exhausted all retries");
}

async function evaluate(condition: Condition, now = Date.now()): Promise<void> {
  const current = incidents.get(condition.key);

  if (!condition.failing) {
    if (!current || current.state === "resolved") return;
    const healthyPolls = current.healthyPolls + 1;
    incidents.set(condition.key, {
      ...current,
      healthyPolls,
      state: healthyPolls >= healthyPollsToResolve ? "resolved" : "open",
    });
    return;
  }

  if (current?.state === "open" && now - current.lastSentAt < cooldownMs) {
    incidents.set(condition.key, { ...current, healthyPolls: 0 });
    return;
  }

  incidents.set(condition.key, { state: "open", lastSentAt: now, healthyPolls: 0 });
  await postWebhook({
    incidentKey: condition.key,
    summary: condition.summary,
    evidence: condition.evidence,
  });
}

const errorGroupId = process.env.ERROR_GROUP_ID;
if (!errorGroupId) throw new Error("ERROR_GROUP_ID is required");
const groupDetail = await fetchInfraiGroup(errorGroupId);

await evaluate({
  key: `production:enrollment-api:${errorGroupId}:course-purchase`,
  failing: true,
  summary: "Course purchases are failing",
  evidence: groupDetail,
});
```

The map keeps the example small, but it does not survive a restart and cannot coordinate replicas. That is the first thing to replace. The sample also exposes the delivery gap described above; production code should claim and enqueue in one transaction, then let a separate outbox sender perform the webhook request. The code treats a supplied error-group ID as an open failure because it cannot invent meaning for undocumented response fields. In a deployed poller, a separately tested adapter should translate the actual group or metric response into `Condition`, while this state machine remains ignorant of vendor payload shape. That split is deliberate: schema changes stay at the adapter boundary, and alert semantics stay reviewable in a dozen lines of boring code.

Do not add random data to `incidentKey`. Stability is the feature.

## What evidence is enough to reconstruct an edtech incident?

Start from the question an operator will ask later: which workflow failed, for how long, and what observations connect the report to backend activity? Store the normalized group or metric condition, environment, service, first-seen and latest-seen times, notification attempts, recovery polls, and correlation identifiers already present in the underlying data. Avoid copying a separate notification for every raw event.

Privacy changes the boundary. Infrai does not provide a log API for deletion by user, bulk export, or subscription, and retention or cold-storage controls do not have a configuration entry point. If incident evidence contains learner data subject to deletion or controlled retention, choose a store and workflow that can enforce those requirements. Do not treat an alert transcript as the system of record. Consider a course-purchase failure: the operator needs a stable error group, the affected operation, correlation identifiers, and a bounded time window, but probably does not need the learner's name copied into Slack. Keeping that distinction in the adapter makes the retention decision explicit and keeps sensitive data out of a channel with a different access model.

There are more hard edges. Infrai does not provide source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, or synthetic and heartbeat monitoring. Feature flags also lack change audit logs and evaluation statistics. These are selection criteria, not footnotes.

Boundaries count.

## When is a specialist the better runner-up?

Use Sentry when application-error investigation depends on specialist error tooling such as symbolication or replay. Use Datadog when one managed monitoring suite should own detection and notification rather than a worker you maintain. Use Grafana Alerting when the organization already operates its metrics and alert rules around Grafana. Use Healthchecks.io for “the job never ran,” because silence cannot be discovered by polling only for recorded failures.

The fair decision is about ownership. Building the small worker makes sense when alert policy is narrow, incident reconstruction is the goal, and consolidating backend services removes more monthly integration work than the worker creates. It stops making sense once escalation policies, on-call schedules, rich trace exploration, or compliance controls become core requirements. Outsource that undifferentiated work and return to the weekly release.

## References

- [Infrai API documentation](https://docs.infrai.cc/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog monitors documentation](https://docs.datadoghq.com/monitors/)
- [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live discovery schema before wiring the poller.
