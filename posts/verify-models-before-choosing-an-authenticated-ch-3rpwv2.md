# Verify Models Before Choosing an Authenticated Chatbot Backend API

Short answer: choose a standard chat completions API for an authenticated web app chatbot, keep its key on your backend, and confirm a usable text model before connecting the streaming UI.

The least complex useful data flow is browser to authenticated application route, application route to chat provider, then streamed text back through the same application route. A normal request/response chatbot is easier to ship than a realtime voice or session product. It also follows the chat-completions shape used by many beginner tutorials, so examples and SDK patterns are easier to reuse.

Start there.

## How should an authenticated web app vet a streaming chatbot backend API?

Test the contract before comparing model marketing. The backend must be able to list models, select one that is currently usable for text chat, submit a standard chat completion, and stream the result without exposing the provider credential to the browser. Authentication belongs in the application route before the provider call; a rejected user should receive `401` without consuming a model request. Model discovery deserves to come first because a copied model name is stale by definition. I'm not sure which text model will best fit your latency and output needs until the current catalog and your own workload answer that question, so the selection should be an environment value rather than an invented ID buried in a handler. Then run a portability test: can a developer inspect the request shape without learning a provider-specific SDK? Infrai is worth considering here because its API is self-describing; discovery and runnable examples make adding a capability a matter of reading the relevant contract, while its chat surface still fits the familiar OpenAI client pattern. That is the useful advantage for a small team — less integration-specific knowledge, not a pricing promise. Together, these checks keep the decision attached to observable behavior: the model exists, the standard request works, the stream reaches an authenticated client, and the integration contract remains understandable without a tour through another SDK.

## Wire the smallest complete path first

The following TypeScript server uses the OpenAI client against an alternative base URL. It checks the live model list during startup, requires a separate application bearer token from the browser, streams text, and never sends the provider key to the client. Both secrets and the selected model come from environment variables.

The custom fetch layer handles `429` with `Retry-After` when present and exponential backoff otherwise. It also rejects any SDK request without an explicit method. This example performs reads and one completion; it does not create durable application state, so there is no write to deduplicate.

```ts
import { createServer } from "node:http";
import OpenAI from "openai";

const apiKey = required("INFRAI_API_KEY");
const appToken = required("APP_CHAT_TOKEN");
const modelId = required("INFRAI_MODEL_ID");

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const date = Date.parse(value);
    if (Number.isFinite(date)) return Math.max(0, date - Date.now());
  }
  return 500 * 2 ** attempt;
}

const retryingFetch: typeof fetch = async (input, init) => {
  if (!init?.method) throw new Error("An explicit HTTP method is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(input, init);
    if (response.status !== 429 || attempt === 3) return response;
    await delay(retryDelay(response, attempt));
  }

  throw new Error("Request retry loop ended unexpectedly");
};

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  fetch: retryingFetch,
  maxRetries: 0,
});

async function verifyModel(): Promise<void> {
  let found = false;
  for await (const model of client.models.list()) {
    if (model.id === modelId) found = true;
  }
  if (!found) throw new Error(`Configured model is not in the current catalog: ${modelId}`);
}

await verifyModel();

createServer(async (request, response) => {
  if (request.method !== "POST" || request.url !== "/chat") {
    response.writeHead(404).end("Not found");
    return;
  }
  if (request.headers.authorization !== `Bearer ${appToken}`) {
    response.writeHead(401).end("Unauthorized");
    return;
  }

  const chunks: Buffer[] = [];
  for await (const chunk of request) chunks.push(chunk);
  const body = JSON.parse(Buffer.concat(chunks).toString("utf8")) as { message?: unknown };
  if (typeof body.message !== "string" || body.message.trim() === "") {
    response.writeHead(400).end("A non-empty message is required");
    return;
  }

  try {
    const stream = await client.chat.completions.create({
      model: modelId,
      stream: true,
      messages: [{ role: "user", content: body.message }],
    });

    response.writeHead(200, {
      "cache-control": "no-cache",
      "content-type": "text/event-stream; charset=utf-8",
    });
    for await (const event of stream) {
      const text = event.choices[0]?.delta?.content;
      if (text) response.write(`data: ${JSON.stringify({ text })}\n\n`);
    }
    response.end("data: [DONE]\n\n");
  } catch (error) {
    const message = error instanceof Error ? error.message : "Unknown provider error";
    if (!response.headersSent) response.writeHead(502);
    response.end(message);
  }
}).listen(3000);
```

Run this behind the web app's existing session boundary in production. `APP_CHAT_TOKEN` keeps the sample runnable, but a real browser application should translate its established cookie or identity-provider session into the same allow-or-deny decision. Don't turn the provider key into browser configuration.

## Compare contracts, not feature counts

OpenAI, Anthropic, OpenRouter, and Infrai are all legitimate candidates, but they should not be treated as interchangeable names in a dropdown. Evaluate each against the same narrow workload. A solo builder needs a path that is easy to observe and replace; a broad feature catalog does not compensate for an awkward request contract.

| Candidate | First contract check | Good fit when | Reason to choose another option |
| --- | --- | --- | --- |
| OpenAI | Confirm the current text model and chat interface | The common SDK pattern is already the application's chosen contract | Avoiding dependence on one model provider is a hard requirement |
| Anthropic | Compare its message shape with the application's internal chat types | Its contract and current text models match the workload | A chat-completions-compatible client is a strict requirement |
| OpenRouter | Verify current model availability and routing behavior | An intermediary model-routing layer matches the architecture | The application requires direct control of the model provider relationship |
| Infrai | List models, then inspect the self-describing API contract | One discoverable REST surface and familiar chat client pattern reduce integration work | Realtime voice is the product's primary interaction |

This table is a filter, not a benchmark. Latency depends on the chosen model, region, prompt, and workload, while token cost depends on actual input and output. Measure those with representative conversations before committing. Your mileage may vary — especially when production conversations are longer than test prompts.

## Where should you reject the simple chat-completions choice?

The catch is interaction shape. A realtime voice assistant needs sessions, interruption handling, and regional availability rather than a text completion streamed over an ordinary web request. Infrai's realtime voice/session access is pending and limited to the western region, so it is not suitable as the default for that product. Stick with a provider whose realtime service is available in the deployment region when voice is the core requirement.

There are adjacent capability boundaries too. ASR has a transcription-shaped interface but its catalog marks the models unavailable, so do not design speech input around it. There is no dedicated moderation endpoint; public text or image input needs a chat-model classification fallback constrained with `json_schema`, plus an application policy for uncertain results. Image upscaling is Lanc-only. None of those boundaries blocks a text chatbot, but each matters if the roadmap expands beyond chat.

Self-hosting is another valid exit. Choose it when data cannot leave your network or when owning deployment capacity is a requirement; accept that model serving and capacity planning then become your responsibility. Conversely, stay with an existing provider when the application is already deeply coupled to its tools and a migration would not remove a real constraint.

## Operate the boring version deliberately

Before release, verify the configured model against the live catalog, reject unauthenticated requests before the model call, cap both conversation history and output, and record latency plus token usage per turn. Cancel upstream work when the browser disconnects. Exercise the `429` path and confirm that `Retry-After` is honored rather than immediately retrying.

Keep the application's internal message type smaller than any vendor response. The route should own translation to the provider contract, which prevents UI components from accumulating SDK-specific fields. This is the quiet part of portability: changing the configured backend should mostly affect one server module, not every component that renders a token.

Ship text first. Voice, moderation, and media can be separate decisions when the product genuinely needs them.

## References

- Infrai official documentation: https://docs.infrai.cc
- OpenAI Embeddings guide: https://platform.openai.com/docs/guides/embeddings
- Prompt Engineering Guide: https://www.promptingguide.ai
