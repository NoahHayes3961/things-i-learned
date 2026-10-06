# 500-Page Internal Assistant: Freshness Decides Full Text or Vector Database

A 500-page wiki changes often enough that stale chunks can matter more than the choice of search engine. Start with full-text search, chunk the wiki along meaningful support-document boundaries, and add a vector index only if real near-duplicate queries repeatedly miss relevant pages. That choice keeps the first release inspectable while preserving a clean path to hybrid retrieval.

**TL;DR:** Full text is enough for an initial internal wiki assistant when users search with product terms, error codes, feature names, and phrases that also occur in the documents. Semantic retrieval earns its extra index, synchronization path, and debugging surface when the job depends on matching differently worded records, such as recognizing that "verification email never arrived" and "account activation message missing" describe a similar support issue. Test that failure directly.

## Should a 500-page internal wiki assistant use a vector database?

Page count is a poor proxy for retrieval difficulty. Five hundred pages can be easy when queries repeat the wiki's vocabulary, or awkward when support agents paraphrase customer language while the documentation uses internal terminology. The deciding variable is lexical overlap, not collection size alone.

The naive experiment indexes each page as one document. It fails quietly. A long troubleshooting page may mention ten symptoms, several product versions, and unrelated resolution paths; a match can retrieve the right page while giving the generator a broad, noisy passage. Splitting every fixed number of characters creates the opposite problem: headings become detached from steps, qualifications land in another chunk, and two chunks from the same page can look like duplicate evidence.

Chunk boundaries decide it.

I would chunk by the document's own structure first: heading path, paragraph groups, lists, and code samples that depend on the preceding explanation. Keep the stable page identifier and revision identifier on every chunk. This is a trade-off, not a magic number: smaller chunks improve localization, while larger chunks preserve the context needed to distinguish a symptom from its exception.

## The focused experiment

Use a frozen sample of actual, permission-safe queries and label which wiki section should support each answer. Include exact identifiers, ordinary how-to questions, paraphrases, and the near-duplicate support language that motivated the assistant. Do not judge the generated prose yet. First ask whether retrieval returns the needed evidence and whether that evidence is current.

Run full-text retrieval as the baseline. Then test a semantic candidate generator over the same chunk set, with the same access filter and the same evaluation queries. A hybrid experiment can combine candidates, but keep the component scores visible; collapsing everything into one score makes a miss hard to explain. Retrieval-augmented generation relies on retrieved external knowledge, so a fluent answer cannot repair absent or obsolete evidence [1].

The useful record is compact:

| Query class | Evidence to inspect | Likely lesson |
| --- | --- | --- |
| Error code or feature name | Exact section rank | Full text often has strong signals |
| Customer paraphrase | Relevant section presence | Semantic candidates may recover vocabulary mismatch |
| Near-duplicate ticket | Shared resolution evidence | Chunk scope can dominate engine choice |
| Recently edited policy | Revision and indexed time | Freshness failure invalidates either method |

One short query can expose the distinction. Suppose a new ticket says "activation message missing," while the wiki section is titled "Troubleshoot undelivered verification email." Token matching may have little to work with. A semantic candidate can help, but only if the chunk contains the heading and the relevant diagnostic steps. Embedding a whole handbook or a severed paragraph weakens the experiment.

## Keep freshness in the data model

A second index creates a second representation of the wiki. Every create, edit, permission change, move, and deletion must reach it. This is where a seemingly small assistant acquires operational weight. Stale advice is worse than a clean miss because it looks authoritative.

Treat chunks as derived data. Their identity should follow the source page and a deterministic chunking version, while their revision follows source content. Upsert the new revision, verify it is queryable, then retire chunks that belong only to the previous revision. A periodic reconciliation job should compare source revisions with indexed revisions; event delivery alone does not prove convergence.

```ts
interface WikiChunk {
  chunkId: string;
  pageId: string;
  sourceRevision: string;
  chunkerVersion: string;
  headingPath: string[];
  text: string;
  indexedAt: string;
}

interface SearchHit {
  chunk: WikiChunk;
  score: number;
  channel: "full_text" | "semantic";
}

function acceptFreshHits(
  hits: SearchHit[],
  currentRevision: ReadonlyMap<string, string>,
): SearchHit[] {
  return hits.filter(({ chunk }) =>
    currentRevision.get(chunk.pageId) === chunk.sourceRevision
  );
}
```

This check is deliberately independent of a search product. It also shows why metadata is part of retrieval correctness rather than administrative decoration. In a production request path, authorization belongs in the candidate-selection boundary as well; post-filtering retrieved text can leak content into logs, caches, or model context before the filter runs.

Re-chunking deserves its own migration. Changing the splitting rules without changing `chunkerVersion` makes comparisons meaningless and can leave old and new boundaries mixed in one result set. Build the replacement set, evaluate it, switch reads, and remove the superseded set after the rollback window.

## When does a vector index earn its place?

Add semantic retrieval when labeled misses cluster around meaning rather than freshness, authorization, or bad chunk boundaries. One memorable paraphrase is not enough. Look for a repeatable query class where the full-text baseline lacks the terms needed to find the correct evidence and semantic candidates recover it without pushing too much irrelevant context into the prompt.

Keep full text even then. Exact ticket IDs, error strings, product codes, names, and quoted UI text are lexical evidence. Semantic similarity answers a different question. A practical hybrid path gathers a bounded candidate set from each channel, deduplicates by chunk identity, and reranks against the original query; it does not pretend the two raw scores share a universal scale.

The cost decision includes more than storage. Count embedding work after edits, reconciliation traffic, additional query latency, operational alerts, and the tokens consumed by irrelevant retrieved chunks. For a solo builder, the debugging tax matters: a full-text miss can often be explained by terms and filters, while semantic similarity adds model version, vector generation, and score behavior to the investigation. Avoiding that machinery until it changes measured outcomes is a ship-first choice, not an argument against vectors.

## Measure this before copying the choice

Measure evidence recall by query class, stale-hit rate after edits, permission-filter correctness, duplicate chunks in the final context, retrieval latency, and context tokens passed to the model. Record the source revision beside every cited chunk so a bad answer can be traced to retrieval, freshness, or generation.

Also inspect no-answer behavior. If neither channel retrieves adequate evidence, the assistant should decline or ask for clarification rather than filling the gap. The experiment succeeds when it tells you which failure you have.

For this 500-page case, the default remains full text over structure-aware, revisioned chunks. Add semantic candidates for demonstrated paraphrase misses, then retain lexical retrieval for exact signals. The database category is secondary. Fresh evidence with a visible provenance trail is the actual requirement.

## Sources

References:

1. Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
