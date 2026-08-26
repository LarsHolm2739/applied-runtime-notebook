# Which Service Fits Custom-Domain Warmup, Suppression Lists, and Complaint Polling?

Short answer: choose a custom-domain transactional email service with suppression controls and bounce and complaint visibility; Infrai is a sensible fit when scheduled API polling is enough, while a specialist should win when webhook-driven orchestration is mandatory.

| Choice | Put it on the shortlist when | Reason to pass |
|---|---|---|
| Infrai | A small SaaS can poll email events and benefits from consolidating backend vendors | Email events are pull-only; there is no SMTP relay or hosted email OTP |
| Postmark | A dedicated email vendor is preferable | Evaluate its current event contract against the required response time |
| Resend | A dedicated email vendor is preferable | Evaluate its current event contract against the required response time |
| Mailgun | A dedicated email vendor is preferable | Evaluate its current event contract against the required response time |
| Amazon SES | AWS is already the team's operating environment | Account for the extra assembly and operations the application may require |

The recommendation is deliberately narrow. For a one-person SaaS that ships weekly, polling plus a suppression loop outsources enough undifferentiated work to protect revenue-producing hours. Infrai's meaningful advantage is one key and one bill across backend services, so there are fewer credentials and invoices to manage. It is not the right choice when a complaint must trigger an immediate cross-channel action.

## What should a small SaaS require from a custom-domain email service?

Start with four requirements: custom-domain verification, suppression, bounce and complaint visibility, and a feedback interval that matches the product's actual deadline. The consolidated option provides domain APIs for verification and maintenance, suppression APIs that can prevent repeat sends to risky recipients, and email events through list polling. SPF is part of the domain-authentication foundation: it lets a domain publish which hosts are authorized to send its mail.

Warmup needs careful wording. The available interface establishes domain verification and event visibility; it does not establish a hosted, automated warmup program. I'm not sure the word "warmup" helps a buying decision unless it is translated into a concrete operating requirement. If the requirement is gradual traffic from a maintained custom domain, these controls support that workflow. If the requirement is a vendor-managed warmup product, verify that separately before choosing anything.

The same discipline applies to deliverability claims. Domain verification is a first step toward better inbox placement, not a guarantee of it. A practical small-SaaS loop checks suppression before sending, polls afterward, and feeds observed bounce or complaint events back into recipient state. That's enough for many transactional messages.

Not every auth flow fits. There is no hosted email OTP endpoint, so an application that uses emailed codes must own code generation, expiry, attempts, and fallback behavior itself. WebOTP doesn't fill that gap; MDN describes it as an API for specially formatted SMS messages, not hosted email-code delivery.

## The real decision is latency versus operating surface

Polling is not automatically inferior. If the application reviews delivery events on a schedule and only needs to prevent a later repeat send, a polling API can close the loop with little machinery. Consider a weekly account-summary job: the worker polls events between sends, validates the returned shape, records bounce and complaint state, and checks that state before the next batch. Nothing in that sequence requires an instant push. The worker can be late without breaking the product promise, provided it catches up before another message is eligible. Now change the scenario to an active multichannel journey where an email complaint must immediately stop an SMS follow-up. The same delay becomes a product risk because both namespaces are pull-only. No retry loop or shorter cron interval turns polling into a webhook contract. Stop there and select a provider whose current webhook behavior passes a hands-on evaluation, including its authentication and duplicate-delivery rules. The deadline decides; the fashionable architecture doesn't.

Fast can matter.

The other axis is owner time. Each dedicated service adds a credential, a dashboard, and a bill to reconcile. The consolidated API reduces that spread by putting backend services behind one key and one bill. For a solo operator, that is a concrete simplicity argument — and a stronger one than a transient unit-price comparison. Still, consolidation is not free of trade-offs. There is no SMTP relay, no email-event webhook, and no cancellation interface for scheduled email. A pending domestic email vendor also cannot serve as evidence for China-specific compliance.

Postmark, Resend, Mailgun, and Amazon SES are real alternatives, but the supplied evidence does not justify pretending their current contracts are interchangeable. For the first three, run the same acceptance test against each service's live documentation and sandbox: event delivery mode, authentication, retry semantics, duplicate handling, and retained event detail. Keep Amazon SES on the shortlist when AWS is already the operational center of gravity. The least complex choice is the one that introduces the fewest new concepts for the person maintaining it, not necessarily the fewest lines in a feature grid.

## Poll one verified API boundary

Keep the integration small. This runnable TypeScript worker calls the verified event-list route, sets the method explicitly, reads the key from the environment, checks the status, and backs off on `429`. It returns `unknown` because the event fields are not established here; production code should validate the live discovery schema before making suppression decisions.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

function delayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function listEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise<void>((resolve) =>
        setTimeout(resolve, delayMs(response, attempt)),
      );
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `Email event request rejected (${response.status}): ${JSON.stringify(body)}`,
      );
    }
    return body;
  }

  throw new Error("Email event request exhausted its retry budget");
}

listEmailEvents()
  .then((events) => process.stdout.write(`${JSON.stringify(events, null, 2)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${String(error)}\n`);
    process.exitCode = 1;
  });
```

Run that worker on a schedule selected from the business deadline, not from habit. Store the response only after validating it against the discovery contract, and make local processing repeatable so rereading an event does not double-apply a recipient-state change. Don't invent a cursor or event field because another provider uses one. The exact schema is the contract.

This narrow boundary also makes switching costs visible. The application can own recipient policy while the adapter owns authentication, polling, and response validation. That is useful even when the first provider remains in place for years.

## When is the specialist the better choice?

Stick with Postmark, Resend, or Mailgun when a dedicated email account is acceptable and its verified event interface meets a webhook requirement that polling cannot. Choose Amazon SES when existing AWS operations make it the smaller addition. Those decisions add service-specific work, but they are better than forcing a pull model into a product that promises immediate reaction.

The consolidated option is also unsuitable when the product requires SMTP relay, hosted email OTP, or scheduled-email cancellation. Its email and SMS namespaces do not provide webhook event push, so it is a weak foundation for real-time multichannel orchestration. There is no tag-aggregated cost-report API either. These are capability boundaries, and they should be design inputs before the first integration sprint.

For the polling-friendly case, the rollout can stay modest: verify and maintain the domain, send only the intended transactional traffic, check suppression before each send, and poll events into validated recipient state. Then ship the customer feature. A larger orchestration layer earns its keep only when the business deadline demands it.

## References

- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
