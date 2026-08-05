# How to Reduce LLM Cost for Summarize, Classify, and Extract JSON Jobs

## TL;DR

For summarize, classify, and extract JSON workloads, the best way to reduce LLM cost is to count prompt and output tokens by job, test small models against a frozen dataset, and batch every request that has no human waiting. Keep a larger model as a measured fallback for the slices where the small one misses the acceptance bar; don't migrate an entire workload on price or a public benchmark alone.

The evaluation constraint matters: a cheaper call is useful only when its valid-output rate and task accuracy leave the total cost per accepted result lower. I ship the boring measurement layer first, then change one lever at a time.

## How should I compare small models, batch processing, and token counting for cheaper LLM JSON extraction?

Start with a unit that maps to the product, not the provider invoice. For classification, mine is cost per correctly labeled item. For extraction, it is cost per schema-valid record that also passes field-level checks. For summarization, I use cost per accepted summary, with human review time included when review is part of the workflow. This prevents a low token price from hiding retries, malformed JSON, or cleanup work.

I build a frozen evaluation set from production-shaped inputs and include the awkward tail: empty fields, conflicting statements, long boilerplate, multilingual fragments, and text that should produce an explicit unknown. Each candidate sees the same prompt, output contract, timeout, and retry policy. I record input tokens, output tokens, latency, parse success, per-field accuracy, and fallback rate. Small models often deserve the first attempt on bounded work because the expected answer is narrow, but that is a hypothesis to grade, not a universal rule.

| Approach | Measure first | Good fit | The catch |
| --- | --- | --- | --- |
| Shorter prompt | Tokens per accepted result | Repeated instructions and oversized examples | Removing context can lower accuracy |
| Small-model first | Field accuracy and fallback rate | Fixed labels and constrained extraction | Hard cases can erase savings through retries |
| Batch processing | Queue age and completion rate | Backfills, nightly jobs, and eval runs | Not suitable when a person is waiting |
| Larger-model fallback | Escalation rate by input slice | Ambiguous or reasoning-heavy records | An unbounded fallback becomes the default bill |
| Self-hosted inference | Utilization and operator time | Stable, sustained load or data constraints | Idle capacity and operations still cost money |

Prompt token counting belongs in the first experiment because it separates model choice from payload waste. Count with the tokenizer that matches the deployed model when one is available, but reconcile estimates with the usage reported by the runtime; tokenization differs, and I'm not sure a local estimate is ever a perfect substitute across model families. Your mileage may vary most on summaries, where quality is less reducible to one exact label.

## Measure the result, not the advertised token rate

The simple approach I tried first was a spreadsheet with input price, output price, and average tokens. It failed as a decision tool because it treated every response as usable. The useful equation is closer to `total runtime cost + retry cost + review cost`, divided by accepted results. That denominator changes the ranking. A model that emits a valid record 99 times out of 100 can beat one with a lower per-token rate that succeeds 90 times, especially when each rejected record triggers another call.

I also separate jobs before aggregating them. A one-label classifier, a five-field extractor, and a summary of a long document don't belong in one average. Track a stable `job_kind`, prompt version, model revision, token counts, validation outcome, latency, and fallback reason for every attempt. Avoid logging raw private text unless the product's data policy permits it; hashes, size buckets, and evaluation IDs are enough for many cost reports.

One config footgun made this painfully concrete. I had set `LLM_REGION=us-east` in one worker while the deployment expected `us-east-1`, and an auth header was assembled from the region-specific credential selected at boot. The first **317 requests** failed authentication in a way that looked like bad input data, then retried through the general queue. I lost most of a morning because the dashboard grouped those attempts under the final successful job. Now I validate the allowed region values at startup and charge every attempt, including retries, to the original job ID — one accepted output can have a surprisingly expensive family tree.

Tiny checks help.

For a decision report, I want the p50 and p95 token counts rather than the mean alone, schema-valid rate, task score with confidence intervals, fallback rate, and cost per accepted result. I run the candidate on the same frozen cases more than once if the runtime is nondeterministic. Then I inspect disagreements instead of averaging them away. A small model may be entirely adequate for receipts but weak on contracts; routing by document type preserves that information, while a single global accuracy number destroys it.

## A focused TypeScript evaluator before model routing

The code below is deliberately provider-independent. It assumes the caller has already captured one attempt at a time and gives the deployment gate a compact report. There is no hard-coded model ID, endpoint, or pricing claim — those values belong in versioned configuration because they change independently of application logic.

```ts
type Attempt = {
  jobId: string;
  jobKind: "summarize" | "classify" | "extract_json";
  modelTier: "small" | "large";
  inputTokens: number;
  outputTokens: number;
  runtimeCostUsd: number;
  reviewCostUsd: number;
  schemaValid: boolean;
  taskScore: number;
  accepted: boolean;
};

type Gate = {
  minimumTaskScore: number;
  minimumAcceptedRate: number;
  maximumFallbackRate: number;
};

function evaluate(attempts: Attempt[], gate: Gate) {
  const finalByJob = new Map<string, Attempt>();
  for (const attempt of attempts) finalByJob.set(attempt.jobId, attempt);

  const finals = [...finalByJob.values()];
  const accepted = finals.filter((item) => item.accepted);
  const totalCostUsd = attempts.reduce(
    (sum, item) => sum + item.runtimeCostUsd + item.reviewCostUsd,
    0,
  );
  const averageScore = finals.length
    ? finals.reduce((sum, item) => sum + item.taskScore, 0) / finals.length
    : 0;
  const acceptedRate = finals.length ? accepted.length / finals.length : 0;
  const fallbackJobs = new Set(
    attempts.filter((item) => item.modelTier === "large").map((item) => item.jobId),
  );
  const fallbackRate = finals.length ? fallbackJobs.size / finals.length : 0;

  return {
    averageScore,
    acceptedRate,
    fallbackRate,
    costPerAcceptedResultUsd: accepted.length
      ? totalCostUsd / accepted.length
      : Number.POSITIVE_INFINITY,
    passes:
      averageScore >= gate.minimumTaskScore &&
      acceptedRate >= gate.minimumAcceptedRate &&
      fallbackRate <= gate.maximumFallbackRate,
  };
}
```

Keep validation outside the model's self-assessment. JSON parsing is only the first gate: verify required properties, allowed labels, numeric ranges, and cross-field rules in ordinary code. A result can be syntactically valid and still claim that an end date precedes a start date. For summaries, automated checks can catch missing required facts or forbidden data, but a small, blinded human sample is still useful because fluency can disguise omissions. The chosen architecture is a cheap-first cascade with explicit exits. Send an eligible job to the tested small tier, validate it, and escalate only on a named condition such as invalid schema, low classifier margin, unsupported input length, or a document class that failed the offline gate. Store the reason next to the attempt and preserve the original job ID through the whole cascade, so the report attributes every token to the result it helped produce. Don't use a broad `catch` as the routing policy; network and authentication errors should follow bounded operational retry rules, while an ambiguous answer is a model-quality decision. If these paths share one fallback counter, a credential typo can look like model regression and a weak extraction can look like infrastructure noise. Separate counters make the action obvious: fix transport for operational failures, revise the eligibility rule for quality failures, and reject the release when either path pushes cost per accepted result above the baseline.

Retries count too.

## Where the cheaper path is the wrong path

Batching wins only when delay is allowed. Nightly enrichment, migrations, evaluation runs, and historical backfills are natural candidates; an interactive autocomplete or support reply is not. The scheduler also needs idempotency, a durable job ID, bounded retries, and a dead-letter state. Otherwise duplicated work can consume the savings while the aggregate completion count still looks healthy.

Small models are not suitable when mistakes carry high downstream cost, the input requires long-range cross-referencing, or the task has no crisp acceptance test. Stick with the larger evaluated tier for those slices, or keep a human decision in the loop. Self-hosting is another conditional choice: I consider it when load is steady enough to keep hardware busy or when deployment constraints require it, but I avoid it for spiky solo-founder traffic because capacity planning, upgrades, and observability become my problem.

Retrieval deserves its own test. If a prompt includes many candidate passages, ranking them before generation can reduce irrelevant context; reranking systems are designed to sort documents by relevance to a query. Measure retrieval recall before dropping passages, though, because a shorter prompt that omits the answer is merely cheap. Speech work is separate again: an open-source speech-recognition model such as Whisper can move transcription outside a per-call text pipeline, but hardware use, latency, and transcription quality still need the same workload-specific accounting.

This is the stopping rule I use: copy the small-model or batch choice only after it passes the frozen task gate, lowers cost per accepted result, and stays inside the latency budget on production-shaped traffic. Watch prompt-token distribution, schema-valid rate, field accuracy, queue age, retries, and fallback rate after release. If fallback climbs, roll back the routing rule before tweaking prompts live. Ship one change, read the counters, then take the next step.

## References

- Cohere Rerank documentation: https://docs.cohere.com/docs/rerank-overview
- OpenAI Whisper repository: https://github.com/openai/whisper
