# Live Queue Position Updates for Learning Rooms with Lease Snapshots

TL;DR: For an edtech shared workspace, represent online status as a short lease, publish coarse state changes, and make reconnecting clients fetch a snapshot before they trust live events. The least complex design is a server-owned presence record with an expiry time. A queue-like waiting room can use the same rule: show a student's current place from a snapshot, then use events as a prompt to refresh it.

| Choice | Presence accuracy | Failure behavior | Work for a one-person SaaS |
| --- | --- | --- | --- |
| Client heartbeat plus server lease | High enough for UI | Expiry removes stale users | Low |
| Open socket means online | Poor after sleep and network changes | Stale users linger | Low at first, high later |
| Periodic polling only | Predictable but coarse | No lost event problem | Medium |
| Event stream plus snapshot recovery | High with a little more state | Reconnect can repair gaps | Medium |

My choice is the first model combined with the last recovery rule. Keep the lease authoritative on the server. Let the browser suggest liveness with a heartbeat. On reconnect, discard assumptions and read a fresh snapshot. That keeps the implementation small while giving a support queue or classroom workspace a defensible answer to “who is online?”

## How should a waiting room show queue position during live updates?

It does not mean that a tab exists. It means that the service has received a valid signal recently enough for the product's promise. Write that definition down before choosing a transport.

That boundary is product policy.

For example, a workspace might use a 30-second lease and renew it every 10 seconds. Those numbers are policy, not a universal standard. A student who closes a laptop may remain visible for at most roughly one lease interval plus detection and delivery delay. If that delay is unacceptable, shorten the lease or label the state more honestly: “active recently” is better than pretending to know the instant a person walked away.

This distinction matters in education. A teacher may interpret a green dot as attendance, while a group member may only need to know whether edits are likely to be seen. One boolean cannot carry both meanings. Keep `online`, `activeRecently`, and attendance evidence separate. Presence is a coordination hint, not an audit record.

I also avoid broadcasting a precise roster on every heartbeat. That turns a small signal into a noisy fan-out problem. Renew leases in storage, publish only meaningful transitions, and let clients refresh the room snapshot after reconnect. The revenue-per-hour test is simple: if a detail does not change a user decision, it probably should not be a realtime event.

## Which delivery shape survives a missed event?

A live event is useful, but it is not a database. Mobile radios sleep. Browser tabs are suspended. A WebSocket can close between two messages, and the client may reconnect without knowing exactly which update it missed. Treating the event stream as the complete truth makes a temporary network gap look like a permanent product state.

Use two paths:

1. A snapshot returns the current members, their coarse status, and a version or observed-at value.
2. A live channel announces that the snapshot may have changed.

The event can stay deliberately boring: `workspace.changed`. It need not contain every member's private metadata. The client receives the notice, asks for the current snapshot, and replaces local state. This is easier to reason about than reconstructing a long sequence of deltas, especially when a solo team is shipping weekly and has no separate reliability group.

A waiting room for support works the same way. Store the queue position as server-owned state. Send a change notification when the position may have changed. On reconnect, fetch the position again. Never infer “still number 4” from an old event merely because the socket is open.

## A small lease and snapshot boundary

The code below shows the boundary, not a framework-specific service. The storage adapter can be backed by a transactional database or a shared in-memory store. The important rules are that the server sets the expiry, compares versions when it publishes, and never accepts a client-supplied online flag as truth.

```ts
type Presence = {
  userId: string;
  workspaceId: string;
  state: "online" | "offline";
  leaseExpiresAt: number;
  version: number;
};

interface PresenceStore {
  renew(input: { userId: string; workspaceId: string; expiresAt: number }): Promise<Presence>;
  expire(now: number): Promise<Presence[]>;
  snapshot(workspaceId: string, now: number): Promise<Presence[]>;
}

interface Broadcaster {
  changed(workspaceId: string, version: number): Promise<void>;
}

const LEASE_MS = 30_000;

export async function renewPresence(
  store: PresenceStore,
  broadcaster: Broadcaster,
  workspaceId: string,
  userId: string,
  now = Date.now(),
): Promise<Presence> {
  const presence = await store.renew({
    workspaceId,
    userId,
    expiresAt: now + LEASE_MS,
  });

  await broadcaster.changed(workspaceId, presence.version);
  return presence;
}

export async function readWorkspacePresence(
  store: PresenceStore,
  workspaceId: string,
  now = Date.now(),
): Promise<Presence[]> {
  return store.snapshot(workspaceId, now);
}
```

A production implementation needs an idempotent renew operation and a cleanup path for expired leases. The cleanup path should be safe to run more than once. Publish after the state transition is committed, or use an outbox if the storage write and notification cannot share a transaction. Do not make a browser request responsible for removing someone else.

For a customer queue, the returned position also needs a stable queue version. A client can display “position 4” with the time it was observed, then replace it after a `queue.changed` notification. That small bit of provenance prevents a stale number from masquerading as a current promise.

## How should reconnect and testing shape the decision?

Reconnection is part of the normal state machine. On connect, authenticate the workspace, fetch a snapshot, subscribe to changes, and fetch again if the subscription boundary is ambiguous. On disconnect, stop claiming freshness. The UI can show “reconnecting” without turning every transient network event into an offline presence write.

Test the ugly sequences, not just the happy demo:

- a heartbeat arrives twice;
- a lease expires while a delayed renewal is in flight;
- two browser tabs renew the same user;
- the client misses three change events;
- the queue moves while a position response is in transit;
- a user loses access while reconnecting.

Use fake clocks for expiry tests and a small deterministic event log for reconnect tests. Measure stale-visible duration, snapshot latency, notification lag, and the number of refreshes caused by reconnect. Those metrics tell me more than a raw message-per-second figure.

There is a useful product boundary here. If the room only needs a rough list of active collaborators, polling every 15 to 30 seconds may be enough and is easier to operate. If a teacher is coordinating a timed exercise or a support agent is assigning the next customer, lease-based state plus snapshot recovery earns its extra moving parts. The runner-up is polling when freshness is intentionally coarse.

The lease model has a real limitation: it cannot prove that a person is looking at the screen. A short lease improves stale-state detection but increases renewals and can mark a user away during a brief outage. A longer lease reduces that churn while leaving stale names visible longer. Choose the interval from the user consequence, and show “active recently” when the evidence does not justify “online.”

**The decision is about the meaning of “online,” not the novelty of the transport.** Pick the smallest mechanism that can uphold that meaning after sleep, reconnect, duplicate delivery, and a missed event. That is a better weekly shipping target than building a perfect presence mesh.

## Further reading

- https://www.w3.org/TR/webrtc/
- https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
- https://www.rfc-editor.org/rfc/rfc6455
