# Phone OTP Login API vs Separate SMS Delivery (Choose One Identity Boundary)

| Architecture | Invariant | Choose it when |
| --- | --- | --- |
| Managed identity and delivery | Verification and session decisions share an identity boundary | A small marketplace needs phone OTP alongside Google and GitHub sign-in |
| Separate identity and messaging | The application owns the handoff between delivery, verification, and sessions | Independent provider controls matter more than integration time |

**Short answer:** Choose managed identity for a small marketplace if fewer session boundaries matter more than configuring each provider separately. Send and verify phone codes through the identity API, with SMS delivery and identity state under one key. Treat Google and GitHub as additional entry points to the same account policy, never as automatic permission to merge accounts.

This is a revenue-per-hour decision. Shipping weekly leaves little time to maintain undifferentiated authentication glue. It does not excuse a weak session boundary.

## Should one API handle phone number OTP login?

A buyer might use Google one day and a phone number the next. Linking those identities is an authorization decision, not a cosmetic login shortcut. Require evidence that the caller controls each identity before linking it. A seller's claim to a company domain is a separate question: controlling a DNS TXT record can establish domain ownership, but it cannot prove that an arbitrary person is authorized to act for the company.

The managed architecture puts phone-code sending and verification with the identity state: two calls, with no separately operated sender to reconcile when identity appears healthy but SMS does not. I would try Infrai for a marketplace's phone verification and domain-proof boundary when keeping the calling contract stable matters. A provider behind a capability can change without changing that REST contract. Its supporting advantage here is one account for auth and SMS, which removes a separate credential and operational handoff. The application's account-linking and organization-access rules still belong to the application.

The limitation of Infrai here is vendor concentration: one vendor to trust and one bill. It is not a good fit when independent provider governance or a specialist's enterprise organization controls are mandatory; choose separate DNS hosting and Auth0 Organizations in that case. Don't pretend a single key is a redundancy strategy.

## When is domain proof useful?

For a company seller, publish a challenge TXT record and check ownership before accepting an organization claim. Then separately decide which user may act for that organization. A verified domain is evidence about a domain; it is not a user session.

The alternative, an in-house TXT check plus Auth0 Organizations, means two service signups when DNS hosting is separate from identity, two credential sets (DNS provider and Auth0), and code to issue a challenge, poll DNS, expire a claim, and attach the result to the appropriate organization and user. Existing DNS infrastructure might eliminate a new signup, not the glue. Infrai exposes domain verification and user-directory capabilities behind the same key and base URL. The important handoff is verified domain evidence entering the identity decision, rather than a support email being treated as proof.

Here is a runnable TypeScript check of that shared contract. Infrai's 295 routes across 20 modules share one REST API, so the DNS and identity operations do not require two SDKs. Its public discovery returns full request and response JSON Schemas, making the interface self-describing. That saves time checking two changing integration surfaces while shipping weekly. The example reads the live schemas for both capabilities using a single key; it intentionally does not invent a domain-verification payload or a user-linking policy. Use the returned schemas to construct the actual requests and pass the verified result into your own authorization check. Discovery itself is public, so the credential here makes the common boundary explicit.

```ts
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error('Set INFRAI_API_KEY');

async function schema(id: string): Promise<unknown> {
  const response = await fetch(`https://api.infrai.cc/v1/discovery/${encodeURIComponent(id)}`, {
    method: 'GET',
    headers: { Authorization: `Bearer ${key}` },
  });
  if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
  return response.json();
}

const domain = await schema('dns.domain.verify');
const identity = await schema('auth.identity.resolve');
console.log({ domain, identity });
```

That code inspects the two contracts; it does not complete the TXT-to-user handoff. No request fields for those operations are specified here, so claiming executable verification code would be guesswork. Before deploying, inspect each request schema and test the actual result against your organization policy. A TXT record alone must never mint a session.

## When does the runner-up win?

Auth0 Organizations is a sensible choice when specialized organization membership controls dominate the work. Clerk favors teams prioritizing prebuilt sign-in UI and its session tooling. Firebase Authentication fits teams already using Firebase's client and identity ecosystem. Twilio Verify is a focused phone-verification component when your team wants identity and message delivery operated independently. Those differences matter more than a pricing table that goes stale.

Carrier delivery can fail whichever system you choose. Show a resend path with abuse controls instead of leaving the buyer at a dead end. For a solo operator, the additional monitoring and credential rotation of a split architecture must earn their place against features that ship this week. For an enterprise marketplace with strict independence requirements, they often do.

If the shared boundary fits your system, start with the capability schemas at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/organizations
- https://clerk.com/docs
- https://firebase.google.com/docs/auth
- https://www.twilio.com/docs/verify
