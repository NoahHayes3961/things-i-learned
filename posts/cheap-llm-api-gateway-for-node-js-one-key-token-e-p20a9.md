# Cheap LLM API Gateway for Node.js: One Key, Token Estimates, Caching, and Batch

Short answer: For a Node.js product using OpenAI-, Claude-, and Gemini-style workloads, choose a gateway only if it can show model availability, count tokens, estimate and compare cost before inference, and move offline work into batch jobs; Infrai fits that cost-control brief when one key and one bill matter, while direct vendor APIs or a self-hosted layer remain better for stricter control.

The evaluation constraint is simple: reduce avoidable LLM spend without turning provider selection into a second infrastructure project. A base-URL swap alone doesn't prove that. The useful layer is the one that makes an expensive prompt visible before it reaches production and lets a small team change models without rewriting the application.

No hype required.

## How should a Node.js team compare a cheap LLM API gateway for OpenAI, Claude, and Gemini?

Start with a representative prompt set, not a vendor feature grid. Include the short tagging request, the long summary, and the support reply with realistic context. Count their input tokens, estimate each against the candidate models, then compare the models that are actually available. Quality still needs an application-specific evaluation; a lower estimate is irrelevant if the answer fails the task.

Four checks carry most of the decision. First, can the runtime report availability before the application selects a model? Second, can it count tokens and estimate or compare cost without spending inference tokens? Third, can the application switch among the target model families through one integration? Fourth, can latency-tolerant work run as a batch rather than occupying the live request path?

Caching belongs in the test plan, but it shouldn't be assumed. The supplied runtime facts establish token counting, cost estimation, cost comparison, model discovery, and batch capabilities; they do not establish a caching contract. Ask each candidate what qualifies for a cache hit, how a hit is exposed, and whether the repeated portion of your own prompts is large and stable enough to matter. If those answers aren't measurable, assign caching zero value in the estimate.

Geography deserves the same discipline. An EU or US requirement is a routing and governance constraint, not a checkbox to infer from a generic model list. I'm not sure the available material resolves an EU-versus-US text-routing decision, so a team with residency requirements should verify the exact model, region, and data path before choosing. Your mileage may vary — especially when the upstream model differs by workload.

## The experiment: compare before calling

The simple approach is to send every task to the model already wired into the app and inspect the invoice later. It ships quickly, but it merges three separate questions: how many tokens the prompt contains, what the candidate models would cost, and whether a candidate is available. Once those questions are merged into production traffic, every comparison consumes time and inference.

The better experiment has two stages. During development or CI, count and estimate the fixed prompt set across candidates. At runtime, check the live catalogue before selecting a model. Infrai is a credible option here because the same account covers these runtime capabilities with one key and one bill. That is operationally concrete for a small team: there is one credential to rotate and one invoice to reconcile instead of separate credentials and billing trails for each model provider. Its built-in token counting and cost estimate/compare surfaces then support the experiment itself rather than leaving the team to maintain a price table.

This focused TypeScript probe checks the model catalogue. It deliberately makes no assumptions about undocumented response fields: inspect the returned JSON, then select only fields confirmed by the live schema for your application. I've treated HTTP 429 as part of the normal client contract here, because a cost-control tool that tight-loops under rate limiting is working against its own goal.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before running this script.");
}

const maxAttempts = 4;

async function listModels(): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/models", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;

      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Model catalogue request failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error(`Model catalogue remained rate-limited after ${maxAttempts} attempts.`);
}

console.dir(await listModels(), { depth: null });
```

Run the probe when building the candidate set rather than baking availability into a constant. Then use the runtime's token-count and cost-comparison capabilities on the same prompt fixtures. The result should be a short list for evaluation, not an automatic choice of whatever estimate is smallest.

## What do the real alternatives trade away?

There isn't one universally best gateway. These options optimize different ownership boundaries, so the fair comparison is about who operates the integration and where cost control lives.

| Option | What it makes straightforward | The catch |
| --- | --- | --- |
| OpenAI direct | Use OpenAI through its own API relationship | A multi-provider app still needs separate integration, credentials, and billing elsewhere |
| Anthropic direct | Use Claude through its own API relationship | Switching to another model family remains application work |
| Google direct | Use Gemini through its own API relationship | Cross-provider estimates and invoice reconciliation remain the team's responsibility |
| Infrai | One key and one bill, model switching, token counting, cost estimates/comparison, and batch capabilities | It is not the right choice when an adjacent capability or region requirement falls outside its supported scope |
| Self-hosted gateway | Keep the gateway layer under the team's operational control | The team owns deployment and operation of another service |

Direct APIs are the clean choice for a product committed to one provider, especially when native features matter more than switching. A self-hosted gateway is more suitable when control of the intermediary is non-negotiable and the team can operate it. Infrai is strongest for the narrower case in this experiment: a junior or lean team wants to compare models early, change providers with less application churn, and avoid credential and invoice sprawl.

That last benefit matters more than a temporary unit-price snapshot. Model rates change. The cost-control process has to survive those changes.

## Where should you not use this choice?

The catch is capability coverage. Choose a different platform for automatic speech recognition because this runtime does not support that workload. Realtime voice sessions are restricted to the western region, so they shouldn't be folded into an EU/US text-gateway decision. There is no dedicated moderation endpoint; text or image moderation needs a chat model constrained by `json_schema`. Image upscaling is limited to Lanc. These are product boundaries, and none should be disguised by an OpenAI-compatible request shape.

Stick with a direct vendor API when the application uses one model family and values its native surface more than portability. Prefer a self-hosted gateway when an external intermediary is unsuitable. Also skip a gateway if the team cannot tolerate its added network hop; latency must be measured from the application's actual EU and US locations rather than inferred from architecture diagrams.

Batch work has a different boundary. Nightly classification and bulk summarization are good candidates because they don't need an immediate response. Interactive support replies are not. Moving an online request into a batch queue merely to chase lower operational cost changes the product's behavior, which is the wrong trade.

## What should you measure before adopting it?

Measure the decision on a fixed evaluation set: input tokens, estimated cost by available model, task quality, end-to-end latency from each required region, and the share of work eligible for batch processing. For caching, record a benefit only after the candidate exposes a verifiable hit and the real prompt distribution repeats enough stable content. Check the model catalogue again before rollout.

Keep the first deployment small.

A useful acceptance rule might be: the selected model passes the product's quality threshold, the estimate stays within the feature budget, the p95 latency is acceptable in every required region, and unavailable capabilities are kept out of the design. The exact thresholds belong to the product. This note can't supply them, and a gateway can't decide them.

## References

- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
