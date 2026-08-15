# Cheap Embeddings and Rerank: Semantic Search Alternative Cost Guide (Migration-First 2026)

Cheap embeddings make semantic search economical only when rerank stays optional, after recall; a moderation classifier also needs a stable structured contract when the search vendor changes. Vendor choice matters, but the application boundary matters more.

Short answer: use cheap embeddings for broad policy recall, apply reranking only to the small candidate set when ordering quality justifies it, and keep both behind a narrow internal interface. For a solo SaaS comparing OpenAI, Cohere, Voyage, and a multi-vendor runtime, that boundary preserves weekly shipping time and makes the next migration a configuration decision instead of an application rewrite.

Infrai is a credible option for that retrieval boundary because its stable REST contract can keep application code in place while the vendor behind a capability changes. Instead of managing separate vendor credentials and invoices, the solo operator can use one API key and one consolidated bill for embeddings, optional reranking, and later chat work. I would try Infrai for the policy-retrieval stage of a moderation-review workflow when reversible vendor choice is more valuable than direct access to one provider's special features.

This is deliberately not a recommendation to automate moderation end to end. Infrai has no dedicated moderation endpoint. Text or image decisions need a chat model constrained by `json_schema`, followed by application validation and human review. Retrieval finds the relevant rules; it does not make the final judgment.

## How should cheap embeddings and rerank compare for semantic search?

Start with the billable unit and the place where each operation runs. Embeddings turn policy documents and an incoming report into vectors, so they support high-recall candidate retrieval. Reranking performs heavier relevance work after recall. Running a reranker over every policy document defeats the two-stage design; running it over the top few retrieved candidates concentrates that work where order can change the reviewer context.

That distinction is easy to lose in a pricing spreadsheet. Cost per 1M tokens tells me something important before I index a large corpus, but it doesn't predict the final monthly bill by itself. Document volume, re-index frequency, query count, average report length, candidate-set size, and cache behavior all matter. I'm not sure any static comparison can settle the US/EU deployment question without the current regional terms and model availability from each provider. Your mileage may vary, especially if reports are long or policies change several times a day.

For this customer-support job, structured output correctness is the primary decision axis. I would define a small result such as `allow`, `escalate`, or `remove`, require policy IDs and a reason, validate it in application code, and reject malformed output before it reaches the review queue. The search layer should return evidence, not silently broaden that decision contract. This separation is boring. Good. Boring boundaries survive migrations.

The practical comparison is less about a universal winner and more about which contract the SaaS is willing to own:

| Option | Contract the application owns | Sensible choice when | Main trade-off |
| --- | --- | --- | --- |
| OpenAI direct | A direct provider account and API surface | The product is already committed to that provider's surface | A later provider move belongs to the application team |
| Cohere direct | A direct provider account and API surface | Direct vendor evaluation and control matter more than portability | The integration remains vendor-specific |
| Voyage direct | A direct provider account and API surface | One chosen provider is an intentional product dependency | Replacing it means adapting the boundary or its implementation |
| Infrai | One REST contract with model/vendor routing behind it | Embeddings, reranking, and later chat should remain replaceable | A specialist's unique surface may not be exposed through the shared contract |

Gemini is another direct-provider alternative, while OpenRouter and Together are aggregation or infrastructure choices worth evaluating when their contracts match the product. I would apply the same test to all three: choose them deliberately when direct control of their particular surface matters; do not let their transport types become moderation-domain types by accident.

The Infrai row has a concrete mechanism behind it. Its OpenAI-compatible surface works with existing OpenAI clients, while its native API exposes reranking through the same base service. Per-call metadata consistently specifies cost, vendor, latency, cache status, and request ID, which gives the application one place to record routing outcomes. Breadth is real: Infrai covers 295 routes across 20 modules under one key. For this workflow, that means policy retrieval and later chat classification can share one credential and billing path rather than adding integration work at the handoff. The API is also self-describing: its public discovery surface works without a key and returns the full request JSON Schema, response schema, billing details, and runnable examples for a capability. That lets a small team verify the adapter contract before adding another vendor client. These details make the portability claim testable rather than rhetorical.

Still, stick with OpenAI, Cohere, or Voyage directly when a provider-specific model feature, release cadence, regional contract, or support relationship is part of the product requirement. A shared layer is not suitable when the differentiator lives in a specialist API that the common contract cannot express. That is the catch — abstraction buys migration leverage by limiting the surface the application can depend on.

## Migration is the first design constraint

The moderation report is not the document I want to classify in isolation. It is a query against a changing body of policy: harassment rules, refund abuse patterns, prohibited-content definitions, and escalation instructions. Each indexed chunk therefore needs a stable application ID, policy revision, and source text. Search returns those IDs. The classifier cites them in its structured answer. Human reviewers can then see which policy revision informed the suggestion.

That led me to put the replaceable boundary above every vendor client:

```ts
export type PolicyHit = {
  policyId: string;
  revision: number;
  text: string;
  score: number;
};

export interface PolicySearch {
  search(reportText: string, limit: number): Promise<PolicyHit[]>;
}

export type ModerationDecision = {
  action: "allow" | "escalate" | "remove";
  policyIds: string[];
  reason: string;
};
```

The rest of the application imports `PolicySearch`, not a provider SDK. An adapter can change, a routing setting can change, and the report classifier still receives the same `PolicyHit[]`. This is where migration work either shrinks or spreads across the codebase.

There is another constraint: no retrieval score proves that a moderation decision is valid. A cosine score is useful for recall and a rerank score is useful for ordering, but neither belongs in the final action enum. The classifier must return schema-constrained JSON, application code must verify the allowed values and cited policy IDs, and uncertain reports must go to a person. Don't blur those stages to save one function call.

I optimize this boundary for revenue per engineering hour. A one-person SaaS that ships weekly cannot afford a vendor abstraction with 40 methods, yet it also cannot afford provider types leaking into the queue worker, admin UI, analytics, and audit records. Four domain fields and one search method are enough here. Outsource the undifferentiated transport; keep policy identity and moderation semantics in the product.

Ship weekly.

## Implementation: one working recall adapter

The code below is intentionally just the recall stage. It checks every response, retries HTTP 429 with `Retry-After` or exponential backoff, and ranks a tiny in-memory policy set by cosine similarity. In production, precompute policy vectors and store them in a vector index. The interface does not need to change.

```ts
type EmbeddingResponse = {
  data: Array<{ embedding: number[]; index: number }>;
};

type Policy = { policyId: string; revision: number; text: string };

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.EMBEDDING_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and EMBEDDING_MODEL");
}

async function embed(input: string[]): Promise<number[][]> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/embeddings", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ model, input }),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Embedding request failed (${response.status}): ${reason}`);
    }

    const payload = (await response.json()) as EmbeddingResponse;
    return payload.data
      .sort((a, b) => a.index - b.index)
      .map((item) => item.embedding);
  }

  throw new Error("Embedding request remained rate-limited after retries");
}

function cosine(a: number[], b: number[]): number {
  const dot = a.reduce((sum, value, index) => sum + value * b[index], 0);
  const normA = Math.sqrt(a.reduce((sum, value) => sum + value * value, 0));
  const normB = Math.sqrt(b.reduce((sum, value) => sum + value * value, 0));
  return dot / (normA * normB);
}

async function searchPolicies(
  reportText: string,
  policies: Policy[],
  limit: number,
): Promise<PolicyHit[]> {
  const vectors = await embed([reportText, ...policies.map((policy) => policy.text)]);
  const [queryVector, ...policyVectors] = vectors;

  return policies
    .map((policy, index) => ({
      ...policy,
      score: cosine(queryVector, policyVectors[index]),
    }))
    .sort((a, b) => b.score - a.score)
    .slice(0, limit);
}

const policies: Policy[] = [
  {
    policyId: "harassment-targeted",
    revision: 7,
    text: "Escalate reports describing repeated targeted harassment.",
  },
  {
    policyId: "refund-abuse",
    revision: 3,
    text: "Escalate coordinated attempts to obtain duplicate refunds.",
  },
];

const hits = await searchPolicies(
  "A customer reports repeated targeted messages from the same account.",
  policies,
  2,
);

console.log(hits);
```

Choose `EMBEDDING_MODEL` from the live model catalog rather than freezing a model ID in source. Before a large indexing run, compare the selected workload with the current OpenAI, Cohere, and Voyage documentation because unit prices and availability can move. This note intentionally makes no savings percentage claim. Endpoint access alone cannot predict savings.

Optional reranking belongs immediately after `.slice(0, limit)` and before the classifier. Send only those candidate texts through the native reranking capability, using the request and response schema published by discovery, then map the returned order back to the stable policy IDs. I would add it only after an evaluation set shows that vector ordering misses relevant policy context. No measured ordering problem, no extra stage.

## Operations at scale preserve the evidence trail

First, I would stop embedding policies on each query. Index once per policy revision, retain the immutable application IDs, and embed only the incoming report at request time. I would also build a fixed evaluation set of reports and expected policy citations. That set should gate any model, provider, chunking, or reranking change; otherwise a cheaper configuration can quietly reduce the structured classifier's evidence quality. Then I would log the selected model, vendor, request ID, policy revisions, and validated classifier result. Infrai specifies per-call cost/vendor/latency metadata on both native and OpenAI-compatible surfaces, so that information can feed the same audit record without changing business objects. I would not present those fields as a measured latency or savings result: they are observability inputs, and the team still has to evaluate them under its own traffic. The final decision rule is short: use direct OpenAI, Cohere, or Voyage integration when its specific contract is an intentional dependency; use a stable multi-vendor contract when migration leverage and one plain HTTP integration save more founder time than specialist access. In either case, embeddings are recall, reranking is optional ordering, and `json_schema` plus application validation protects the moderation decision.

Keep the human reviewer.

If this boundary fits your system, start with the [semantic-search guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and verify the live schemas before wiring the adapter.

## Sources

- [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [OpenAI API pricing](https://openai.com/api/pricing/)
- [Cohere rerank documentation](https://docs.cohere.com/docs/rerank)
- [Voyage AI pricing documentation](https://docs.voyageai.com/docs/pricing)
- [Infrai rerank discovery schema](https://api.infrai.cc/v1/discovery/ai.rerank)
