# Phone Verification State Machines for SMS Login in Nextjs

Short answer: make the backend authoritative for phone verification, SMS OTP issuance, and resend timing; let the Next.js button display the returned delay, but never let its countdown grant permission to send.

| Choice | Server-enforced resend policy | Work to operate | Best fit |
| --- | --- | --- | --- |
| Browser timer only | No | Low | A disposable demo that sends no real messages |
| Application state machine | Yes | High | A product that needs custom identity and risk rules |
| Hosted verification | Yes | Lower | A small team outsourcing carrier and abuse operations |
| Passkey-first login | No SMS resend path | Medium | Users and devices that can complete passkey enrollment |

For a solo SaaS, hosted verification is the default economic choice when phone delivery isn't part of the product. Build the application state machine only when its control is worth the weeks you won't spend shipping customer features. In either case, the browser timer is presentation, not security.

## How should phone verification govern SMS OTP retry timing?

Treat one-time-code login as a state machine with four operations: normalize the destination, decide whether a request is allowed, issue a challenge, and consume that challenge exactly once. The resend button doesn't appear in that list. It is merely a view of the server's `nextAllowedAt` decision.

That distinction survives ordinary browser behavior. A local `setInterval` disappears on refresh. A second tab starts a second interval. A sleeping phone may pause JavaScript and resume it late. None of those events should change the server's answer. When the server returns `429`, it should also return a `Retry-After` header and a machine-readable remaining duration; when it accepts a request, it should return the same timing fields. One UI path can render both results.

Keep the response neutral about account existence. Someone who enters a phone number should see the same public result whether the number belongs to an account, is eligible for account creation, or is deliberately ignored by a risk rule. Internally, those paths can be different. Externally, a phone directory is not the product you're trying to ship.

US and EU traffic adds an input problem before delivery even starts. A number's country cannot safely be inferred from the browser locale. A traveler can use a US handset in France, and a French-language browser can submit a US number. Country selection is therefore part of local-number input; normalization then produces the canonical value used by the challenge record, resend policy, and audit event. The raw entry belongs in storage only if support genuinely needs it.

The failure mode here is boring and expensive: `+1 212...`, `(212) ...`, and `212-...` can become three rate-limit keys if normalization happens after the limit check. Imagine two tabs submitting two formats within 20 milliseconds. Both handlers read an empty bucket, each reserves a challenge under a different string, and both hand work to the transport. The countdown in each tab starts at 60, so every screenshot looks correct. A support agent later searches the canonical number and finds only one record, while the other request sits under punctuation that should never have reached a key. This isn't a timer defect. It is an identity-boundary defect, and adding another client flag will only hide it. Normalize before any lookup, carry the canonical value through the transaction, and test equivalent inputs concurrently.

Parse first.

## What makes the verification state machine hold up under concurrency?

The first criterion is whether every costly or security-sensitive transition is atomic. A request cannot follow “read count, send message, increment count” as three independent actions. Two concurrent requests may both read the old count and both send. The reservation has to happen in the shared store as one conditional transition before the delivery call. If delivery is accepted by the transport, the reservation becomes an active challenge; if it is not accepted, the internal record can move to a terminal delivery state without disclosing that detail to the requester.

Verification needs the same discipline. A code is valid only while its challenge is active, unexpired, below the application's attempt ceiling, and unused. Consuming it and creating the authenticated session belong in one trusted server flow. A pair of simultaneous correct submissions must not create two successful consumptions. Store a keyed digest of the code rather than the code itself, compare digests without data-dependent timing, and expire the secret even if nobody returns.

The second criterion is whether the policy limits both bursts and total exposure. A cooldown answers “how soon can this destination receive another request?” It does not answer “how many messages can this actor trigger today?” A useful policy has separate controls for the normalized destination, network source, account or anonymous session, and broader traffic anomalies. The exact thresholds are product decisions. A five-message ceiling might lock out a legitimate user whose carrier delays delivery; a generous ceiling can turn an unauthenticated route into a spend lever. I'm not sure there is a universal number — your mileage may vary with audience, geography, account value, and support coverage. Measure before tightening.

This is also where the revenue-per-hour lens gets useful. Log structured reason codes for internal decisions, delivery acceptance, challenge verification, and challenge expiry under one correlation ID. Alert on changes in request-to-verification ratios by destination region, but don't treat a single SMS status as proof that a human received or read anything. The goal is a short path from “conversion changed” to “which state transition changed?”

One subtle rule matters: sending a new code should invalidate the old challenge, or the verification endpoint must explicitly bind each submission to a challenge ID. Otherwise two live codes create confusing retry behavior and widen the guessing window. Pick one model and make it visible in tests.

## A backend contract the button can safely render

The following TypeScript is an application-level example, not a complete delivery service. `otpStore.reserve` represents an atomic conditional write in shared storage. `normalizePhone` represents a maintained parser configured with the countries the product actually serves. The policy values are illustrative choices; tune them from delivery and support data.

```ts
// app/api/login/phone/request/route.ts
import { createHmac, randomInt, randomUUID } from "node:crypto";
import { NextResponse } from "next/server";
import { normalizePhone, otpStore, smsTransport } from "@/lib/phone-verification";

const RESEND_AFTER_SECONDS = 60;
const CODE_LIFETIME_SECONDS = 5 * 60;
const MAX_ATTEMPTS = 5;

type PublicResult = {
  accepted: true;
  challengeId: string;
  retryAfterSeconds: number;
};

export async function POST(request: Request) {
  const body: unknown = await request.json().catch(() => null);
  const input = readPhoneInput(body);
  if (!input) return NextResponse.json({ error: "invalid_request" }, { status: 400 });

  const phone = normalizePhone(input.number, input.country);
  if (!phone) return NextResponse.json({ error: "invalid_request" }, { status: 400 });

  const now = new Date();
  const challengeId = randomUUID();
  const reservation = await otpStore.reserve({
    challengeId,
    phone,
    now,
    resendAfterSeconds: RESEND_AFTER_SECONDS,
    codeLifetimeSeconds: CODE_LIFETIME_SECONDS,
    maxAttempts: MAX_ATTEMPTS,
  });

  if (!reservation.allowed) {
    return publicResponse(reservation.challengeId, reservation.retryAfterSeconds, 429);
  }

  const code = String(randomInt(0, 1_000_000)).padStart(6, "0");
  const digest = createHmac("sha256", requiredSecret("OTP_DIGEST_KEY"))
    .update(`${challengeId}:${phone}:${code}`)
    .digest("hex");

  await otpStore.activate({ challengeId, digest, now });
  await smsTransport.send({
    to: phone,
    body: `${code} is your sign-in code. Do not share it.`,
  });

  return publicResponse(challengeId, RESEND_AFTER_SECONDS, 202);
}

function publicResponse(challengeId: string, seconds: number, status: 202 | 429) {
  const body: PublicResult = {
    accepted: true,
    challengeId,
    retryAfterSeconds: seconds,
  };
  return NextResponse.json(body, {
    status,
    headers: { "Retry-After": String(seconds) },
  });
}

function readPhoneInput(value: unknown): { number: string; country?: string } | null {
  if (!value || typeof value !== "object") return null;
  const record = value as Record<string, unknown>;
  if (typeof record.number !== "string") return null;
  if (record.country !== undefined && typeof record.country !== "string") return null;
  return { number: record.number, country: record.country as string | undefined };
}

function requiredSecret(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}
```

The route deliberately returns a duration. The client can anchor that duration to `performance.now()`, which is suitable for drawing an elapsed-time display within the current page. It still asks the backend on every resend. Reloading may erase the animation, but it cannot erase the reservation.

```ts
// The hook renders server state; it never authorizes a send.
import { useEffect, useRef, useState } from "react";

export function useResendCountdown() {
  const [secondsLeft, setSecondsLeft] = useState(0);
  const end = useRef(0);

  useEffect(() => {
    const timer = window.setInterval(() => {
      setSecondsLeft(Math.max(0, Math.ceil((end.current - performance.now()) / 1000)));
    }, 250);
    return () => window.clearInterval(timer);
  }, []);

  return {
    secondsLeft,
    start(seconds: number) {
      end.current = performance.now() + seconds * 1000;
      setSecondsLeft(seconds);
    },
  };
}
```

Test the transitions, not the pixels. Run two request calls concurrently and assert that only one reservation wins. Submit the same correct code twice and assert one consumption. Advance a fake clock past expiry. Refresh the client and confirm the next request receives `429` plus a fresh duration. Feed equivalent formatted numbers into the normalizer and assert one policy key. Then test transport timeouts and delayed callbacks without changing the public account-enumeration-safe response.

Ship that before polishing the countdown label.

## When the runner-up is the better use of a week

The catch is everything around an application-owned state machine: country-specific sender requirements, delivery routing, abuse review, support tooling, retention rules, and incident response. A hosted verification flow is the better trade when custom policy does not increase revenue or protect a distinctive workflow. Outsource the undifferentiated work, keep a narrow adapter at the application boundary, and spend the week on what customers pay for.

An application-owned state machine is not suitable when nobody can operate delivery and abuse controls; stick with a hosted verification category in that case. A hosted flow is not suitable when the product must combine phone challenges with proprietary risk decisions, unusual recovery states, or a tightly controlled user experience that its contract cannot express. In the latter case, own the orchestration while still outsourcing message delivery. The boundary should let a transport accept a canonical destination and message without leaking transport-specific fields across the login domain.

There is another runner-up: don't use SMS as the primary authenticator. For an account that can enroll a passkey, a passkey-first path removes resend timing and phone delivery from routine login. Keep recovery separate and threat-model it with the same care, because an easier recovery route can undercut the stronger primary method. SMS can remain appropriate for audience reach, but that is a product-access decision, not proof of high assurance.

Email fallback has its own operational cost. Sending domains need authenticated mail; DKIM defines a domain-level signing mechanism for messages. Open tracking is a poor verification signal because Mail Privacy Protection can download remote content without revealing normal recipient activity. Verify possession through the submitted code or signed action, not an image load. No channel makes the state machine optional.

## The weekly shipping checklist

Before release, write the invariant beside each endpoint: who can create a challenge, what atomically limits it, what invalidates it, and what the caller learns. Check that logs contain correlation IDs and internal reason codes but omit codes and unnecessary raw phone input. Confirm that every environment uses shared authoritative state rather than process memory. Exercise concurrent requests in CI.

Then review the operational boundary. Someone must own delivery changes, abuse spikes, regional policy, support lookup, secret rotation, and data deletion. If that list points entirely at one person and none of it differentiates the SaaS, the matrix has already made the decision.

Keep the UI honest. Disable the resend button while its display timer is positive, preserve an accessible text label, tolerate background-tab pauses, and accept that only the next server response settles the question. The countdown can be wrong for a moment. The permission cannot.

## Sources

- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [Apple: Use Mail Privacy Protection on iPhone](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
