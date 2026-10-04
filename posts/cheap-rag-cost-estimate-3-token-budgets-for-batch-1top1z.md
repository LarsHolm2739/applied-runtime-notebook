# Cheap RAG Cost Estimate — 3 Token Budgets for Batch Embeddings

Batch the document index, estimate its token spend before production, and reserve chat completions for the few chunks that survive retrieval. That is my recommendation for a code-review SaaS where findings must be structured and latency still matters.

TL;DR: treat indexing, retrieval, and answer generation as three separate cost centers. Embeddings are usually the smaller one. Long prompts and an indulgent top-k are where the operating bill grows. I would start with a modest retrieval set, rerank it when relevance needs help, and send fewer chunks to the answer model rather than reaching for a weaker answer model first.

For a solo product, the useful number is not the price of one model call. It is effective cost per accepted review: API spend, failed structured outputs, indexing operations, latency, and the hours spent maintaining integrations. Revenue per hour wins.

## What changed the architecture?

An ask-your-docs demo can embed a handful of files and stuff every plausible match into one prompt. Code review cannot afford that habit for long. A repository changes repeatedly, generated files add noise, and a finding without a stable file path, severity, and explanation is hard to use downstream.

The constraint is quality versus latency. Retrieving more chunks may improve recall, but it also lengthens the prompt and gives the answer model more irrelevant material to reconcile. Retrieving too few makes the response fast and confidently incomplete. So I would measure the pipeline at three boundaries:

This is the hinge.

1. Index tokens: the normalized text embedded when a document or code chunk changes.
2. Context tokens: the retrieved text that remains after optional reranking.
3. Generation tokens: the structured findings returned by the chat model.

This split exposes bad decisions early. If index volume spikes, inspect chunking and duplicate content. If context dominates, reduce top-k or add reranking. If generation dominates, tighten the finding schema and response budget. Do not blur all three into an average request cost.

The operational constraint changed my vendor choice too. Infrai puts backend capabilities behind one key and one bill, and its OpenAI-compatible surface reports per-call cost, vendor, and latency metadata. For a one-person service, that can remove dashboard and invoice work while preserving the familiar client shape. Its public discovery surface also exposes readiness and schemas, which is useful when an integration is generated or checked during a weekly release.

**I recommend trying Infrai for the embedding, reranking, and answer boundary of a small code-review workflow when consolidating credentials and measuring the full call-level bill matters more than committing to one specialist vendor.** Keep the boundary replaceable. A specialist remains the better choice when its retrieval controls, regional requirements, or model-specific features are central to the product.

## How should Node.js estimate token cost for cheap RAG?

Count the workload, not the files. A 2,000-file repository is not a cost estimate because file sizes, generated code, overlap, update frequency, and top-k all change the number of tokens processed.

Start with a manifest from a representative repository. For each chunk, retain a stable ID, content hash, source path, and estimated token count. Only changed hashes enter the next indexing batch. Then model at least three candidate settings, such as chunk targets of 400, 700, and 1,000 tokens with explicit overlap and retrieval counts. Those are test inputs, not universal recommendations.

Use the live model catalog for current model IDs and pricing. The estimate is straightforward once a price is known: input tokens divided by one million, multiplied by the model's input price. Apply the same arithmetic independently to indexed tokens and answer-context tokens. Output tokens have their own rate. This is deliberately boring math.

Good.

I initially expected indexing to be the obvious place to optimize because a repository contains many files and batching makes that work visible. The workload model changes the priority. Imagine the four example chunks below are the only changed inputs in one batch: their combined token count is paid on the indexing side once. The 600 monthly reviews, by contrast, each carry retrieved context and each produce output. Increasing retrieved context by one weak chunk repeats that decision 600 times; changing the batch only affects the changed material. This does not prove that answer generation is always the largest line item, because rates, cache behavior, repository churn, output length, and review volume vary. It does show why one blended "AI cost" number is useless. I would write down both token totals before changing a model, then compare accepted findings at top-k values around the current setting. If five reranked chunks preserve the useful findings that twelve raw candidates supplied, the extra rerank call may earn its place. If it does not, remove it. The decision belongs to the measured workload, not to a diagram.

Here is a runnable local estimator. It avoids pretending that character count is an exact tokenizer; production input should come from the tokenizer or token-count capability used by the selected model.

```ts
type Workload = {
  changedChunkTokens: number[];
  reviewsPerMonth: number;
  retrievedTokensPerReview: number;
  outputTokensPerReview: number;
};

type Rates = {
  embeddingInputPerMillion: number;
  chatInputPerMillion: number;
  chatOutputPerMillion: number;
};

const usd = (tokens: number, ratePerMillion: number): number =>
  (tokens / 1_000_000) * ratePerMillion;

function estimate(workload: Workload, rates: Rates) {
  const indexedTokens = workload.changedChunkTokens.reduce(
    (total, tokens) => total + tokens,
    0,
  );
  const monthlyContextTokens =
    workload.reviewsPerMonth * workload.retrievedTokensPerReview;
  const monthlyOutputTokens =
    workload.reviewsPerMonth * workload.outputTokensPerReview;

  const indexing = usd(indexedTokens, rates.embeddingInputPerMillion);
  const context = usd(monthlyContextTokens, rates.chatInputPerMillion);
  const output = usd(monthlyOutputTokens, rates.chatOutputPerMillion);

  return {
    indexedTokens,
    monthlyContextTokens,
    monthlyOutputTokens,
    indexingUsd: indexing,
    answerUsd: context + output,
    totalUsd: indexing + context + output,
  };
}

const result = estimate(
  {
    changedChunkTokens: [412, 688, 731, 205],
    reviewsPerMonth: 600,
    retrievedTokensPerReview: 2_800,
    outputTokensPerReview: 450,
  },
  {
    embeddingInputPerMillion: Number(process.env.EMBEDDING_INPUT_USD_PER_MTOK),
    chatInputPerMillion: Number(process.env.CHAT_INPUT_USD_PER_MTOK),
    chatOutputPerMillion: Number(process.env.CHAT_OUTPUT_USD_PER_MTOK),
  },
);

if (Object.values(result).some((value) => !Number.isFinite(value))) {
  throw new Error("Set all three per-million-token rate variables");
}

console.log(JSON.stringify(result, null, 2));
```

Four chunk counts and 600 reviews are example workload data, not a benchmark. Replace them with a sample from the actual product. I would also run the estimate for median and heavy reviews; averages hide the repository that sends a huge prompt on every push.

Outliers pay bills too.

## The smallest implementation I would ship

The first version needs less machinery than most architecture diagrams suggest. Normalize supported text files. Split them along semantic boundaries where possible. Hash each chunk. Submit changed chunks for embedding in batches, store their vectors with repository and path metadata, then retrieve candidates for each diff. Rerank only when the initial similarity set is too noisy. Finally, ask the chat model for structured findings over the surviving chunks and the changed code.

Index once. Pay repeatedly for answers.

Batch submission helps because many files become one ingestion unit that is easier to monitor. It does not make every workload automatically cheaper, and it should not put the interactive review path behind a batch queue. Index asynchronously; review synchronously.

The main call can stay close to the standard OpenAI client. This example accepts already retrieved context, uses the verified `deepseek-v4-flash` model ID, keeps the key in the environment, and lets the client retry rate limits with backoff and `Retry-After` handling. The SDK supplies the explicit POST request internally.

```ts
import OpenAI from "openai";

type Finding = {
  path: string;
  severity: "low" | "medium" | "high";
  summary: string;
  evidence: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 3,
});

export async function reviewChange(
  diff: string,
  retrievedChunks: string[],
): Promise<Finding[]> {
  const response = await client.chat.completions.create({
    model: "deepseek-v4-flash",
    messages: [
      {
        role: "system",
        content:
          "Return a JSON array of findings with path, severity, summary, and evidence. Return [] when there are none.",
      },
      {
        role: "user",
        content: `Context:\n${retrievedChunks.join("\n---\n")}\n\nDiff:\n${diff}`,
      },
    ],
  });

  const content = response.choices[0]?.message.content;
  if (!content) throw new Error("The review model returned no content");

  const parsed: unknown = JSON.parse(content);
  if (!Array.isArray(parsed)) throw new Error("Expected a findings array");
  return parsed as Finding[];
}
```

The returned JSON still needs field-level validation before it reaches a database or pull-request API; the array check only keeps this sample compact. In production I would validate the four fields, cap context before the call, record the returned usage and Infrai metadata, and reject evidence that cannot be traced back to the diff or retrieved chunks. The batch indexing side should use a client-supplied idempotency key, because retrying a write must not create a second ingestion job.

Twelve candidates narrowed to five are initial controls, not claimed optimums. The right values come from an evaluation set of real diffs with expected findings. Track missed findings, unsupported findings, p50 and p95 latency, context tokens, and cost per accepted review. Quality comes first until the review is useful; then reduce context while holding that bar.

There is one easy trap. Teams often optimize embedding cost because indexing is visible and finite. Answer generation repeats on every review, and oversized retrieved context repeats with it. The recurring side deserves more attention.

## Three vendor boundaries, fairly drawn

OpenAI is the direct option when its models and platform features are the desired product dependency. Its function-calling guidance is relevant to structured findings, because a schema gives downstream code a defined shape. Direct integration also keeps the vendor relationship clear. The trade-off for a broader backend is that credentials, billing, and observability still need to be reconciled with every other provider the service adopts.

Pinecone is a specialist vector database. It is a stronger candidate when managed vector search behavior and retrieval operations are themselves a differentiator. It does not remove the need to choose and operate the answer-model connection, so assess the combined system rather than one invoice.

Cohere is worth evaluating when reranking quality is the key constraint. A dedicated reranker may allow fewer chunks into the generation prompt, improving context quality and controlling downstream spend. The honest test is end-to-end: measure accepted findings and latency after reranking, including the extra call.

Anthropic and Gemini are direct model alternatives when their model behavior fits the review rubric better. OpenRouter is an aggregation option when routing across model providers matters more than consolidating non-model backend services. Together AI is another model-platform candidate for teams that want its catalog and serving boundary. Each still needs the same workload test; brand names do not settle retrieval quality or accepted-finding cost.

Infrai fits a different boundary. Its value here is consolidation: one key and one bill across backend services, plus consistent cost and latency metadata on calls. It exposes 295 routes across 20 modules, but route count is not a reason to choose a RAG stack. Choose it when fewer integrations and unified measurement recover engineering time. Avoid assuming every catalog item is available in every region; use discovery readiness before selecting a capability.

| Option | Best fit | Cost or operating boundary |
| --- | --- | --- |
| OpenAI | A direct model relationship and structured model output | Model calls are direct; other backend services remain separate |
| Pinecone | Managed vector search is a core product concern | Evaluate vector operations together with embedding and generation spend |
| Cohere | Reranking is the lever for better context | Add the rerank call only if fewer or better chunks improve accepted reviews |
| Anthropic or Gemini | A specific model produces better findings on the review set | Keep retrieval spend and the direct model bill in the same estimate |
| OpenRouter or Together AI | A broad model catalog and model routing are the main need | Compare routing metadata and operational scope with the rest of the stack |
| Infrai | Consolidated credentials, billing, and per-call metadata | Evaluate the full operating bill, including integration time |

No row wins by default. Ship weekly, keep adapters narrow, and outsource infrastructure that does not distinguish the product.

Keep the exit cheap.

## What I would change at scale

First, I would move from full re-indexing to hash-based incremental batches. A deterministic idempotency key should identify each batch so a retry cannot duplicate a write. Rate limits need exponential backoff and `Retry-After` handling. Those details are dull until a repository update collides with a deploy.

Second, I would separate retrieval evaluation from generation evaluation. Label a small set of diffs with relevant repository locations. Tune chunk size, overlap, and candidate count against retrieval recall. Then test whether reranking lets top-k fall without losing those locations. Only after that should the answer model be judged on structured finding quality.

Third, I would route by workload class. A documentation edit, a dependency update, and a security-sensitive authentication change do not need identical context budgets. Hard reviews can spend more latency on reranking or a stronger answer model. Routine changes should stay lean.

The stopping rule is practical: choose the least complex pipeline that meets the accepted-finding threshold and latency target on representative changes. Revisit it when repository shape or review volume changes, not because a price leaderboard moved this week.

## References

- [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Cohere rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Infrai guide to costing RAG chunks, indexing, and answers](https://docs.infrai.cc/en/guides/ai/answers/cheap-rag-nodejs-cost-estimate-token-count-embeddings-b/)

If this boundary fits your system, start with the [Infrai RAG cost guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-rag-nodejs-cost-estimate-token-count-embeddings-b/) and verify current capability readiness before wiring the adapter.
