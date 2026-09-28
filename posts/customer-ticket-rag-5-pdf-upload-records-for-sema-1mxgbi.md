# Customer Ticket RAG — 5 PDF Upload Records for Semantic Search

The expensive constraint in customer-support triage is not the first successful PDF search. It is changing an embedding provider or chunking rule without losing the evidence behind an answer. For a one-person SaaS shipping weekly, the sensible choice is to store five durable records outside the model call: document, page, chunk, embedding profile, and retrieval hit.

**TL;DR:** Extract page-aware text once, give every chunk a stable content hash, keep embeddings replaceable, and build citations from stored locations rather than generated prose. This makes a supplier change a background re-index, not an application rewrite.

## How should Node.js RAG handle PDF upload and semantic search?

An incoming ticket might ask, "Can I return the blue jacket after removing the tag?" The source material could be a returns-policy PDF, a seasonal exception sheet, and a product-care guide. A cosine score can rank passages, but it cannot establish which uploaded revision, page, or exact text supported the reply.

The five records change at different speeds. A document record holds the file hash and business identity. A page record preserves extraction order and page number. A chunk record holds quoted text plus offsets into that page. An embedding profile names the model, vector dimensions, and chunker version used for one indexing run. A retrieval hit links a query run to a chunk and its score. In a Node.js RAG service, that metadata is the semantic search contract; the generated answer is only a consumer.

This is deliberate duplication. Storage is cheaper than reconstructing provenance during a support dispute, and the records let an operator replace one policy without guessing which vectors came from it. Consider a seasonal return policy uploaded on November 1 and replaced on January 3. If the application stores only vectors, an old ticket can point to text that no longer exists in the current PDF. A file hash, page number, character span, and embedding profile preserve the old decision without keeping the old policy active for new tickets. The revenue-per-hour test is blunt: schema work that prevents a manual citation audit is worth doing; a clever orchestration layer usually is not. Outsource the undifferentiated model computation behind a narrow interface.

Own the evidence.

## The smallest working ingestion path

The implementation boundary is two functions: page extraction and embedding. Either can move without changing the records consumed by search. PDF extraction itself is library-specific, so this example receives already extracted pages instead of pretending every PDF has the same text layer. Scanned files need OCR before this point.

```ts
import { createHash } from "node:crypto";

type Page = { page: number; text: string };
type Chunk = {
  id: string;
  documentId: string;
  page: number;
  start: number;
  end: number;
  text: string;
  chunkerVersion: "sentence-window-v1";
};
type EmbeddingProfile = {
  id: string;
  model: string;
  dimensions: number;
};
type Embedder = {
  profile: EmbeddingProfile;
  embed(input: string[]): Promise<number[][]>;
};

const sha256 = (value: string) =>
  createHash("sha256").update(value).digest("hex");

function chunkPage(documentId: string, source: Page, limit = 900): Chunk[] {
  const sentences = source.text.match(/[^.!?]+[.!?]+|[^.!?]+$/g) ?? [];
  const chunks: Chunk[] = [];
  let buffer = "";
  let start = 0;

  const append = (text: string) => {
    const clean = text.trim();
    if (!clean) return;
    const actualStart = source.text.indexOf(clean, start);
    if (actualStart < 0) throw new Error("Chunk cannot be located on page");
    chunks.push({
      id: sha256(`${documentId}:${source.page}:${actualStart}:${clean}`),
      documentId,
      page: source.page,
      start: actualStart,
      end: actualStart + clean.length,
      text: clean,
      chunkerVersion: "sentence-window-v1"
    });
    start = actualStart + clean.length;
  };

  for (const sentence of sentences) {
    if (buffer && buffer.length + sentence.length > limit) {
      append(buffer);
      buffer = "";
    }
    buffer += sentence;
  }
  append(buffer);
  return chunks;
}

async function indexPages(documentId: string, pages: Page[], embedder: Embedder) {
  const chunks = pages.flatMap((page) => chunkPage(documentId, page));
  const vectors = await embedder.embed(chunks.map((chunk) => chunk.text));
  if (vectors.some((vector) => vector.length !== embedder.profile.dimensions)) {
    throw new Error("Embedding dimensions do not match the stored profile");
  }
  return chunks.map((chunk, index) => ({
    chunk,
    profileId: embedder.profile.id,
    embedding: vectors[index]
  }));
}
```

This chunker is intentionally small. It retains sentence boundaries where extracted text exposes them, caps the working window at 900 characters, and records offsets after trimming. It is not a universal parser. Tables, columns, headers, and repeated footers require extraction tests against the actual policy corpus.

One trap deserves attention: `indexOf` can match an earlier repeated sentence if the cursor is wrong. Advancing `start` after every emitted chunk makes the lookup monotonic. Production ingestion should reject a negative offset and quarantine the page instead of storing a citation it cannot reproduce.

Document and chunk rows are source-derived records. The embedding row is disposable. That distinction makes provider portability practical: create a new profile, embed all current chunks into a new column or table, validate retrieval, then switch the active profile. Do not overwrite the old profile halfway through a job.

With pgvector, cosine distance uses the `<=>` operator, while HNSW and IVFFlat indexes support approximate nearest-neighbor search. The index choice is an operational decision, not part of the application contract. Keep the query result shaped like this:

```ts
type RetrievalHit = {
  queryId: string;
  chunkId: string;
  documentId: string;
  page: number;
  start: number;
  end: number;
  text: string;
  score: number;
  profileId: string;
};

function citation(hit: RetrievalHit, fileName: string) {
  return {
    label: `${fileName}, page ${hit.page}`,
    quote: hit.text,
    locator: {
      documentId: hit.documentId,
      page: hit.page,
      start: hit.start,
      end: hit.end
    }
  };
}
```

The answer generator may paraphrase a return rule. The citation must not. It should point to the retrieved chunk and include the stored quote, so the UI can open the relevant page and an operator can verify it. If a PDF revision changes, its file hash creates a new document version; old ticket decisions can still resolve against the evidence used at the time. The limitation is equally concrete: character offsets are unsuitable when the rendered PDF location matters, because extraction order can differ from visual order. Store page coordinates from the extractor instead, or show the entire cited page for manual review.

## Test the evidence chain before the prose

A polished answer can hide weak retrieval. Start evaluation earlier. Build a small fixture set from real policy shapes: a straightforward return window, a conflicting seasonal exception, a scanned page, a two-column table, and a policy revision. The expected result for each fixture is a document version, page, and acceptable quote span, not an exact generated sentence.

Three checks catch the costly failures:

1. Re-extract the stored page and confirm every chunk offset selects the stored text.
2. Run the same queries against old and candidate embedding profiles, then compare whether the expected evidence remains in the retrieved set.
3. Refuse to answer automatically when no retrieved passage clears a threshold established from the fixture set; route that ticket for human review.

Do not copy a threshold from a tutorial. Similarity scores depend on the embedding model, corpus, query wording, and distance conversion. Calibrate with labeled tickets, record the profile used, and monitor abstention and citation-selection rates after deployment.

Scores drift.

Ship the ingestion path separately from the ticket responder. A failed re-index should leave the current profile serving traffic, while resumable jobs continue from recorded chunk IDs. Log document version, profile ID, selected chunk IDs, distances, latency, and final disposition. Avoid logging raw customer messages unless retention and access controls explicitly permit it; support text often contains personal or order data.

## What I would change at scale

At modest volume, exact search keeps the system legible. As the corpus and query load grow, benchmark approximate indexes with the same labeled fixture set before enabling one. HNSW and IVFFlat trade build time, memory, or recall against query speed. My decision rule is to choose exact search first because it removes an approximation variable from citation testing, then accept approximate search only when measured query load requires it. Measure on the deployed data.

I would also move OCR and extraction into isolated workers, add per-document ingestion states, and partition re-index work by embedding profile. None of those changes should alter the responder contract. The application still asks for ranked `RetrievalHit` records and renders citations from immutable source locations.

This design spends more rows and keeps old vectors during migrations. That is the trade. In return, supplier changes, chunker revisions, and index tuning become reversible maintenance work. For a one-person operation, reversibility protects the weekly shipping cadence better than hiding the pipeline behind a large framework.

## References

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/pgvector/pgvector
