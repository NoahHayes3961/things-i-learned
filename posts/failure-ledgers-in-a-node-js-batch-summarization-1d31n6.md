# Failure Ledgers in a Node.js Batch Summarization API: Auditable Results

**Short answer:** Build the Node.js batch summarization API as an asynchronous, per-document state machine: return a job ID immediately, cap worker concurrency, persist each result, and export both successes and failures without regenerating completed work.

The key trade-off is extra bookkeeping in exchange for bounded retries, visible progress, and no need to regenerate successful summaries when one input fails.

For a small Node.js service, I would start with one persisted job record, a capped worker pool, and newline-delimited JSON output. Don't begin with a workflow platform unless the product already needs one. The data flow is plain: `POST /jobs` stores input and answers `202`; a worker claims pending items; `GET /jobs/:id` reports counts; and `GET /jobs/:id/results` streams the completed rows. The model call sits behind one narrow function, so its wire format can change without changing job state or export semantics.

## How should a Node.js API summarize multiple documents and export results?

Treat the document, rather than the submitted collection, as the retry and accounting unit. Each item needs a stable ID and one of four states: `pending`, `running`, `succeeded`, or `failed`. A job is complete when no item remains pending or running, not when every item succeeded. That distinction prevents one permanently bad input from holding an otherwise useful export open forever.

Keep two identifiers. The client supplies an idempotency key for the submission, while the service assigns an item ID to every document. Repeating the same submission should return the existing job; retrying an item should update that item. A summary without an item ID is hard to reconcile after a restart, and a retry without an idempotency key can quietly create duplicate model calls.

The export should preserve failures as rows. Omitting them produces a clean-looking file whose row count no longer matches the input. A compact result record can carry `id`, `status`, `summary`, `attempts`, and a machine-readable error category. It should not store credentials, and it usually should not echo the full source document.

This is deliberately boring.

Ship that first.

## A runnable state-machine example

The example below keeps state in memory so the job transitions are visible in one file. Its `summarize` dependency is the adapter boundary: production code can connect that function to an approved model API, while tests can supply a deterministic implementation. The public routes belong to this sample service, not to any external provider.

```ts
import { createServer, IncomingMessage, ServerResponse } from "node:http";
import { randomUUID } from "node:crypto";

type InputDocument = { id: string; text: string };
type Item = InputDocument & {
  status: "pending" | "running" | "succeeded" | "failed";
  attempts: number;
  summary?: string;
  error?: "invalid_input" | "temporarily_unavailable" | "rejected";
};
type Job = { id: string; idempotencyKey: string; items: Item[] };
type Summarize = (text: string) => Promise<string>;

const jobs = new Map<string, Job>();
const submissions = new Map<string, string>();
const concurrency = 4;

async function readJson(req: IncomingMessage): Promise<unknown> {
  const chunks: Buffer[] = [];
  for await (const chunk of req) chunks.push(Buffer.from(chunk));
  return JSON.parse(Buffer.concat(chunks).toString("utf8"));
}

function send(res: ServerResponse, status: number, body: unknown): void {
  res.writeHead(status, { "content-type": "application/json" });
  res.end(JSON.stringify(body));
}

function counts(job: Job) {
  return job.items.reduce(
    (n, item) => ({ ...n, [item.status]: n[item.status] + 1 }),
    { pending: 0, running: 0, succeeded: 0, failed: 0 },
  );
}

async function run(job: Job, summarize: Summarize): Promise<void> {
  let cursor = 0;
  const workers = Array.from({ length: concurrency }, async () => {
    while (cursor < job.items.length) {
      const item = job.items[cursor++];
      if (item.status !== "pending") continue;
      item.status = "running";
      item.attempts += 1;
      try {
        const summary = (await summarize(item.text)).trim();
        if (!summary) throw new Error("rejected");
        item.summary = summary;
        item.status = "succeeded";
      } catch (error) {
        item.status = "failed";
        item.error = error instanceof Error && error.message === "rejected"
          ? "rejected"
          : "temporarily_unavailable";
      }
    }
  });
  await Promise.all(workers);
}

const summarize: Summarize = async (text) => {
  // Deterministic local adapter for running the example; replace at this boundary.
  const sentences = text.match(/[^.!?]+[.!?]+/g) ?? [text];
  return sentences.slice(0, 2).join(" ").trim();
};

createServer(async (req, res) => {
  const url = new URL(req.url ?? "/", "http://localhost");

  if (req.method === "POST" && url.pathname === "/jobs") {
    const key = req.headers["idempotency-key"];
    if (typeof key !== "string" || key.length === 0) {
      return send(res, 400, { error: "missing_idempotency_key" });
    }
    const existing = submissions.get(key);
    if (existing) return send(res, 200, { jobId: existing });

    const body = await readJson(req) as { documents?: InputDocument[] };
    if (!Array.isArray(body.documents) || body.documents.length === 0) {
      return send(res, 400, { error: "invalid_documents" });
    }
    const job: Job = {
      id: randomUUID(),
      idempotencyKey: key,
      items: body.documents.map((d) => ({ ...d, status: "pending", attempts: 0 })),
    };
    jobs.set(job.id, job);
    submissions.set(key, job.id);
    void run(job, summarize);
    return send(res, 202, { jobId: job.id });
  }

  const match = url.pathname.match(/^\/jobs\/([^/]+)(\/results)?$/);
  const job = match ? jobs.get(match[1]) : undefined;
  if (req.method !== "GET" || !job) return send(res, 404, { error: "not_found" });

  if (match?.[2] === "/results") {
    res.writeHead(200, { "content-type": "application/x-ndjson" });
    for (const { id, status, summary, attempts, error } of job.items) {
      if (status === "succeeded" || status === "failed") {
        res.write(JSON.stringify({ id, status, summary, attempts, error }) + "\n");
      }
    }
    return res.end();
  }
  return send(res, 200, { jobId: job.id, counts: counts(job) });
}).listen(3000);
```

Run it with a current Node.js release that supports TypeScript execution in your chosen toolchain, then submit unique document IDs and an `Idempotency-Key` header. In a real deployment, replace the maps before replacing anything else. A process restart erases them, so durable storage must own the job, items, idempotency mapping, and attempt count. The worker should claim items atomically; otherwise two worker processes can both observe `pending` and pay to summarize the same text.

The sample marks an unsuccessful adapter call as terminal because retry policy depends on the external API contract. Production code should split failures into categories before deciding what to retry. Authentication and invalid input are terminal. Rate limits and transient network failures may be retried, but only with a cap, delay, and jitter. I'm not sure what cap fits your workload; the answer depends on the service's documented response semantics and the latency budget you can tolerate. Measure it instead of inheriting an arbitrary five-retry helper.

## Failure accounting matters more than queue choice

The dangerous state is neither `failed` nor `running`. It is a false success: an adapter returns an empty string, a parser accepts it, and the job increments `succeeded`. The dashboard looks healthy while the export contains holes. Consider what follows after a burst of HTTP `429` responses: a generic retry helper exhausts its attempts but returns the last response body, the adapter treats a missing summary field as an empty string, and the worker records the item as complete because no exception escaped. The job count reaches the submitted count, yet the downloaded file has blank summaries. Nothing in that chain requires an outage or a corrupt queue; it only requires three layers to disagree about what success means. Validate the summary before the state transition, make retry exhaustion an explicit result, and calculate job counts from persisted item states. At minimum, reject empty output and retain an error category; for structured summaries, validate the schema and record the parser version as well. This is also why an export must include failed IDs rather than quietly filtering them out: reconciliation should expose the mismatch at the boundary where another system can act on it.

No silent drops.

Retries create a second accounting trap. Suppose an item times out after the remote system accepted it. Retrying may perform the same paid work twice, even if the local database records one final summary. An idempotency facility from the downstream API can prevent that when available. Without one, the honest design is at-least-once execution plus duplicate-call telemetry, not a claim of exactly-once behavior.

Token limits belong at ingestion, not deep inside the adapter. Count tokens with the tokenizer appropriate to the selected model when one is available; `tiktoken`, for example, is a BPE tokenizer library and exposes model-aware encoding helpers. A character slice is not an equivalent token budget. If a document exceeds the chosen input ceiling, either reject it with a useful reason or apply an explicit chunk-and-reduce policy. Silent truncation makes summaries difficult to trust because the export does not reveal which part of the source disappeared.

Chunking changes the task. Summarizing chunks independently and summarizing those summaries is useful for long inputs, but global details can be lost at either stage. Store the chunk boundaries and strategy version with the item so a later evaluation can distinguish a model change from a preprocessing change. Your mileage may vary across legal text, support tickets, and meeting transcripts; a held-out evaluation set resolves more uncertainty than another prompt adjective.

Backpressure is the remaining control. Fixed concurrency is a reasonable first release, but it is not a promise that four requests always fit. Watch the rate-limit response category, queue age, adapter latency, and tokens per successful item. Reduce concurrency when throttling persists; raise it cautiously when queue age grows and the downstream contract permits more traffic. Fast is good. Predictable is better.

## Choosing the smallest suitable execution model

| Execution model | Suitable when | The catch |
| --- | --- | --- |
| Bounded work in the request | The input is tiny and completion fits comfortably inside the caller's time budget | A disconnected or retried request can complicate ownership of unfinished work |
| Database-backed worker | The product needs per-item progress, prompt results, and controlled retries | You own claiming, leases, recovery, and metrics |
| Provider-managed batch | Turnaround is flexible and its documented file contract matches the task | Progress and retry controls may be coarser than per-item orchestration |
| Durable workflow engine | Long-running jobs and timers are already common across the product | It adds an operational dependency and migration surface |

For an indie application, the database-backed worker is often the useful middle, but it is not a universal recommendation. Keep bounded request work when there are only a few small inputs and the synchronous experience is materially simpler. Choose provider-managed batching when latency is unimportant and the published contract removes enough scheduling work to matter. Use a durable workflow engine when recovery, timers, and multi-step coordination are already platform concerns; adopting one for a single short queue can be more machinery than the feature deserves.

Cost is mostly controlled before a request leaves the worker. Deduplicate identical content, cap accepted tokens, avoid retrying terminal failures, and persist successful output immediately. Record input tokens, output tokens, attempts, and latency per item if the adapter provides those measurements. Aggregate percentiles and totals by prompt version. An invoice can tell you that usage rose; these fields can tell you whether documents got longer, retries climbed, or a prompt began producing more output.

Before shipping, exercise the state machine rather than merely testing the happy response. Submit the same idempotency key twice and verify that one job exists. Stop a worker after it claims an item, restart it after the lease expires, and confirm that completed items stay completed. Feed the adapter an empty result, a terminal rejection, and a retryable response. Confirm that an export made during processing contains only terminal rows, then confirm that the final exported IDs reconcile exactly with the submitted IDs. Finally, compare summaries against a small, fixed evaluation set whenever the prompt, model, tokenizer, or chunking policy changes. That is the operational checklist: identity, recovery, classification, reconciliation, and quality drift — in that order.

## Sources

- https://github.com/openai/tiktoken
- https://elevenlabs.io/docs
