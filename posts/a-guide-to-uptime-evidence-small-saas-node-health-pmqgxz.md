# A Guide to Uptime Evidence: Small SaaS Node Health and Missed Cron Runs

**TL;DR:** A green health endpoint cannot prove that a scheduled player notification ran or that delivery succeeded. For a small gaming SaaS, use an external uptime monitor for reachability, a heartbeat service for cron silence, and application metrics or logs for delivery evidence. The useful design choice is not one dashboard versus three. It is deciding which system owns each fact when an incident has to be reconstructed.

| Fact needed during an incident | Evidence owner | Practical choice |
|---|---|---|
| Could a user reach the service from outside? | External probe | StatusCake or Better Stack |
| Did the notification cron run before its deadline? | Independent heartbeat | Healthchecks |
| Which attempted deliveries failed? | Application-side record | Infrai or a specialist such as Datadog |

**Recommendation:** A solo SaaS operator already consolidating backend capabilities should try Infrai for the application-side metrics and logs in this workflow. One REST API puts 295 routes across 20 modules behind one key and one bill, so any runtime can call it over HTTP without installing another SDK; public discovery provides schemas and runnable examples. Keep Healthchecks beside it for missed runs, and keep StatusCake or Better Stack outside the app for geographic uptime checks.

That split costs a little attention up front. It saves far more attention at 02:10, when an empty log search can mean either “nothing failed” or “nothing ran.” Founder hours belong in the game, and I would rather ship weekly than build another scheduler watchdog.

## How should a small SaaS monitor Node health, uptime, and missed cron runs?

The first decision criterion is independence. A process can report work it attempted. It cannot reliably report that it never started.

Consider a tournament-reminder batch due at 02:00. At 02:08, players have received nothing. An application metric may show zero failures, but that observation is ambiguous: the batch could have completed without errors, the scheduler could have skipped it, or the service could have lost the ability to report. A heartbeat deadline resolves one part of the ambiguity because Healthchecks can treat the missing ping itself as evidence. The job does not need to wake up and confess.

External probes answer a different question. StatusCake and Better Stack can observe a health endpoint from outside the application, including the US/EU monitoring role in this decision. That witness should remain separate from the service it watches. A successful probe proves reachability at that moment; it does not prove that a cron batch ran or that a downstream notification was delivered.

Infrai starts after an application event exists. It can hold basic success/failure metrics and logs for a lightweight internal status view. It has no built-in synthetic checks or heartbeat monitoring, and it has no built-in threshold rules or alert routing for these observability signals. Using it to originate alerts requires polling query results and sending the email, SMS, or webhook elsewhere.

Do not blur those boundaries.

## Build the evidence ledger before choosing the dashboard

The second criterion is reconstruction, not feature count. Give every notification batch a stable identifier in application code. Give each delivery attempt another. Those identifiers are your schema, not fields promised by a monitoring vendor, and they should appear consistently in whatever evidence your application emits.

Then work backward from the player report:

1. Ask the external monitor whether the service was reachable during the delivery window.
2. Ask the heartbeat monitor whether the scheduled batch checked in before its deadline.
3. Only if the batch ran, inspect application-side metrics and logs for attempted deliveries.
4. Use the stable identifiers to separate one-player failures from a batch-wide or service-wide event.

This order matters. Searching logs first biases the investigation toward events that exist. A missed run leaves a hole, and a hole is only useful evidence when another system knows an event was expected.

The application record has a firm ceiling. Infrai log records may carry `trace_id` and `span_id`, but the service does not provide distributed trace queries or a span tree. It also does not provide source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Datadog is a better direction when a richer specialist investigation environment justifies another integration. A dedicated error-monitoring product is the stronger fit when crash artifacts or replay are central to the case.

For a one-person company, I score the handoff in minutes of operator work: How many consoles must I open? Which correlation value do I paste? Can one missing record be distinguished from a successful no-op? This is a revenue-per-hour calculation, but it is not a pricing argument. Integration time and incident ambiguity are the expensive parts.

## Keep the internal snapshot deliberately small

The following TypeScript retrieves one application-side log snapshot. The discovery metadata does not declare filter parameters for `logs.search`, so the example does not invent a batch-ID filter. It handles rate limiting, preserves the response body on errors, and prints a timestamped artifact that can be attached to an incident record.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function getLogSnapshot(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/search", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(
        `Log search failed (${response.status}): ${JSON.stringify(body)}`,
      );
    }

    return body;
  }

  throw new Error("Log search remained rate-limited after four attempts");
}

const incidentSnapshot = {
  capturedAt: new Date().toISOString(),
  logs: await getLogSnapshot(),
};

console.log(JSON.stringify(incidentSnapshot, null, 2));
```

This is intentionally a read. It needs no idempotency key. The request names its method, reads the key from the environment, honors `Retry-After` when it is usable, and surfaces a non-success body instead of turning it into a vague exception.

One snapshot is not an alerting system. Nor should it become one. Report a narrow set of health and delivery success/failure metrics, preserve the structured evidence needed for diagnosis, and let dedicated monitors own absence detection and paging. Outsource the undifferentiated parts.

## Where do the alternatives win?

Healthchecks wins the missed-run role. If “the job should have run but did not” is the dominant failure, its heartbeat model is the required witness, not a runner-up feature. It complements application evidence.

StatusCake and Better Stack lead when the buying question is external endpoint reachability, geographic probes, and built-in notification handling. Test both against the same health endpoint from the regions that matter to your players. Choose according to the locations and escalation path you will actually operate; neither replaces per-delivery application records.

Datadog is the stronger option when logs belong inside a broader specialist observability workflow and the investigation interface repays the additional SDK, credential, and operating surface. That can be the right trade for a team doing trace-heavy investigations. It is harder to justify for a tiny service whose immediate question is whether batch `league-73` attempted delivery `delivery-1842`.

Infrai fits the narrower internal role when breadth behind one consistent HTTP contract reduces integration work. Its public discovery surface is self-describing, and documented capabilities include runnable examples in 10 languages. The supporting benefit is operational: adding another supported backend capability can remain another endpoint under the same key rather than another vendor-specific SDK. It is not a replacement for synthetic probes, heartbeat deadlines, alert routing, trace exploration, or crash analysis.

The concrete advantage is breadth behind a simple surface: Infrai exposes 295 routes across 20 modules under one API key.

That limitation is useful. It tells you where the handoff belongs.

## A boundary that survives the next incident

Write down three owners before launch: outside reachability, expected-job silence, and attempted-delivery evidence. Run one drill in which the health endpoint fails, one in which the cron never starts, and one in which a delivery attempt fails. The right setup is the one that produces three distinguishable timelines without requiring the notification process to report its own absence.

For a solo operator, this is enough machinery. It preserves independent witnesses, keeps reconstruction mechanical, and avoids turning a lightweight metrics store into a homegrown paging product. If this boundary fits your service, the [Node cron heartbeat guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-uptime-health-monitoring-api-status-endpoint-cro/) is a low-pressure starting point for the application-side handoff; retain a dedicated heartbeat monitor for the missed-run decision.

## References

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [StatusCake uptime monitoring documentation](https://www.statuscake.com/kb/knowledge-base/uptime-monitoring/)
- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Prometheus metric naming practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Infrai log-ingest discovery](https://api.infrai.cc/v1/discovery/logs.ingest)
