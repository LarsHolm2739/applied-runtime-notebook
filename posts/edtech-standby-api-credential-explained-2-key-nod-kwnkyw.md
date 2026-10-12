# Edtech Standby API Credential Explained — 2-Key Node.js Incident Playbook

TL;DR: Pre-create one disabled standby credential, store it separately, and switch credentials at the request boundary while keeping the billing event identity unchanged. For an edtech platform, the winning design is the one that preserves each customer's usage attribution during recovery. A fast failover that bills a lesson twice is not continuity.

| Choice | Incident readiness | Billing attribution risk | Operating burden |
|---|---|---|---|
| Create a key during the incident | Low; provisioning is on the critical path | Medium; rushed configuration can lose tenant context | Low before the incident, high during it |
| Pre-create an active spare | High | High if both credentials can be used accidentally | Medium; both secrets require protection and monitoring |
| Pre-create a disabled spare | High after an enable step | Low when activation and cutover are recorded together | Medium; rehearsals must test the enable path |

**Decision: use a pre-created, disabled spare when the upstream account supports it.** Keep the active and standby credentials in separate secret records, attach the same least-privilege scope, and rehearse activation without emitting a billable event. If disabling is unavailable, an active spare can work, but it needs stricter access controls and alerts because it is already usable.

This choice spends a little operating time to protect the number that matters: attributable customer usage. For a one-person SaaS, that is a good revenue-per-hour trade. The secret plumbing is undifferentiated work; keep it small, automate the proof, and get back to shipping weekly.

## Should we create a standby API credential before an incident?

Credential failover and billing failover are different state transitions. The credential answers, "May this process call the upstream API?" The usage event answers, "Which customer consumed what, and has this event already been counted?" Joining those concepts by using the API key as the billing identity creates a quiet failure: switching keys can split one customer across two counters or replay the same operation under a new identity.

The stable identity belongs in the application data. Give every attempted unit of work an immutable event ID, tenant ID, meter name, and quantity. Persist that record before making the external call. The active credential version is useful audit metadata, but it must not be the idempotency key or the customer key.

Consider a tutoring platform processing a 42-minute live lesson. A timeout leaves the caller unsure whether the upstream accepted the request. The process then switches to the standby credential and retries. Without a stable event ID, both attempts may become invoice line items. With one event ID and a uniqueness constraint in the metering store, the retry remains one usage event even though two credential versions appear in the attempt log.

Walk that sequence all the way through. At 10:14:00, the worker reserves `lesson_7f3` for tenant `school_21` and captures primary credential version `v7`. The upstream receives the call, but the response never reaches the worker. At 10:14:05, an operator approves the cutover to standby version `v8`; only newly admitted jobs see it. The uncertain job stays reserved rather than being marked failed, because a transport timeout cannot establish that no work occurred. A reconciliation worker can query whatever status mechanism the upstream actually documents, or it can send a retry only when the operation accepts the same idempotency identity. Either way, the usage ledger still has one `lesson_7f3` row. The timestamps and labels here illustrate the state transitions; they are not a recovery-time claim. This is the unglamorous part of the design, and it is where invoice accuracy is won.

That is the main criterion: **a credential change must never change billing identity**. Test this before optimizing recovery speed.

No guessing.

## Two boundaries decide whether recovery is safe

The first boundary is activation authority. OWASP recommends centralizing secrets management, applying least privilege, automating rotation where possible, and keeping audit records for secret use and changes. A standby credential should follow the same rules as the primary. Pre-created does not mean broadly readable. Only the workload that needs the secret and the operator or automation responsible for activation should have access.

Separate storage records also matter. Replacing `API_KEY` in place destroys useful evidence about which version handled an attempt and makes rollback ambiguous. Store versioned references such as `provider/primary/v7` and `provider/standby/v8`; expose only a logical selection to the application. Logs should contain the version label, never the secret value.

The second boundary is request admission. Do not swap a process-wide variable while requests are in flight. Resolve a credential snapshot once for each job, then use that snapshot for every network attempt belonging to the job. New jobs can move to the standby after the cutover record commits. Existing jobs either finish with their captured version or enter an explicit retry path with the same billing event ID.

This sounds fussy. It is cheaper than reconciling invoices by hand.

Keep that boundary.

## A minimal Node.js cutover pattern

The useful abstraction is tiny: a credential resolver and a usage ledger. The resolver returns secret material plus a non-secret version; the ledger owns idempotency. This TypeScript sketch leaves secret storage and the database behind interfaces so the control flow stays visible.

```ts
type Credential = { version: string; value: string };
type UsageEvent = {
  eventId: string;
  tenantId: string;
  meter: "lesson_minutes";
  quantity: number;
};

interface CredentialStore {
  current(): Promise<Credential>;
}

interface UsageLedger {
  reserve(event: UsageEvent): Promise<"new" | "duplicate">;
  recordAttempt(eventId: string, credentialVersion: string): Promise<void>;
  commit(eventId: string): Promise<void>;
}

async function sendMeteredRequest(
  event: UsageEvent,
  credentials: CredentialStore,
  ledger: UsageLedger,
  callUpstream: (credential: string, event: UsageEvent) => Promise<void>,
): Promise<void> {
  const reservation = await ledger.reserve(event);
  if (reservation === "duplicate") return;

  // Capture once. A concurrent cutover cannot change this attempt's credential.
  const credential = await credentials.current();
  await ledger.recordAttempt(event.eventId, credential.version);
  await callUpstream(credential.value, event);
  await ledger.commit(event.eventId);
}
```

The database implementation should enforce uniqueness on `eventId`; an in-memory set is not enough across restarts or multiple nodes. The reservation also needs a recoverable state model, because a crash can occur after the external call and before `commit`. Whether that uncertain event may be retried depends on the upstream operation's idempotency semantics. Do not infer success from a timeout.

The code deliberately does not rotate secrets, activate keys, or mutate configuration. Those are control-plane actions. Keeping them out of the request handler makes authorization and auditing easier to reason about, while the handler stays boring enough to test.

A focused rehearsal can use a fake upstream. Start a job with the primary version, change the resolver to the standby before releasing the fake response, and assert that the first job records the primary while the next job records the standby. Then submit the same event twice and assert that the ledger contains one committed charge. No real billable request is needed.

## Rehearse evidence, not secret exposure

A rotation drill should prove four things: the standby can be activated, the workload can read only the selected secret, new work moves at the chosen boundary, and duplicate usage is rejected. Capture timestamps, approver identity, credential version labels, and event IDs. Never capture credential values. OWASP warns against logging secrets and recommends accounting for rotation, revocation, expiration, and audit needs throughout the secret lifecycle.

Run the drill on a schedule tied to operational change, not a vague promise. A deployment that changes secret access, a new worker pool, or an upstream account-policy change is a reason to repeat it. The exact interval is a risk decision; the evidence is what tells you whether the runbook still matches reality.

Failure handling deserves its own assertions. An authentication rejection should stop automatic retries with that credential and open the controlled cutover path. A timeout is ambiguous and should preserve the event for reconciliation. A rate-limit response is not proof that the credential is invalid, so rotating keys in response can multiply traffic without fixing capacity. Keep those branches distinct in metrics and alerts.

Watch counts of authentication failures by non-secret credential version, reserved-but-uncommitted events, duplicate reservations, and cutover duration. Those signals connect the security event to invoice correctness. They also avoid the tempting but dangerous practice of putting raw authorization headers into debug logs.

## When is the runner-up better?

Creating the spare during an incident is reasonable when credentials are short-lived, issuance is automated through an independently available control plane, and a tested recovery identity can mint a replacement without relying on the failed workload. In that model, there is no dormant long-lived secret to protect. The trade is dependency risk: issuance, authorization, and secret delivery must all work during the incident.

The limitation of the disabled-spare design is its extra control-plane dependency. It is not suitable when the provider cannot enable a credential through a tested recovery path, when activation requires the same unavailable identity as normal traffic, or when policy forbids dormant credentials. In those cases, choose just-in-time issuance if its independent recovery chain has been rehearsed. The downside is more moving parts during the incident; the benefit is avoiding a stored spare.

An already-active spare fits systems that cannot represent a disabled credential or where activation latency exceeds the recovery objective. Treat it as production access from day one. It needs the same scope restrictions, storage controls, audit trail, and revocation plan as the primary, plus detection for any use before an approved cutover.

Neither runner-up changes the billing rule. Tenant and event identity remain independent of the credential, every retry carries the original event ID, and the ledger decides whether usage is chargeable. **Continuity is complete only when service access and invoice attribution recover together.**

## References

- OWASP, "Secrets Management Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
