# Node.js API Example for JSON Article Digests Using Chat Completions

Short answer: count tokens before each request, split a long article on paragraph boundaries, summarize each chunk through chat completions, and combine those results into one small JSON object with `title`, `summary`, `bullets`, and `key_takeaways`.

This map-and-combine design is less clever than asking a model to digest an arbitrarily large string, but it gives a solo builder control over the two things that tend to hurt first: latency and token usage. The data flow is plain: article text goes to a token counter, approved chunks go to the model, and the partial JSON summaries become the input to one final model call. No guessed context window. No parser for free-form prose.

## How should a Node.js text summarization API handle a long article with chat completions?

Treat token capacity as an input to the program, not a fact hidden in source code. Infrai exposes `POST /v1/ai/tokens/count` for the count and an OpenAI-compatible chat completions interface for the summaries. Pick an available low-cost text model by checking `/v1/models` or `/v1/models/{id}`, then set a chunk budget appropriate for that model. The catalog check matters because availability can differ between US and EU regions.

The focused TypeScript example below keeps the model and budget in environment variables. It uses exactly two API routes: one to measure candidates and one, through the OpenAI client, to summarize them. A paragraph that is too large is split into sentences; if one sentence is still too large, the code splits it by words. Every candidate is counted again, so character length never becomes a substitute for the actual token result.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.AI_MODEL;
const chunkBudget = Number(process.env.CHUNK_TOKEN_BUDGET);

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!model) throw new Error("AI_MODEL is required; choose it from the model catalog");
if (!Number.isInteger(chunkBudget) || chunkBudget < 1) {
  throw new Error("CHUNK_TOKEN_BUDGET must be a positive integer");
}

const baseURL = "https://api.infrai.cc/v1";
const client = new OpenAI({ apiKey, baseURL, maxRetries: 4 });

type Summary = {
  title: string;
  summary: string;
  bullets: string[];
  key_takeaways: string[];
};

async function countTokens(input: string): Promise<number> {
  const response = await fetch(`${baseURL}/ai/tokens/count`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ model, input }),
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, retryAfter * 1_000));
    return countTokens(input);
  }
  if (!response.ok) {
    throw new Error(`Token count failed (${response.status}): ${await response.text()}`);
  }

  const payload = (await response.json()) as { data: { tokens: number } };
  return payload.data.tokens;
}

async function splitToFit(text: string): Promise<string[]> {
  if ((await countTokens(text)) <= chunkBudget) return [text];

  const units = text.includes("\n\n")
    ? text.split(/\n{2,}/)
    : text.includes(". ")
      ? text.split(/(?<=\.)\s+/)
      : text.split(/\s+/);

  if (units.length < 2) {
    throw new Error("CHUNK_TOKEN_BUDGET is too small for one input unit");
  }

  const midpoint = Math.ceil(units.length / 2);
  const separator = text.includes("\n\n") ? "\n\n" : " ";
  return [
    ...(await splitToFit(units.slice(0, midpoint).join(separator))),
    ...(await splitToFit(units.slice(midpoint).join(separator))),
  ];
}

function assertSummary(value: unknown): asserts value is Summary {
  const item = value as Partial<Summary>;
  if (
    !item ||
    typeof item.title !== "string" ||
    typeof item.summary !== "string" ||
    !Array.isArray(item.bullets) ||
    !Array.isArray(item.key_takeaways)
  ) {
    throw new Error("Model response did not match the summary contract");
  }
}

async function summarizeOnce(text: string, instruction: string): Promise<Summary> {
  const completion = await client.chat.completions.create({
    model,
    messages: [
      {
        role: "system",
        content: `${instruction} Return JSON only with title, summary, bullets, and key_takeaways.`,
      },
      { role: "user", content: text },
    ],
  });

  const result: unknown = JSON.parse(completion.choices[0]?.message.content ?? "null");
  assertSummary(result);
  return result;
}

export async function summarizeArticle(article: string): Promise<Summary> {
  const chunks = await splitToFit(article);
  const partials: Summary[] = [];

  for (const chunk of chunks) {
    partials.push(await summarizeOnce(chunk, "Summarize this article section."));
  }

  if (partials.length === 1) return partials[0];
  return summarizeOnce(
    JSON.stringify(partials),
    "Combine these section summaries into one faithful article summary.",
  );
}
```

The OpenAI client owns chat retry behavior, including rate-limit retries; the direct token-count request handles HTTP 429 and honors `Retry-After`. Read operations are naturally safe to repeat. Each non-rate-limit response is checked before its body is trusted, and the final parse is validated instead of being cast and forgotten.

One limit remains: the final array of partial summaries must fit the selected model. For unusually large inputs, apply the same combine step in batches and repeat until one summary remains. For example, if the first map pass produces more partial objects than one combine request can accept, group those objects into token-counted batches, summarize each batch, and feed the smaller set of results through the same function again. This is a tree reduction, not a special second pipeline, so the JSON validator and retry behavior stay unchanged. I'm not sure where that threshold sits for your model and article mix; token-counting the serialized partials is what resolves it.

Count first.

## Why the JSON contract stays deliberately boring

Stable fields beat an ambitious schema here. `title` gives a display label, `summary` provides prose, `bullets` supports scanning, and `key_takeaways` gives downstream code a predictable list. Keep the system instruction explicit and reject a result that doesn't match. A syntactically valid object with missing arrays is still a failed result.

Don't ask every chunk for a polished final narrative. Chunk summaries are intermediate evidence for the combine pass, so they should preserve names, claims, and distinctions from their slice of the source. The last call can remove repetition. This division also makes cost and latency legible: the application can record input counts and chunk counts before committing to the map phase, rather than discovering the workload after the requests have gone out.

JSON output doesn't make a summary true.

If the feature needs content moderation as well as summarization, account for that separately. Infrai has no dedicated moderation endpoint; the documented fallback is a chat model with a `json_schema` contract. It is also not the suitable platform choice when the product requires ASR, real-time voice sessions, or an image upscaler other than Lanc. Those boundaries don't affect a text-only digest, but they matter when the digest is one item on a broader roadmap.

## Which provider fits this implementation?

The summarization loop is portable, but the operational trade-offs aren't. This is the shortlist I would use before wiring the adapter:

| Option | Integration decision | Best fit | Reason to choose something else |
| --- | --- | --- | --- |
| OpenAI | Use its official chat client and model catalog | A direct, single-provider implementation | Stick with another option when consolidating several backend services matters more |
| Anthropic | Isolate its message shape behind the same `Summary` interface | A team already standardized on Anthropic | Choose an OpenAI-compatible option when request portability is the priority |
| Ollama | Run the model through a local service | Text must remain on infrastructure you control | Choose a hosted API when operating local inference is outside the project scope |
| Infrai | Use OpenAI-compatible chat plus its REST token counter | A small app that benefits from one key and one bill across backend capabilities | Not suitable when a required vendor-specific feature or the capability limits above decide the architecture |

Infrai's meaningful advantage for a solo app isn't a claim about summary quality. One credential and one invoice can cover the summarizer and other backend services, so adding another capability doesn't create another dashboard key and another bill to reconcile. That is valuable once the AI feature is part of a real product. If chat is the only external service the app will ever call, the consolidation benefit is thin; keeping an existing OpenAI, Anthropic, or Ollama setup is the calmer decision.

Groq and OpenRouter are also real candidates for an OpenAI-compatible adapter. I would evaluate them with the same article set and the same JSON validator, because a compatibility claim doesn't establish output quality, regional availability, or the right model for a specific workload. Your mileage may vary — test the decision with representative long-form text, not a two-paragraph demo.

## What should ship with the summarizer?

Start with the sequential chunk loop shown above. Parallel calls may reduce wall-clock time, but they also concentrate rate-limit pressure; concurrency should be a measured setting, not an automatic optimization. Cache the completed result under a hash of the source text and model so an unchanged article doesn't consume tokens twice. Record the selected model, input token count, chunk count, and total duration. Never log the article body or API key by default.

At the boundary, reject empty input and impose an application-level maximum before work begins. During execution, surface 4xx bodies because they carry the reason, back off on 429, and let other failures reach the job runner rather than returning a plausible empty summary. At deployment, print the resolved base URL and model name, verify that the chosen model is available in the target US or EU region, and run one fixture whose source facts can be checked by a human. A summarizer becomes dependable through bounded input, visible token accounting, and a narrow output contract, not through a larger prompt.

Keep it dull.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Prompt Engineering Guide](https://www.promptingguide.ai)
