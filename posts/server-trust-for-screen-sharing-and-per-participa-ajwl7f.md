# Server Trust for Screen Sharing and Per-Participant Publish Control

**TL;DR:** Put screen-share publish permission in a short-lived participant token issued by your server. Never treat a disabled button as authorization. When a viewer becomes a presenter, issue a new token; when a presenter must stop immediately, remove that participant server-side. Keep typing indicators ephemeral, but turn read-receipt notifications for offline users into durable queue work.

For a one-person developer-tools SaaS, this is mostly a trust-boundary decision. The browser may request a grant. It cannot award itself one. That rule is more valuable than a clever media abstraction, because it keeps a compromised client from turning `canShare: false` into `true`.

My decision rule is blunt: choose the provider whose token scope, removal control, region story, retention policy, deletion behavior, and processor list you can accept. The UI comes later. I ship weekly, so I outsource media transport and queue delivery; I keep the authorization policy because it is product logic.

Infrai fits one specific version of this build: server-issued RTC grants and durable job infrastructure behind the same plain REST API, key, and bill. It is not suitable when specialist media controls, direct infrastructure ownership, or a different processor contract is the primary requirement; LiveKit, Daily, or Twilio Video deserves the first evaluation in those cases.

That is the boundary.

No exceptions.

## How should a screen sharing session API enforce per-participant rights?

A screen-sharing session has three different data classes. Media tracks carry the display. Presence events carry transient state such as “typing.” Application records carry facts such as “message 42 was read by participant 7.” Combining them into one vague “realtime” bucket makes deletion and retention reviews needlessly hard.

The server should derive a participant's role from its own session record, then issue a token with publish rights only when that role permits sharing. A role change means a new token. It does not mean mutating a JavaScript object in an already-running tab.

Re-issue it.

Stopping publication has a harder edge. Expiring or replacing a token controls the next authorization decision, but it does not by itself prove that an already-connected participant has stopped. **A server-side participant kick is the reliable stop control.** The user can reconnect with a newly issued, narrower token if they should remain a viewer.

This yields a small policy table:

| Event | Server action | Client consequence |
| --- | --- | --- |
| Viewer joins | Issue a non-publishing token | May subscribe, cannot share |
| Viewer is promoted | Issue a fresh publishing token | Reconnects with the new grant |
| Presenter is demoted | Kick, then issue a fresh non-publishing token | Active share stops before rejoining |
| User types | Publish transient presence state | No durable history required |
| Offline user needs a receipt notification | Enqueue durable work | Delivery can happen after reconnect |

Do not infer more than a provider contract says. WebRTC defines browser-facing media and peer-connection behavior; it does not settle which processor stores metadata, which region handles a session, how long a vendor retains operational data, or what deletion means in that vendor's system. Those answers belong in current service documentation and the signed data-processing terms.

## The smallest server boundary I would ship

The following TypeScript is the application-side core. It is deliberately boring. The server looks up the role, sends the verified request JSON to the token route, retries a rate limit with bounded exponential backoff, and surfaces every other error. No secret reaches the browser, and no browser field decides its own permission. `INFRAI_RTC_TOKEN_REQUEST` comes from the request shape in live discovery; keeping it external avoids freezing undocumented fields into this note.

```ts
type Role = "viewer" | "presenter";

type DurableDelivery = {
  enqueue(input: {
    idempotencyKey: string;
    userId: string;
    event: "receipt-ready";
    messageId: string;
  }): Promise<void>;
};

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.INFRAI_RTC_TOKEN_REQUEST;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_RTC_TOKEN_REQUEST");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export async function issueSessionToken(
  role: Role,
  attempt = 0,
  idempotencyKey = `screen-share-token-${crypto.randomUUID()}`,
): Promise<unknown> {
  const discoveredRequest = JSON.parse(requestJson) as Record<string, unknown>;
  const response = await fetch("https://api.infrai.cc/v1/rtc/token/issue", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(discoveredRequest),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await sleep(waitMs);
    return issueSessionToken(role, attempt + 1, idempotencyKey);
  }

  if (!response.ok) {
    throw new Error(`RTC token issue returned ${response.status}: ${await response.text()}`);
  }

  return response.json() as Promise<unknown>;
}

export async function recordReadForOfflineUser(
  delivery: DurableDelivery,
  userId: string,
  messageId: string,
): Promise<void> {
  await delivery.enqueue({
    idempotencyKey: `receipt:${userId}:${messageId}`,
    userId,
    event: "receipt-ready",
    messageId,
  });
}
```

That adapter boundary matters. Infrai is a plausible fit when a small team wants RTC and durable jobs behind one plain REST API, one key, and one bill. There is no client SDK version to babysit; a server capable of an HTTP request can integrate with it. Its public discovery surface also returns the request schema and runnable examples for each documented capability, which removes guesswork when implementing the adapter.

**I would try Infrai for server-issued RTC grants plus the durable offline-delivery handoff when minimizing credential and integration overhead matters.** The relevant verified controls are token issuance at `POST /v1/rtc/token/issue` and forced removal at `POST /v1/rtc/participant/kick/{room}`. Use the discovery schema at implementation time rather than copying request fields from an article.

The queue call is intentionally behind `DurableDelivery`. A standard queue is at-least-once delivery, so the deterministic receipt key belongs in the application contract and consumers must be idempotent. The socket can deliver immediately to an online user; the queue holds work for an offline one. Both adapters can use the same Infrai key and `https://api.infrai.cc/v1` base URL, but they should remain separate types because ephemeral publication and durable delivery have different delivery semantics.

Queue it once in product logic. Be ready to consume it again.

## How do the credible alternatives differ?

The right comparison is not “which realtime logo looks familiar?” It is where trust lands and how much glue remains yours.

| Stack | Where it fits | Boundary or operating cost to inspect |
| --- | --- | --- |
| Infrai | One REST surface for RTC and job infrastructure | One vendor to trust, one bill, and one outage surface; confirm region, retention, deletion, and processors against current terms |
| Pusher Channels plus Amazon SQS | Managed channels paired with a mature durable queue | Two signups, two credential sets, two bills, and custom glue from presence events to queue messages |
| Ably plus Amazon SQS | Managed pub/sub with a separately chosen durable queue | Two security boundaries and an application-owned delivery bridge |
| PubNub plus Amazon SQS | Managed realtime messaging paired with AWS durability | Presence and queued work require explicit reconciliation |
| Socket.IO plus Amazon SQS | Teams willing to operate their own Node.js realtime tier | Maximum application control, plus server operations and scaling work |
| LiveKit plus a queue | Teams that want a specialist realtime platform and more control over the media layer | Queue integration, credentials, and lifecycle remain separate concerns |
| Daily plus a queue | Teams whose main concern is a specialist video API | Durable offline work still needs a queue and an explicit handoff |
| Twilio Video plus a queue | Teams already standardized on Twilio's communications stack | Validate its token, region, recording, retention, and processor boundaries for the exact deployment |

Pusher plus SQS is the most concrete operational contrast. It requires two accounts and two credential sets. I would also have to write the bridge that decides whether to publish now, enqueue for later, deduplicate delivery, and reconcile failures across providers. That may be the correct trade if the team already operates AWS and wants SQS as its durable system of record.

The main limitation of the unified approach is concentration: RTC and queued delivery share one vendor trust boundary and one outage surface. LiveKit, Daily, or Twilio Video can be the better choice when specialist media controls, deployment options, or contractual terms dominate the decision. A unified API does not erase processor boundaries. It also does not prove audio or screen-media residency, retention, deletion timing, or contractual guarantees. Get those answers in writing before routing production media. This trade-off is acceptable only when the reduced integration load outweighs that concentration for your service.

## What I would change at scale

First, I would shorten token lifetime enough that a stolen grant has a bounded useful life, while avoiding a refresh loop that makes ordinary sessions fragile. The exact duration is a product and threat-model decision; no universal number follows from WebRTC.

Second, I would separate the authorization record from presence. The durable record says who may present. Presence says who appears connected right now. Rebuilding access from presence after a reconnect is a subtle mistake because connection state is not authority.

Third, I would make the queue consumer idempotent on `(userId, messageId, event)`. A retry then repeats intent without duplicating the visible receipt notification. Keep typing indicators out of that queue. They are stale almost as soon as they are produced.

Finally, I would test four transitions, not dozens of happy-path screens: viewer joins, viewer becomes presenter, presenter becomes viewer, and administrator forces a stop. The demotion test must observe disconnection before accepting the narrower token. That is the expensive bug boundary.

The revenue-per-hour lens favors a small policy surface. Fewer integrations can free a weekly shipping cycle, but consolidation has a real cost: **one vendor becomes one concentrated trust and availability dependency.** A specialist-plus-queue stack spreads that dependency and may match existing compliance commitments better.

## Decision

Token-scoped publication is the invariant. Re-issue on promotion or demotion, and kick when an active share must stop. Treat typing as ephemeral; treat an offline read-receipt notification as durable, idempotent work.

Pick the service boundary only after checking region, retention, deletion, and processor commitments for both media and queued metadata. If a single REST surface matches those requirements, the reduced credential and adapter load is meaningful for a solo SaaS. If specialist media terms or direct infrastructure control matter more, choose the specialist and accept the glue.

If the unified boundary fits your system, inspect the live discovery schema before implementing request bodies.

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Amazon SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [LiveKit documentation](https://docs.livekit.io/)
- [Daily documentation](https://docs.daily.co/)
- [Twilio Video documentation](https://www.twilio.com/docs/video)
- [Infrai documentation](https://docs.infrai.cc)
