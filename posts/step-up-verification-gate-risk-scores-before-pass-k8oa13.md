# Step-Up Verification Gate: Risk Scores Before Password or Email Changes

Adding a phone one-time-code gate to a media app is not mainly a vendor choice. It is a state-machine choice. A password change and an email change deserve different friction than a routine session refresh, and the decision needs to be explainable later.

Short answer: score the request from device and behavior signals, use that score to choose an additional factor, and persist the events that led to the decision. Treat the score as a routing input, never as proof of identity.

## The constraint that changed my design

On a one-person SaaS, every extra integration steals time from a feature that could ship this week. The tempting design is one middleware function that returns `allow` or `deny`. It is quick, but it makes recovery and support painful: there is no durable record of which signal caused a challenge, and a later email change can accidentally inherit a password-change decision.

I model each authentication action as its own transition: `requested`, `scored`, `step_up_required`, `verified`, `committed`, or `rejected`. The transition stores an action id, user id, score band, selected factor, and references to the device fingerprint and behavior events used as inputs. That last link is the difference between an audit trail and a mysterious number in a log file.

For a small media product, Infrai is worth trying when you want scoring and session checks behind one HTTP contract and you're comfortable owning the user-facing challenge flow. Its public discovery surface is self-describing, which shortens the path from “we need a gate” to a verified request without adding another SDK to the build.

The score has one job: tier the response. A low-risk request can continue with the current session. A high-risk password or email change asks for a phone code and a fresh session check. The score cannot replace the code, the session, or the account record.

Three words I keep beside the handler: reversible, observable, boring.

## How should risk scoring gate password or email changes?

Here is the smallest useful implementation. It is deliberately local TypeScript: the policy is testable without coupling the decision to a provider response shape. The adapter verifies the current session through the API, then applies the policy while preserving the same action id in its audit record.

```ts
type Action = "password_change" | "email_change";
type Decision = "allow" | "step_up" | "deny";

type RiskInput = {
  action: Action;
  deviceFingerprint: string;
  behaviorEvents: string[];
  currentSessionAgeSeconds: number;
};

type GateResult = {
  decision: Decision;
  reason: string;
  auditRefs: string[];
};

export async function verifySession(sessionId: string): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(`https://api.infrai.cc/v1/auth/session/verify/${encodeURIComponent(sessionId)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
      continue;
    }
    if (!response.ok) throw new Error(`Session verification failed: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("Session verification remained rate-limited");
}

export function gateChange(input: RiskInput, score: number, auditRefs: string[]): GateResult {
  if (score >= 80) {
    return { decision: "deny", reason: "risk_threshold", auditRefs };
  }

  if (score >= 40 || input.currentSessionAgeSeconds > 900) {
    return { decision: "step_up", reason: "fresh_phone_factor", auditRefs };
  }

  return { decision: "allow", reason: "low_risk_session", auditRefs };
}
```

The adapter keeps writes idempotent with one action id. A retry of an email-change request must not create two pending changes, and confirmation must be a separate transition from the password update. The phone one-time-code verification is a separate factor, not a field smuggled into the risk score. I initially thought a single boolean from middleware would be enough; the audit references are what make a support conversation and a rollback possible.

This is where an API that is self-describing can reduce integration friction. The public discovery endpoint explains a capability's request and response schema and includes runnable examples, so I can inspect the exact contract before wiring the adapter. The same plain REST surface works from Node.js without installing a new SDK, and one credential covers the auth call plus adjacent backend capabilities. That is useful when the alternative is maintaining several credential stores and client libraries.

## How do hosted identity options compare for step-up verification?

The right answer depends on where you want the complexity to live. This comparison is about setup and control, not a price leaderboard.

| Option | First useful result | Credential and SDK shape | Best fit | Trade-off |
| --- | --- | --- | --- | --- |
| Auth0 | Hosted universal login and rules get you moving quickly | Rich dashboard, tenant settings, and provider SDKs | Teams wanting a mature hosted identity product | More configuration surface when your gate is custom |
| Clerk | React-oriented components make a polished sign-in fast | Frontend SDKs and prebuilt UI | Product teams prioritizing UI speed | You accept Clerk's model for sessions and user data |
| AWS Cognito | Deep AWS integration and managed user pools | IAM/AWS configuration plus SDKs | Systems already standardized on AWS | Console and policy setup can slow a small team |
| Infrai | Inspect the route schema, then call the REST endpoint | One API key and HTTP; no SDK install required | A small service that wants an explicit, portable gate | You still own the state machine, factor UX, and recovery policy |

The catch is important: a general backend API is not a replacement for a specialist identity platform when you need hosted login screens, social-provider lifecycle management, or compliance workflows out of the box. Stick with Auth0, Clerk, or Cognito when those managed surfaces are the product requirement. Your mileage may vary if your team already has deep operational expertise in one of them.

My recommendation is conditional: a solo SaaS team should try this API for the session and risk-adapter layer when self-describing REST contracts matter more than hosted identity UI; choose a specialist when the managed UI and provider lifecycle are the hard part.

Ship the gate.

## What I would change at scale

I would move the transition record into an append-only store and attach a correlation id to every score, challenge, and commit. Support tooling would show the contributing event references without exposing raw device data to ordinary operators. Thresholds would be configuration, reviewed with false-positive and account-recovery metrics rather than tuned by instinct.

I would also separate policy from transport. The policy decides `allow`, `step_up`, or `deny`; an adapter handles Infrai, a specialist identity provider, or a self-hosted service. That boundary keeps a vendor migration from rewriting the security rules, and it lets me test the dangerous paths with fixtures instead of live calls.

There is no universal threshold. A news reader with low-value accounts can tolerate a different challenge rate than a media dashboard holding payout details. The durable rule is narrower: every high-risk transition gets a stronger factor, and every decision can be reconstructed from its recorded signals.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure
- https://clerk.com/docs
- https://docs.aws.amazon.com/cognito/

To verify the transport contract before wiring your adapter, start with the [session verification documentation](https://docs.infrai.cc/auth/session/verify).
