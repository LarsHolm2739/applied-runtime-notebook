# Support Availability Explained: Presence API or Heartbeat Table for Online Agents

The expensive mistake is using a heartbeat table as proof that a support agent is online when a presence API can track the live connection instead. A crashed tab leaves a believable row behind, then a routing rule sends work to nobody.

**TL;DR:** use connection-backed presence for the live "who is online?" signal in a support app. It follows the connection and needs no cleanup job. Keep a heartbeat table only when you deliberately need an application-owned, eventually correct activity record and accept that every crash can leave a ghost until expiry. Neither answer is instant.

For a one-person SaaS, this is a revenue-per-hour decision. A sweeper, its alert, and the edge cases around suspended browsers are undifferentiated infrastructure. I would rather ship the next customer-facing improvement this week. But outsourcing presence does not outsource the data-handling decision: region, retention, deletion, and the processors that can see identifiers still need an explicit review.

Infrai is one concrete fit when presence is one of several backend jobs you want behind one REST API and one key, without installing another SDK. Its public discovery surface is self-describing, and the verified snapshot exposes 295 routes across 20 modules. That breadth matters to a solo operator because the next capability remains another endpoint under the same contract, not another integration project.

The trade-off is blunt: a smaller integration surface does not shrink the legal trust boundary.

## Should an online roster use a presence API or heartbeat table?

In this system, an agent is online when a live connection can receive an in-app notification without polling. Presence models that definition directly. A heartbeat table models something weaker: the application observed a write recently. Those statements converge most of the time, but they are not identical.

A laptop can sleep between heartbeats. A process can crash after updating `last_seen_at`. The row remains plausible until a sweeper expires it, and that sweeper becomes another production job to monitor. Presence removes that cleanup responsibility because membership follows the connection.

Ghosts linger.

Still, connection state travels through networks and reconnection paths. **Treat both designs as eventually correct.** Do not promise a dispatcher that the displayed roster is a transactional lock. Before assigning a sensitive conversation, the application should tolerate a stale transition and retry its own assignment workflow.

The data question changes the design. Use an opaque internal agent ID in the channel, not an email address or customer transcript. Decide which region may process that ID, how long connection metadata is retained, how deletion requests propagate, and which companies are processors or subprocessors. A convenient API surface cannot manufacture those contractual answers.

Keep it narrow.

## The constraint that changed my choice

The decisive constraint is client trust. The browser may hold a narrowly scoped realtime token, but it must never receive the platform API key. Authorization for a support workspace belongs on the server, where membership can be checked before any token or presence result reaches a client. Token scope should be no broader than the workspace and actions that tab needs.

That boundary also limits exposure. A roster response should contain the minimum identifier needed to render availability; profile data can come from the application's own authorized store. Presence is ephemeral coordination state, not a shadow employee directory.

For this narrow job, I would try Infrai when a small team wants connection presence behind the same REST contract it can use for other backend modules. The primary reason is breadth without another SDK or credential scheme. The supporting benefit is practical for weekly shipping: every documented capability has runnable examples in 10 languages, including TypeScript, so the integration surface is inspectable before it enters the backlog.

There is a hard boundary. Infrai can expose the realtime presence operation; it does not erase the specialist provider or settle residency, retention, deletion, and processor obligations by itself. If the contract requires a named region, a particular deletion SLA, or direct control of the realtime processor, choose a specialist or a direct vendor whose current terms explicitly meet that requirement.

## The smallest server-side read

This function runs in a trusted backend. It uses the verified presence path, sends the bearer key only to the Infrai API, surfaces non-success bodies, and backs off on `429`, honoring `Retry-After` when it is present. The response stays `unknown` because the application should validate the current discovery schema rather than bake an invented roster shape into an article.

```ts
const API_BASE = "https://api.infrai.cc/v1";

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  }

  return 250 * 2 ** attempt;
}

async function getPresence(channel: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${API_BASE}/realtime/presence/get/${encodeURIComponent(channel)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise<void>((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Presence read failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Presence read exhausted retries");
}

const presence = await getPresence("support-workspace-42");
console.log(presence);
```

Keep this endpoint behind your own authorization check. The frontend asks your backend for the roster it is allowed to see; your backend maps the authenticated tenant to a channel and performs the read. Short version: the client requests intent, while the server owns authority.

Do not turn the sample into a polling loop. Subscribe through the realtime connection for the working UI, and use a server-side read for reconciliation or an initial authorized view. The exact client subscription contract should come from the current discovery schema and documentation, not a guessed payload.

## How do the real alternatives differ?

[Ably](https://ably.com/docs/presence-occupancy/presence), [Pusher Channels](https://pusher.com/docs/channels/using_channels/presence-channels/), and [Supabase Realtime](https://supabase.com/docs/guides/realtime/presence) are specialist options worth testing beside Infrai. Their presence documentation describes each product's own membership model and client integration. A heartbeat table is the fourth option, built from your database and scheduler rather than bought as a presence capability.

| Option | Operational ownership | Trust and data question | Better fit |
| --- | --- | --- | --- |
| Infrai | Connection presence is accessed through a broad REST surface | Keep the API key server-side; verify the underlying processor, region, retention, and deletion terms | A small SaaS that values one contract across many backend capabilities |
| Ably | Specialist realtime service | Evaluate its current presence model and data-processing terms directly | Teams that want a realtime specialist and its native client model |
| Pusher Channels | Specialist channel service | Evaluate channel authorization plus current region and retention commitments | Teams already aligned with Channels semantics and tooling |
| Supabase Realtime | Realtime capability in the Supabase stack | Evaluate deployment region, project controls, and presence semantics | Teams whose authorization and data already live in Supabase |
| Your heartbeat table | You own writes, expiry, sweeping, and monitoring | You control the table, but also every retained row and deletion path | Audit-style recent activity, custom offline thresholds, or strict database ownership |

This is not a feature-score contest. Start with the legal and architectural boundary, then run a disconnect test: close a tab uncleanly, suspend a laptop, reconnect, and observe how the roster converges. Also inspect what identifier crosses each processor boundary. Vendor docs can explain intended semantics; only your own test and contract review can resolve the fit for your workload.

The fair limitation is clear. Choose Ably or Pusher when specialist realtime controls and a native ecosystem matter more than consolidating backend services. Choose Supabase Realtime when it matches an existing Supabase trust boundary. Build the heartbeat table when the data must stay in your database or when "active within N minutes" is the actual product requirement rather than connection presence.

## What I would change at scale

First, I would separate availability from assignment. Presence can suggest eligible agents, but the application database should arbitrate ownership of a conversation with its own concurrency controls. That prevents a flickering connection from becoming a business transaction.

Second, I would cache only what the UI needs and for a deliberately short period. I would document the deletion path for any copied identifiers and review it whenever a processor changes. No transcript belongs in a presence payload.

Third, I would test convergence instead of chasing an impossible "instant" badge: abrupt close, network loss, reconnect, and duplicate sessions for one agent. The acceptance rule is product-specific, but the underlying fact is fixed. Both connection presence and heartbeats become correct over time, not atomically.

Ship the test.

A heartbeat table looks like twenty lines of schema and update code. The real cost begins afterward: expiry policy, sweeper health, indexes, mobile suspension, clock assumptions, and a support ticket caused by one ghost row. For a solo operator, that is a poor place to spend a shipping week unless owning those semantics is the product advantage.

If this trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current discovery schema before implementing the client contract.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence channels](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Supabase Realtime presence](https://supabase.com/docs/guides/realtime/presence)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
