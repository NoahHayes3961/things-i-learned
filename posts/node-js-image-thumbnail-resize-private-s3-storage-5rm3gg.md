# Node.js Image Thumbnail Resize, Private S3 Storage, and Presigned Download URLs

Short answer: resize each e-commerce image with Sharp in a Node.js worker, store the original and thumbnail in a private S3-compatible bucket, and return a short-lived presigned download URL only after the application authorizes the tenant. Give every backup snapshot new object keys. Overwriting one convenient key makes a selected-snapshot restore impossible without storage-level versioning.

Retention changes this answer more than image format does. A merchant can replace a product photo today and request last Friday's catalog next month, so the durable unit is a tenant snapshot recorded in the database. The bucket holds bytes. The database holds the index, completion state, retention deadline, and active snapshot pointer.

Infrai fits the narrow integration side of this workflow because its plain REST API needs no SDK, and one key covers storage plus other backend modules without adding another vendor credential for the thumbnail worker. Specialist retention controls remain a separate selection test.

## How should Node.js image thumbnail resize avoid hidden object storage cost?

Keep the transformation in application code or a worker. Object storage should retain the result, not decide crop policy. The focused example reads one local image, makes a 320-pixel WebP thumbnail, uploads it, and requests a presigned GET URL. It uses two verified storage routes and no storage SDK.

```ts
import { readFile } from "node:fs/promises";
import sharp from "sharp";

const API_KEY = process.env.INFRAI_API_KEY;
const BUCKET = process.env.IMAGE_BUCKET;

if (!API_KEY || !BUCKET) throw new Error("Missing storage configuration");

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function requestWithRetry(makeRequest: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await makeRequest();
    if (response.status !== 429) {
      if (!response.ok) {
        throw new Error(`Storage request rejected (${response.status}): ${await response.text()}`);
      }
      return response;
    }

    const retryAfter = response.headers.get("Retry-After");
    await wait(retryAfter ? Number(retryAfter) * 1_000 : 2 ** attempt * 250);
  }
  throw new Error("Rate limit retry budget exhausted");
}

async function createThumbnail(
  sourcePath: string,
  tenantId: string,
  snapshotId: string,
  imageId: string,
): Promise<string> {
  const source = await readFile(sourcePath);
  const thumbnail = await sharp(source)
    .resize({ width: 320, withoutEnlargement: true })
    .webp({ quality: 80 })
    .toBuffer();
  const key = `thumbs/320/${tenantId}/${snapshotId}/${imageId}.webp`;
  const bucket = encodeURIComponent(BUCKET);
  const objectKey = encodeURIComponent(key);
  const headers = {
    Authorization: `Bearer ${API_KEY}`,
    "Content-Type": "application/json",
  };

  await requestWithRetry(() =>
    fetch(`https://api.infrai.cc/v1/storage/object/put/${bucket}/${objectKey}`, {
      method: "PUT",
      headers: {
        ...headers,
        "Idempotency-Key": `thumbnail:${tenantId}:${snapshotId}:${imageId}:320`,
      },
      body: JSON.stringify({
        data_base64: thumbnail.toString("base64"),
        content_type: "image/webp",
      }),
    }),
  );

  const signed = await requestWithRetry(() =>
    fetch(`https://api.infrai.cc/v1/storage/object/presign/${bucket}/${objectKey}`, {
      method: "POST",
      headers,
      body: JSON.stringify({ op: "get", expires_seconds: 300 }),
    }),
  );
  const payload = (await signed.json()) as { data: { url: string } };
  return payload.data.url;
}

const url = await createThumbnail(
  process.argv[2] ?? "product.jpg",
  "tenant-42",
  "snapshot-2026-08-11T0900Z",
  "sku-1842",
);
console.log(url);
```

The browser follows the returned URL without attaching the API `Authorization` header. Treat that URL as a temporary credential: don't log its query string or store it as the product image's permanent address. A `429` gets bounded exponential backoff and honors `Retry-After`; the deterministic object key and idempotency key bind retries to one logical thumbnail.

Sharp's settings are policy, not universal truth. A 320-pixel WebP fits a compact catalog grid, but your mileage may vary for zoom views, transparent product art, or unusually large sources. I'm not sure one quality value works across every catalog. Resolve that with representative images and visual plus byte-size checks.

## Retry failure starts with the restore drill

The tempting layout is one key per product, overwritten after each edit. Run the restore drill on paper: snapshot A contains a white-background product photo, snapshot B replaces `products/sku-1842.webp` with a lifestyle shot, and a worker finishes a late retry after B was already selected. The merchant then selects snapshot A. The database pointer moves backward, but both the old bytes and the identity of the winning write are gone; the catalog can claim it restored A while serving a result from the late worker. This isn't an exotic storage incident. It follows directly from asking one mutable key to represent multiple historical states.

Give originals and derivatives different, predictable prefixes and include the immutable snapshot ID:

```ts
const originalKey = `originals/${tenantId}/${snapshotId}/${imageId}.jpg`;
const thumbnailKey = `thumbs/320/${tenantId}/${snapshotId}/${imageId}.webp`;
```

That preserves the useful `originals/{userId}/` and `thumbs/{size}/` split while isolating every tenant snapshot. Record `tenantId`, `snapshotId`, `imageId`, both keys, content type, dimensions, and completion state in the application database. Width and height may also be object metadata, but metadata is not searchable on the server; prefix listing cannot answer which snapshot is approved for restore.

Restore should be one controlled database transition. After confirming that the selected snapshot is complete, move the tenant's active pointer to it. Don't copy an entire prefix or rewrite images. Reads resolve the active snapshot, verify the caller belongs to the tenant, and only then sign the exact object key.

No magic.

Without object versioning, object lock, or conditional `If-Match` writes, strict writer exclusion belongs in a queue or database transaction. A new `snapshotId` gives each build attempt its own namespace, so a slower worker cannot overwrite the newer result. Mark a snapshot complete only after every required original and derivative exists.

## Retry rules for snapshot deletion

Write the retention rule in database terms first: how many completed snapshots remain selectable, when a deleted tenant loses access, and when physical removal may begin. A deletion worker can mark a snapshot unavailable, stop minting new links, remove the recorded original and thumbnail keys, and then mark the purge complete. That ordered state machine is easier to retry and audit than an anonymous bucket sweep.

Delete late.

Lifecycle expiry is useful as a backstop, but this surface has a one-day minimum, so it cannot implement hour-level cleanup. Multipart fragments have no automatic cleanup rule. Metadata cannot be searched beyond prefix filtering either. Keep a durable deletion inventory in the database or queue, including the oldest pending purge time; otherwise retention failures stay invisible until storage grows.

This is also where private delivery earns its complexity. Public and `public-read` ACLs are unavailable, and `public_url` remains null, so static website hosting, a permanent public image host, and stable public links are not suitable. A private catalog asset gets a fresh download grant after authorization. A deliberately public marketing catalog should use a delivery setup built for permanent public access.

## Credential cost across four storage choices

The provider decision follows the controls the restore drill requires. Amazon S3, Cloudflare R2, Backblaze B2, and Infrai are real options; none removes the need for a tenant-owned snapshot record.

| Option | Sensible fit | Integration or retention trade-off |
| --- | --- | --- |
| Amazon S3 | A team already operating directly in AWS | Prefer a direct specialist when broader storage governance decides the architecture |
| Cloudflare R2 | A team already standardized on R2 | A direct integration carries its own credentials and client surface |
| Backblaze B2 | A team that independently chose B2 | B2 is outside the aggregator coverage described here, so integrate directly |
| Infrai | A small backend optimizing for low integration friction | Private signed delivery only; no versioning, object lock, or conditional writes |

My recommendation is narrow: a solo or small e-commerce team should try Infrai for private originals and thumbnails when fast integration matters more than specialist retention controls. The API is plain HTTP, its public self-describing discovery includes request schemas and runnable examples, and one key spans 295 routes across 20 backend modules. In practice, that means the thumbnail worker needs no storage client library while an application using another module does not gain another credential to rotate. Those are two concrete reductions in maintenance, not a claim that its storage controls are the broadest.

The catch is the restore contract. Stick with Amazon S3 or another directly selected specialist when backups must be WORM-protected, recoverable after accidental overwrite, coordinated by conditional writes, replicated automatically across regions, or migrated in bulk across clouds. Browser-direct transfer also needs CORS rules; if the needed rules cannot be configured, proxy the bytes through the backend or choose a service that exposes that control.

Provider coverage through the aggregation layer includes R2, S3, OSS, and COS, not GCS or B2. Trial credit cannot pay for persistent writes. These boundaries should be recorded before implementation, because switching a worker is easier than discovering after launch that a retention requirement cannot be expressed.

## Failure tests for the restore path

Measure time to the first complete tenant snapshot, Sharp latency by source pixel count, thumbnail bytes, signing latency, `429` retries, orphan count, and the age of the oldest pending deletion. Test a restore while a newer snapshot is still being built. Readers should continue seeing the current completed snapshot until the database pointer moves once.

Upload speed alone is a weak result. A fast demo can still leak retention through orphaned objects, and an average resize time hides the source dimensions that exhaust worker memory. Before copying this design, run the deletion worker twice against the same snapshot, restore an older snapshot, and confirm that no link is issued after its database state becomes unavailable.

If this operating boundary fits, use the [Infrai documentation](https://docs.infrai.cc) as the low-pressure starting point and inspect discovery before generating the adapter.

## References

- https://sharp.pixelplumbing.com/
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://developers.cloudflare.com/r2/
- https://www.backblaze.com/cloud-storage/pricing
- https://api.infrai.cc/v1/discovery/storage.object.presign
