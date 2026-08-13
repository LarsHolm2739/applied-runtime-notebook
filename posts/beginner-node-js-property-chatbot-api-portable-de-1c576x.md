# Beginner Node.js Property Chatbot API: Portable Developer Experience Contracts

**Short answer:** for a beginner building a Node.js in-app chatbot over private property records, own the application contract and require each API transport to pass the same access, citation, and failure tests.

Provider portability is the deciding constraint for a property-management chatbot over a private knowledge base. Put either transport behind an application-owned TypeScript interface, while keeping retrieval, citations, access checks, and conversation state outside it.

| Choice | Best fit | Cost of the choice |
| --- | --- | --- |
| OpenAI-compatible transport | A beginner needs a small first adapter and expects to test other compatible backends | Compatibility is a starting hypothesis, not proof that every backend behaves identically |
| Native Anthropic transport | The application deliberately depends on that native request and response contract | A later provider change requires another adapter |
| Provider-neutral application contract | The chatbot must preserve product behavior across either transport | You own normalization, tests, and error taxonomy |

**Recommendation:** make the third row the permanent boundary. Evaluate each transport against the same contract tests instead of treating its label as proof of fit. This is not a claim that one model answers lease questions better. It is a way to keep a weekly shipping cadence while preserving an exit.

## How should a beginner choose an API for a Node.js in-app chatbot?

Choose by counting the concepts that can leak into the product, not by counting setup lines in a quickstart. A property manager asking, "Can I keep a bicycle on the balcony?" triggers more than generation: the application identifies the tenant and property, retrieves only authorized handbook passages, asks for an answer grounded in those passages, and returns citations the user can inspect. The API transport is one step in that chain.

The OpenAI-compatible shape is the practical first transport when provider portability leads the decision. The catch is that the word "compatible" does not make model behavior, optional fields, streaming events, limits, or operational policies interchangeable. I don't treat a successful hello-world request as portability evidence. I treat the same contract tests passing against a second transport as evidence.

A native Anthropic integration is a reasonable runner-up when the team has intentionally selected that native contract and accepts the coupling. Stick with it when removing provider-specific fields would discard behavior the product actually needs. Do not add a generic layer merely to look flexible; an abstraction with one implementation and no contract tests is paperwork.

Keep the user-facing promise narrower than either upstream API. The chatbot should promise an answer, source references, and a stable application error shape. It should not promise that raw provider messages, finish reasons, or event names will reach the browser unchanged.

Ship the boundary.

## Portability starts with evidence and access control

The first criterion is whether the application owns the retrieval record. For each answer, retain the document identifiers and revision identifiers used to assemble context, subject to the product's retention policy. A generated sentence without that record cannot be traced back to a building handbook or lease addendum. Switching the generation transport later will not repair missing lineage.

Private knowledge bases also make authorization a pre-retrieval rule. A tenant in Building A must never retrieve a maintenance note, owner memo, or lease clause from Building B. Put property and role filters into retrieval before prompt construction. Prompt text such as "only use this tenant's documents" is not an access-control boundary.

This is where the revenue-per-hour lens helps. I would rather spend an afternoon on one authorization function and its tests than maintain two elaborate prompt stacks. The former protects every answer. The latter can be replaced.

Data lifecycle is the second criterion. The GDPR text includes rights and obligations relevant to personal data, including a right to erasure with stated conditions and exceptions. An engineering design still needs legal review, but it should at least make deletion possible: conversation rows, retrieved excerpts, feedback, logs, and cached answers need documented owners and retention rules. I'm not sure one retention period fits leases, support chats, and operational records; the governing purpose and legal basis have to resolve that question.

Don't log the prompt by default. Record a request ID, tenant scope, document revision IDs, adapter name, latency, token or usage metadata returned by the selected transport, citation coverage, and the normalized outcome. Redact or omit private text. That gives an operator useful evidence without quietly building a second knowledge base in log storage.

## A thin TypeScript boundary is enough

The application contract below knows nothing about either upstream wire format. Each transport adapter must turn the same `ChatRequest` into the same `ChatAnswer`. The retrieval layer stays outside, which means changing generation providers does not change authorization or citation assembly.

```ts
type Passage = {
  documentId: string;
  revisionId: string;
  text: string;
};

type ChatRequest = {
  question: string;
  propertyId: string;
  passages: Passage[];
};

type ChatAnswer = {
  text: string;
  citedDocumentIds: string[];
  outcome: "answered" | "insufficient_context";
};

interface ChatTransport {
  answer(request: ChatRequest): Promise<ChatAnswer>;
}

type SearchPrivateKnowledge = (input: {
  propertyId: string;
  actorId: string;
  question: string;
}) => Promise<Passage[]>;

export async function answerPropertyQuestion(
  input: { propertyId: string; actorId: string; question: string },
  search: SearchPrivateKnowledge,
  transport: ChatTransport,
): Promise<ChatAnswer> {
  const passages = await search(input);

  if (passages.length === 0) {
    return {
      text: "I couldn't find an authorized source for that property.",
      citedDocumentIds: [],
      outcome: "insufficient_context",
    };
  }

  const answer = await transport.answer({
    question: input.question,
    propertyId: input.propertyId,
    passages,
  });

  const allowed = new Set(passages.map((passage) => passage.documentId));
  const citationsAreValid = answer.citedDocumentIds.every((id) => allowed.has(id));

  if (!citationsAreValid) {
    throw new Error("Transport returned a citation outside the retrieved set");
  }

  return answer;
}
```

The two concrete adapters can use different SDKs or plain HTTP; that code belongs in two small modules. Normalize upstream rate limits, authentication failures, timeouts, and invalid response shapes into an internal error union there. Retry only operations your policy considers safe, use a ceiling, and keep retry behavior out of the route handler so it can be tested with a fake clock.

The long paragraph is the test plan, because this is where "portable" either becomes real or evaporates. Run both adapters against the same fixtures: an ordinary policy question, no authorized passages, two revisions of the same handbook, a passage containing instructions aimed at the model, citations outside the retrieved set, cancellation by the browser, and a response that fails the application schema. Assert the normalized outcome rather than exact prose. Then run a small, reviewed evaluation set of real property-policy questions with expected source documents and a rule that unsupported claims fail. Model output varies, so exact string matching will be brittle; source selection, citation membership, refusal on insufficient context, and absence of cross-property material are stable product requirements. Your mileage may vary on the size of that evaluation set, but ten carefully reviewed cases teach more at the start than a dashboard with no failure definitions.

Tiny interface. Serious tests.

## Batch work and live chat are different lanes

The OpenAI Batch API guide describes an asynchronous batch workflow. That makes it relevant to offline jobs, not a substitute for the live request path of an in-app chatbot. Keep bulk evaluation, nightly reprocessing, or other delay-tolerant work behind a separate queue and interface; do not let a batch job's lifecycle leak into the browser contract.

This separation also protects weekly delivery. The first release can answer one authorized question with citations. Evaluation can run offline against saved, approved fixtures. Index refresh can be a background process. None of those jobs needs to complicate the live `ChatTransport` interface.

It sounds almost dull. Good. Undifferentiated plumbing should stay dull enough to outsource or replace.

## When does the native API fit the contract?

The native Anthropic API fits when its specific contract is an intentional dependency, the team is prepared to maintain that adapter, and a portability layer would hide behavior the product relies on. In that case, direct coupling is an explicit engineering trade-off. An honest dependency is easier to operate than a supposedly universal interface full of escape hatches.

The OpenAI-compatible route is also not suitable when procurement, data handling, regional availability, or a required model narrows the acceptable provider set to one. Those checks are deployment-specific, and the two supplied references do not settle them. Resolve them with current contracts, security documentation, and a data-protection review before sending private property records.

For a solo SaaS, the decision rule stays blunt: keep the application contract, retrieval evidence, tenant boundary, and evaluation set under your control. Choose the thinnest transport that passes those tests today. Revisit the choice when a real product requirement changes, not because a new quickstart looks nicer.

## Sources

- https://platform.openai.com/docs/guides/batch
- https://gdpr-info.eu
