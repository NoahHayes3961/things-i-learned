# Property Image Uploads: a Next.js API Route with Sharp and Private Object Storage

**Short answer:** for a property-management app, let a Node.js Next.js API route authenticate the upload, create a small fixed set of Sharp thumbnails, and write the original plus derivatives to private object storage; move the Sharp work to a queue when large-file throughput matters more than an immediate response.

That decision keeps the browser away from storage credentials and keeps the image contract boring. A maintenance coordinator uploads a phone photo of a leaking ceiling. The application records the property and work-order IDs, stores the original under an application-generated key, creates the display sizes, and returns a short-lived link only after authorization. The thumbnail is an object with a known name, not a resize request hidden in a URL.

The important constraint is throughput. A 12 MB photo and a 200 KB preview may belong to the same work order, but they have different effects on request time, memory, and download traffic. I would rather define three supported derivatives than accept arbitrary width and height parameters that turn every image view into an unbounded transformation workload.

## What is the Node.js upload contract for Sharp and private images?

Start with one upload record and one idempotency key. The record owns the relationship between tenant, property, work order, original object, derivative keys, and processing state. The browser can send the file and a client request ID, but it should not choose an object path. A server-generated key prevents a user from selecting another tenant's prefix and gives a retry a stable target.

The synchronous path should be deliberately narrow: authenticate the user, check the declared content type and byte limit, decode the image with Sharp, write the original and fixed derivatives, persist the resulting keys, then create an expiring download link for the authorized view. If a mobile connection makes the original upload slow, that is a reason to measure the upload path and consider direct multipart upload. It is not a reason to make the read path public.

Here is the transformation boundary. The storage adapter is intentionally abstract: the application needs `putObject` and `createDownloadLink`, while the adapter owns provider-specific authentication and request details. The example keeps the work-order context in the key, uses immutable names, and does not allow a caller to request a new size.

```ts
import sharp from "sharp";

type Variant = {
  key: string;
  body: Buffer;
  contentType: "image/webp";
};

type ObjectStore = {
  putObject(input: {
    key: string;
    body: Buffer;
    contentType: string;
  }): Promise<void>;
  createDownloadLink(input: {
    key: string;
    expiresInSeconds: number;
  }): Promise<string>;
};

export async function storeInspectionPhoto(input: {
  store: ObjectStore;
  tenantId: string;
  workOrderId: string;
  uploadId: string;
  original: Buffer;
}) {
  const prefix = `tenants/${input.tenantId}/work-orders/${input.workOrderId}/${input.uploadId}`;
  const variants: Variant[] = [
    {
      key: `${prefix}/thumb-320.webp`,
      body: await sharp(input.original)
        .rotate()
        .resize(320, 320, { fit: "inside", withoutEnlargement: true })
        .webp()
        .toBuffer(),
      contentType: "image/webp",
    },
    {
      key: `${prefix}/preview-1280.webp`,
      body: await sharp(input.original)
        .rotate()
        .resize(1280, 1280, { fit: "inside", withoutEnlargement: true })
        .webp()
        .toBuffer(),
      contentType: "image/webp",
    },
  ];

  await input.store.putObject({
    key: `${prefix}/original`,
    body: input.original,
    contentType: "application/octet-stream",
  });

  for (const variant of variants) {
    await input.store.putObject(variant);
  }

  return {
    originalKey: `${prefix}/original`,
    derivativeKeys: variants.map((variant) => variant.key),
  };
}
```

The code is intentionally not an entire route handler. The route still has to enforce tenant authorization, cap the request body, validate the decoded image rather than trusting only a filename, and save the database row in a transactionally understandable order. A successful response should identify the upload record and the allowed derivative, not expose a bucket-wide listing.

One short rule: fixed variants beat clever URLs.

No resize query.

## Reliable intake needs a large-file throughput budget

Sharp consumes CPU and memory while it decodes and resizes. Running that work inside the request gives the user a simple “uploaded and ready” result, but it ties request concurrency to image processing. For a property app with a handful of inspection photos, that can be a sensible first release. For batches from a field team, the same route can become a queue in disguise, with slow responses and a growing number of open connections.

The practical split is based on the largest expected file, concurrent uploads, and the acceptable time before a preview appears. Keep the upload endpoint responsible for authorization and durable intake. It can store the original, enqueue a job containing the upload ID, and return `202 Accepted` with a processing state. A worker then reads the original, writes the fixed derivatives, and updates the row. This also makes retries explicit: the job can use the upload ID as its idempotency key and safely re-run writes to the same derivative keys.

| Boundary | Good default | Cost to accept |
|---|---|---|
| Request-time Sharp | Small files and an immediate preview promise | API concurrency also pays the decode and resize cost |
| Queued Sharp | Bursty field uploads and large originals | The UI must represent `processing` and eventual readiness |
| Direct original upload | Slow mobile uplinks or high application bandwidth | The authorization handoff and upload policy need their own tests |

The calculation is easy to miss. Suppose a crew uploads ten phone photos at once and each request holds its decoded pixels while two fixed derivatives are produced. The byte size of the compressed file is only one part of the resource budget: decoded dimensions, concurrent Sharp calls, request-body buffering, and the worker's retry behavior all influence whether the API stays responsive. I would start with a deliberately low transformation concurrency, record the p95 intake and ready times, then raise concurrency only while memory remains bounded. If the original is durable before processing begins, a worker retry does not require the field worker to upload the same photo again. That is the useful operational property of the queue, and it is more important than making the first response look synchronous.

Do not use a thumbnail URL as the job's source of truth. Keep the original key and derivative keys in the database, and emit structured events for `received`, `processing`, `ready`, and `rejected`. A missing derivative is then a state transition that can be inspected, rather than a mysterious broken image in a work-order screen.

The choice is not universal. A background worker is not suitable when the product must display a transformed image in the same request and the files are small enough to fit the measured latency budget. Stay synchronous for that narrow case. Use a queue when burst capacity, large files, or predictable API latency matters more than immediate derivative availability. Your mileage may vary because the right threshold depends on the runtime memory limit and the number of simultaneous transformations; measure those values with representative phone images.

## A rollout path for private image processing

Private object storage should remain the default for inspection photos. The application checks the current user's relationship to the tenant and work order before asking the storage adapter for an expiring download link. The link is a capability: anyone who receives it may use it until it expires, so its lifetime should match the screen or download action rather than a general account lifetime.

A browser can download the link directly, which keeps image bytes off a server-rendering path. That does not remove authorization; it moves the authorization decision to the moment the link is issued. Do not persist signed links as permanent fields. Persist object keys, and mint links when the client needs them. This avoids storing access material that will outlive the intended session.

Tenant isolation needs more than a naming convention. Prefixes make review and cleanup understandable, but every read and write still needs an application authorization check. A work-order ID supplied by the client is input, not proof of membership. Test a user from tenant A requesting tenant B's key, including a key that happens to have a valid-looking prefix. The expected result is a denial before link creation.

Public caching is a different requirement. If a property photo must appear on an unauthenticated marketing page, give that public media flow its own policy and catalog decision. Do not weaken private inspection storage because a second screen wants a permanent URL.

Start with one route and one worker contract. Once the state transitions are observable, moving the derivative step out of the request is a deployment change instead of a rewrite of the property UI. A queue is not suitable for a product promise that requires the preview in the same response; keep the synchronous path for that case and set a measured file-size limit.

## Private image access, signed links, and tenant boundaries

## Which failure cases deserve a test before shipping?

The happy path is the least interesting test. I would cover a valid JPEG, a file whose extension says `jpg` but whose bytes are not an image, a file above the request limit, a corrupt image, and two submissions with the same upload ID. I would also test a worker retry after the original write succeeds but before the derivative record is updated. The expected result is the same immutable derivative keys, not a second set of random objects.

Watch four measurements: bytes accepted, time spent decoding, memory during concurrent Sharp calls, and time from intake to `ready`. Log tenant-safe identifiers such as upload ID and work-order ID, but do not log signed links or image bytes. A `413` should be a normal rejected-input metric, while an authorization denial should be distinguishable from a malformed image. That distinction makes capacity work and security review much less speculative.

There are trade-offs. A synchronous route is easier to deploy but couples API capacity to image CPU. A queue absorbs bursts but adds state, worker deployment, and eventual consistency. Direct browser upload can reduce application bandwidth but requires a carefully designed authorization handoff and storage-side upload policy. No single boundary wins every time.

The release checklist is short prose: verify the byte limit at the framework boundary, decode before trusting metadata, write under generated tenant-scoped keys, make retries address the same upload ID, issue links only after an authorization check, and alert when processing stays pending beyond the product's promise. Then load-test the largest realistic photo set with the same concurrency a field crew can create. Ship the smallest measured path; move one responsibility at a time when the numbers demand it.

## References

Further reading:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://cloud.google.com/storage/docs
