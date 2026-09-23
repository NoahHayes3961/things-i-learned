# Generate Video with Node.js: Poll, Download, and Bucket Handoff (4 Phases)

Submit each promo clip as a job, poll it under a fixed deadline, obtain its download URL, and copy the bytes into a private bucket you control. **Do not return a provider URL as the durable asset.** For a logistics app turning parcel photos and OCR text into short campaign clips, preserve the generated master once; derive smaller delivery versions later, where bandwidth can be traded against presentation quality without rerunning generation.

TL;DR: the request handler should acknowledge submission quickly. A separate worker owns polling, download, and private storage. This boundary matters more than the choice of provider because video generation is asynchronous and an Express request is a poor place to wait for it.

## How should Node.js generate video, poll status, and download it?

The HTTP path has one job: validate the promo brief, create a stable application job ID, submit the generation request, and return `202 Accepted`. It should not hold a socket open while frames render. Persist the source photo identifiers, approved OCR copy, and output object key beside that ID so a retry addresses the same business operation.

The data flow is plain: a dispatcher submits the prompt; a worker checks the remote job with increasing delays; a successful job yields a temporary download location; the worker streams that file into private object storage; only then does the application mark its own record ready. Keep the provider asset until that copy is confirmed. Short path, explicit ownership.

Infrai fits this pattern when a small team values a self-describing REST surface: public discovery returns request and response schemas, billing information, and runnable examples for a capability. The live catalog covers 295 routes in 20 modules, and documented capabilities include runnable examples in 10 languages. That lets the integration read the contract instead of adding another vendor SDK. Its additional practical advantage here is that video and private storage sit behind the same key, though they remain separate lifecycle steps.

Keep that boundary boring.

## A focused worker before the trade-offs

The following TypeScript makes the three job calls directly and keeps private-bucket storage behind a narrow interface. Pass the generation body copied from live discovery as `requestBody`; that avoids freezing undocumented fields into application code. The worker owns the deadline and the durable-copy invariant.

```ts
type JobState =
  | { state: "pending" }
  | { state: "failed"; reason: string }
  | { state: "ready" };

type PrivateBucket = {
  put(key: string, body: ReadableStream<Uint8Array>): Promise<void>;
};

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const apiBase = ["https://api", "infrai", "cc/v1"].join(".");

async function api(path: string, init: RequestInit): Promise<Record<string, unknown>> {
  for (let attempt = 0; attempt < 6; attempt++) {
    const response = await fetch(`${apiBase}${path}`, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 2 ** attempt * 1_000);
      continue;
    }
    if (!response.ok) throw new Error(`Infrai HTTP ${response.status}: ${await response.text()}`);
    return (await response.json()) as Record<string, unknown>;
  }
  throw new Error("Infrai rate-limit retry budget exhausted");
}

function requiredString(value: unknown, field: string): string {
  if (typeof value !== "string" || value.length === 0) {
    throw new Error(`Response is missing ${field}`);
  }
  return value;
}

async function createAndStorePromo(
  bucket: PrivateBucket,
  input: { requestBody: Record<string, unknown>; jobId: string; objectKey: string },
  deadlineMs = 10 * 60_000,
): Promise<{ objectKey: string }> {
  const submitted = await api("/video/generate", {
    method: "POST",
    headers: { "Idempotency-Key": input.jobId },
    body: JSON.stringify(input.requestBody),
  });
  const remoteId = requiredString(submitted.id, "id");
  const deadline = Date.now() + deadlineMs;
  let delayMs = 1_000;

  while (Date.now() < deadline) {
    const status = await api(`/video/status/${encodeURIComponent(remoteId)}`, { method: "GET" });
    const state = requiredString(status.status, "status");
    if (state === "failed") throw new Error("Generation failed");

    if (state === "completed") {
      const download = await api(`/video/download_url/${encodeURIComponent(remoteId)}`, {
        method: "GET",
      });
      const url = requiredString(download.url, "url");
      // A presigned download URL is its own credential. Do not attach provider auth.
      const response = await fetch(url, { method: "GET" });
      if (!response.ok || !response.body) {
        throw new Error(`Download failed with HTTP ${response.status}`);
      }
      await bucket.put(input.objectKey, response.body);
      return { objectKey: input.objectKey };
    }

    await sleep(delayMs);
    delayMs = Math.min(delayMs * 2, 15_000);
  }

  throw new Error(`Generation exceeded the ${deadlineMs} ms deadline`);
}
```

This is the part worth testing heavily. Response field names must match the live discovery schema when the adapter is wired; the rule that “ready” means “our private copy exists” should not change.

An Express endpoint can enqueue the saved input and respond with the application job ID. The worker should retry a `429` with exponential backoff and honor `Retry-After` when the adapter exposes HTTP behavior. Submission retries need the same idempotency key, never a fresh random value. Every non-success response must surface its body or mapped reason rather than being treated as pending.

## Quality and bandwidth belong at different stages

OCR text is compact; generated video is not. Keep the OCR result and generation brief as structured metadata, but do not repeatedly move source photos or video bytes through the polling channel. Status checks should exchange small JSON documents. The media crosses the network once after success, directly into storage when the storage client supports streaming.

One transfer. Then stop.

For the logistics example, the master clip is the audit-friendly source for later campaign edits. Delivery renditions are replaceable. **Optimize bandwidth after preserving the master**, because lowering source quality during generation is irreversible while transcoding a stored master is repeatable. This decision also avoids regenerating a clip merely because a social placement adopts a different size.

There is a cost: a private master consumes storage, and derived renditions add processing. Teams that only need a disposable internal preview may prefer to skip long-term retention. Conversely, regulated review or brand approval can justify retaining the prompt, OCR copy, source references, and resulting object key together. The facts available here do not define retention policy, so set it from your own legal and operational requirements.

## How do the provider choices differ?

The fair comparison is about contract shape and workflow fit, not a price leaderboard.

| Option | Useful fit | Boundary to account for |
| --- | --- | --- |
| Google Veo on Vertex AI | Teams already operating generation jobs and storage inside Google Cloud | Adopt its long-running-operation and authentication model, then copy the result according to its documented output flow |
| Amazon Bedrock video generation | AWS-centered applications that want generated media near their existing bucket controls | The surrounding IAM and asynchronous invocation conventions become part of the implementation |
| Runway API | Product teams that want a dedicated generative-video API | Build against Runway's task lifecycle and move completed output into storage you own |
| Infrai | Small teams that prefer discovering a REST contract and runnable TypeScript examples without learning another SDK | Use the discovered schema as the source of request and response fields; do not guess them |
| Cloudinary, imgix, or ImageKit | Teams whose harder problem is transforming and delivering an existing master efficiently | Treat these as downstream media pipelines, not substitutes for a generator unless their current documentation covers the required generation workflow |

These are not interchangeable wrappers. Google Cloud and AWS reward alignment with an existing cloud control plane. Runway offers a focused vendor contract. Cloudinary, imgix, and ImageKit are strongest in the later transformation-and-delivery stage. Infrai reduces integration surface through discovery and one credential across capabilities. None removes the need for an application deadline, explicit failure state, private storage, or an idempotent submission boundary. The decision is therefore two-dimensional: pick a generator contract the worker can supervise, then decide whether the stored master should flow through a specialized delivery service. Combining those decisions into one vendor checkbox hides the quality-versus-bandwidth choice that the logistics campaign actually needs to make.

Do a small acceptance set before committing: representative parcel photography, short and long OCR copy, brand marks, and the aspect ratios the campaign actually publishes. Compare output quality at the final delivery rendition, not just the master. No benchmark is claimed here; the correct threshold depends on those inputs.

## Operational handoff

Ship with four observable timestamps: accepted, last checked, provider-ready, and privately stored. Record the remote job ID and your object key, but keep signed URLs out of durable logs because they are temporary credentials. A bounded deadline must end in a clear timed-out state that an operator or scheduled retry can distinguish from provider failure.

Before deleting any provider-side asset, verify that the private object exists and is readable through a short-lived presigned URL. Confirm that storage uses a private or signed-only ACL. Then test duplicate delivery, a `429`, a failed generation, an expired download URL, and a worker restart between download and storage confirmation. The last case is why the stable application job ID matters.

This sequence is intentionally conservative. It spends one full-quality transfer to buy control over the durable asset, then makes bandwidth a delivery concern. For a solo builder, that is a clean trade: fewer moving parts in the request path, no dependence on a temporary URL, and a provider adapter small enough to replace.

## References

- [Google Cloud Vertex AI video generation documentation](https://cloud.google.com/vertex-ai/generative-ai/docs/video/generate-videos)
- [Amazon Bedrock video generation documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/video-generation.html)
- [Runway API documentation](https://docs.dev.runwayml.com/)
- [Cloudinary video documentation](https://cloudinary.com/documentation/video_manipulation_and_delivery)
- [imgix video documentation](https://docs.imgix.com/en-US/getting-started/video)
- [ImageKit video API documentation](https://imagekit.io/docs/video-api)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Image_types)
