# Registrar Exit Scheduling for Tenant Subdomain CNAME Records and Apex Addresses

Moving a B2B SaaS zone off a registrar-specific API puts two clocks on tenant subdomain provisioning: the time to publish a CNAME or address record and the time for cached DNS answers to expire. Publishing faster doesn't retire those cached answers faster. That constraint determines the cutover schedule.

Short answer: use a CNAME for a tenant subdomain when a stable hostname should control its destination. Use A records when the name must directly publish IPv4 addresses, including an ordinary zone apex where a CNAME cannot coexist with the required SOA and NS records. During the registrar exit, preserve the existing answers first. Schedule a record-type change separately unless there is a concrete reason to combine the risks.

The working example below is a proposed migration, not a production incident: a SaaS serves a customer at `workspace.customer.example`, while another customer uses the apex `customer.example`. The job is to replace the registrar-bound provisioning adapter without making either customer coordinate a second DNS edit during the same release. The useful measure is revenue per hour of engineering attention. A clever migration that consumes the weekly feature release has a real opportunity cost.

## Why does a quick record write still need a slow cutover?

DNS caching makes a successful write an incomplete observation. RFC 1034 describes TTL-based caching; RFC 2308 specifies negative caching, which matters when a resolver has already received a negative response for a name being provisioned. A newly created tenant can therefore encounter cached absence even after the authoritative server has the intended record. Repeating the write does not remove that cache entry.

Consider an illustrative existing TTL of 3,600 seconds and a proposed migration TTL of 300 seconds. Lowering the TTL does not rewrite an answer already cached for 3,600 seconds. Allow the previous lifetime to elapse after the change is consistently served before relying on the shorter value. These numbers are planning inputs, not measured propagation times or a promise that every resolver will refresh simultaneously. There is another clock if the exit also replaces authoritative nameservers: cached delegation data can keep queries reaching the old service. An answer TTL and a delegation TTL describe different data. A migration that merely replaces the control API while keeping the same authoritative service has a different scope from a migration that changes the parent delegation. Establish which job is actually being shipped before choosing a cutover window. In the example, shortening the TTL on `workspace.customer.example` changes the caching instructions for that answer. It does not shorten the lifetime of a previously cached delegation for `customer.example`. Treat those as separate entries in the migration schedule, with separate evidence that the intended configuration is being served.

Wait before switching.

This is the constraint that changes the build. The smallest useful release replaces the API dependency and preserves record intent. Changing CNAMEs into address records at the same time creates another reason for two resolvers to see different destinations. Keep the old authoritative service serving compatible answers through the planned overlap when delegation changes, and account for parent delegation caching when determining that overlap.

## Should a tenant subdomain use a CNAME record when the apex cannot?

A CNAME makes its owner an alias for another DNS name. An A record publishes an IPv4 address. RFC 1034 defines the distinction, and RFC 2181 clarifies the restrictions on data accompanying a CNAME. For a tenant subdomain, an alias to a stable service hostname lets the service operator change that target's address records without asking the customer to replace their alias. The target still needs valid answers, and the alias adds another dependency to the lookup.

For the example customer name `workspace.customer.example`, a CNAME to `ingress.platform.example` is a reasonable desired state if the service controls that target. The choice does not establish that this customer is authorized to bind the hostname, configure application routing, or provision a certificate. Those are separate activation checks. DNS answers a naming question; the application still has to accept the intended tenant and hostname.

The apex is different. This limitation is structural: an ordinary CNAME is not suitable for the zone apex, regardless of how quickly the provisioning API accepts writes.

At `customer.example`, the zone already requires SOA and NS data. An ordinary CNAME cannot share that owner with those records. An A record avoids that conflict, but it requires an actual IPv4 destination whose operational lifecycle the service can support. IPv6 addresses use AAAA records. An address copied from a transient lookup is a poor substitute for an explicit contract about which addresses customers should publish.

An alias-like feature that synthesizes apex address answers may be an option, but its configuration and behavior must be evaluated separately from a standard CNAME. Do not assume a registrar's proprietary record object can be replayed unchanged through another API. The migration inventory needs the intended DNS behavior as well as the old configuration representation.

That is the trade-off: a standard subdomain CNAME keeps the destination indirect, while direct apex addresses require the service to manage the published address lifecycle. Neither choice removes caching delays.

Preserve unrelated records too. Mail policy is an easy omission when the migration export focuses only on web traffic. RFC 7489 specifies DMARC policy publication through TXT records at the relevant `_dmarc` name. A working tenant website does not establish that the rest of its zone survived the move.

## The smallest implementation keeps intent outside the adapter

The useful code boundary is a small record planner, followed by an adapter for the destination API. Store the tenant hostname, zone apex, routing mode, destination, and TTL as desired state. Keep registrar-specific identifiers inside the adapter. That gives the migration a reviewable input before any external write occurs.

This TypeScript example deliberately supports two cases: an ordinary subdomain alias and a caller-supplied IPv4 address. The 300-second TTL is illustrative. Names enter as ASCII DNS names; internationalized name conversion belongs before this boundary. The function checks the apex conflict and the selected address syntax. It does not apply changes or claim to validate an entire zone.

```ts
type Binding = {
  zoneApex: string;
  hostname: string;
  ttl: number;
  destination:
    | { kind: "alias"; target: string }
    | { kind: "ipv4"; address: string };
};

type PlannedRecord = {
  owner: string;
  type: "CNAME" | "A";
  value: string;
  ttl: number;
};

function dnsName(value: string): string {
  return value.toLowerCase().replace(/\.$/, "");
}

function planRecord(binding: Binding): PlannedRecord {
  const owner = dnsName(binding.hostname);
  const apex = dnsName(binding.zoneApex);
  if (owner !== apex && !owner.endsWith(`.${apex}`)) {
    throw new Error("Hostname is outside the selected zone");
  }
  if (!Number.isInteger(binding.ttl) ||
      binding.ttl < 0 || binding.ttl > 2147483647) {
    throw new Error("TTL is outside the supported range");
  }

  const destination = binding.destination;
  if (destination.kind === "alias") {
    if (owner === apex) {
      throw new Error("An ordinary apex cannot use CNAME");
    }
    const target = dnsName(destination.target);
    if (target === owner) throw new Error("Self-alias rejected");
    return { owner, type: "CNAME", value: target, ttl: binding.ttl };
  }

  const octets = destination.address.split(".");
  if (octets.length !== 4 || octets.some((part) =>
    !/^(0|[1-9]\d{0,2})$/.test(part) || Number(part) > 255
  )) {
    throw new Error("Expected a dotted-decimal IPv4 address");
  }
  return {
    owner, type: "A", value: destination.address, ttl: binding.ttl
  };
}

const tenantRecord = planRecord({
  zoneApex: "customer.example",
  hostname: "workspace.customer.example",
  ttl: 300,
  destination: { kind: "alias", target: "ingress.platform.example" }
});
```

Before applying that plan, inspect existing records at the owner. Reject incompatible CNAME coexistence and check the target chain for loops; the local self-alias guard catches only the simplest loop. The IPv4 syntax check also says nothing about whether the address belongs to the service. Authorization, destination validation, and comparison against existing state belong in the surrounding workflow.

Keep retries narrow. If a write times out, read the destination state before repeating it, because the request may have succeeded despite the missing response. Compare normalized record intent rather than assuming an unknown response means that nothing changed. Record the plan and the observed result so the next weekly release does not depend on remembering which tenant needed attention.

## Ship the adapter with evidence for each transition

A practical build log has three checkpoints: intended records, authoritative answers, and customer-visible service behavior. Capture the complete zone inventory before the move, reconcile the destination configuration against it, then query the new authoritative servers directly. Recursive queries provide a separate view of cached behavior. One successful recursive lookup cannot establish that every cache has refreshed.

Test the planner with an apex alias that must be rejected, a valid subdomain alias, an out-of-zone hostname, and malformed address input. Test the adapter against unknown write outcomes and duplicate retries. For the release itself, inspect a real tenant hostname through DNS resolution, TLS certificate validation, and application routing. A correct CNAME can still lead to a service that has not activated that hostname.

If the zone is signed, include DNSSEC in the migration plan. RFC 4035 describes validation through the chain of trust. A parent DS record and the replacement zone's keys must remain consistent with the chosen transition procedure; copying web records alone does not address that dependency. Choose and verify the signing transition before changing delegation.

Rollback has a cache lifetime too. Keep the previous service available for the planned overlap, preserve the last known desired state, and distinguish an API rollback from a delegation rollback. Reversing a nameserver edit does not instantly pull every resolver back. Make the release decision using observed answers and service checks, with the remaining cache windows written down.

## What I would change at scale

At larger tenant counts, the next investment would be a reconciler that compares desired and observed state, limits concurrent writes, and makes pending activation visible. Persist separate states for configuration published and hostname verified. A DNS write response should not silently become the application's readiness signal.

For a one-person SaaS, start with a reviewable plan and a deliberate overlap window. Ship weekly, outsource the undifferentiated authoritative serving, and retain control of tenant intent and verification. CNAME offers destination indirection for subdomains; A records make address responsibility explicit at names such as the apex. The cutover schedule should reflect cached data and the scope of the registrar exit, rather than the speed of the new API.

## Sources

References for DNS behavior, record constraints, caching, validation, and mail policy:

- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names — Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2181, Clarifications to the DNS Specification: https://datatracker.ietf.org/doc/html/rfc2181
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 4035, Protocol Modifications for the DNS Security Extensions: https://datatracker.ietf.org/doc/html/rfc4035
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
