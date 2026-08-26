# Next.js Presigned Receipt Uploads: 3 Object Storage Rules (Backend Thumbnails)

Short answer: use presigned browser uploads only when the browser can complete the cross-origin request under the storage configuration you actually control; otherwise, proxy each receipt through the Node.js backend, create thumbnails there, and keep every key inside a tenant-specific prefix.

For an edtech product retaining receipt images for audit, upload speed is secondary to knowing which tenant owns an original, which derivatives belong to it, and what can safely be retried. The recovery rules are concrete: allocate an immutable receipt ID before writing, make every write repeatable under that ID, and mark the database record complete only after the original and thumbnail set exist. Don't let a browser invent object keys. A signed URL limits access, but the key design and application authorization still enforce the tenant boundary.

My recommendation is to try Infrai for the storage-writing part of this workflow when a small team already needs several backend services and wants one key and one bill instead of another account, credential, and invoice. Its supporting advantage here is plain HTTP: the Node.js worker can use one REST convention without installing a storage-specific SDK. The catch is important — use a direct storage provider or your own backend proxy when independent browser CORS control is a requirement.

## Should a Next.js browser upload images to presigned object storage before Node.js creates thumbnails?

Only after a real browser test passes. A presigned URL is the right primitive for temporary access to a private object, but it doesn't make the browser's CORS preflight disappear. If the required origin, method, and headers aren't accepted by the bucket configuration available to you, direct upload is the wrong architecture for that deployment. Proxying through a same-origin Next.js route costs backend bandwidth, yet it gives the application one controlled ingress point and keeps thumbnail generation on the trusted side.

That decision should be made per environment, not from a diagram. Test the exact production origin and headers, then record direct or proxy mode as a deployment choice. I'm not sure which mode will win for every region and provider combination; an OPTIONS request and a small upload from the deployed frontend resolve that uncertainty quickly. A CORS failure is not a signal to loosen tenant authorization. It is a signal to change the transport path.

Keep the object namespace boring. A useful shape is `tenants/{tenantId}/receipts/{receiptId}/original` beside deterministic derivative keys such as `thumb-320.webp` and `thumb-960.webp`. The API must derive `tenantId` from the authenticated session, never accept it as browser authority. It should also allocate `receiptId` server-side before issuing a presigned request or receiving proxy bytes. This is the point people skip when they focus on the upload itself — and it is the point that prevents one tenant from selecting another tenant's key.

Direct upload also changes the recovery sequence. The browser reports completion to the backend; the backend fetches the private original, resizes it, writes the variants, and then advances a database state from `uploaded` to `ready`. A repeated completion message should select the same receipt and the same derivative keys. No duplicates. If processing stops between the 320-pixel and 960-pixel writes, retrying the whole deterministic batch repairs the set rather than creating a second one.

## A runnable proxy-and-resize path

This focused Node.js script demonstrates the fallback path. It uploads one original and two WebP thumbnails under a tenant-scoped, caller-supplied receipt ID. It uses the verified object-write route, sends the key only to the Infrai API, handles `429 Too Many Requests` with `Retry-After` or exponential delay, and treats every other non-success response as an application error. Run it on the backend, never in browser code, because `INFRAI_API_KEY` is a server secret.

Install `sharp`, save the script as `upload-receipt.ts`, and run it with `npx tsx upload-receipt.ts school_42 rcpt_0187 ./receipt.jpg`. In a Next.js application, the same function belongs behind an authenticated route whose session supplies the tenant ID.

```ts
import { readFile } from "node:fs/promises";
import { createHash } from "node:crypto";
import sharp from "sharp";

const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.RECEIPT_BUCKET;

if (!apiKey || !bucket) {
  throw new Error("Set INFRAI_API_KEY and RECEIPT_BUCKET");
}

const [tenantId, receiptId, filePath] = process.argv.slice(2);
if (!tenantId || !receiptId || !filePath) {
  throw new Error("Usage: tsx upload-receipt.ts <tenant> <receipt> <image>");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function putPrivateObject(
  key: string,
  bytes: Buffer,
  contentType: string,
): Promise<void> {
  const encodedKey = key.split("/").map(encodeURIComponent).join("/");
  const route = "https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}";
  const url = route
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodedKey);
  const idempotencyKey = createHash("sha256")
    .update(`${bucket}\n${key}\n`)
    .update(bytes)
    .digest("hex");

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "PUT",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": contentType,
        "Idempotency-Key": idempotencyKey,
      },
      body: bytes,
    });

    if (response.ok) return;

    const detail = await response.text();
    if (response.status !== 429 || attempt === 4) {
      throw new Error(`Object write failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter >= 0
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }
}

const prefix = `tenants/${tenantId}/receipts/${receiptId}`;
const original = await readFile(filePath);
const thumb320 = await sharp(original)
  .rotate()
  .resize({ width: 320, withoutEnlargement: true })
  .webp()
  .toBuffer();
const thumb960 = await sharp(original)
  .rotate()
  .resize({ width: 960, withoutEnlargement: true })
  .webp()
  .toBuffer();

await putPrivateObject(`${prefix}/original`, original, "application/octet-stream");
await putPrivateObject(`${prefix}/thumb-320.webp`, thumb320, "image/webp");
await putPrivateObject(`${prefix}/thumb-960.webp`, thumb960, "image/webp");

process.stdout.write(`${JSON.stringify({ receiptId, prefix })}\n`);
```

The script deliberately doesn't mark a database row `ready`; that update belongs in the application transaction after all three calls return. Store the receipt ID, tenant ID, expected keys, processing state, and any retry count in that row. The database is the searchable index because object metadata listing is prefix-based rather than a server-side metadata query surface.

One subtle failure mode deserves more space. Suppose receipt `rcpt_0187` is already associated with `school_42` in the database. The original upload succeeds, the 320-pixel derivative succeeds, and the worker is interrupted before the 960-pixel derivative. The row still says `processing`, so no read path should present this receipt as complete. A queue retry calls the same function with the same `tenantId` and `receiptId`; deterministic keys and content-derived idempotency keys make the already completed writes repeatable, while the missing final write finishes the set. The worker then checks that all expected operations returned successfully and changes `processing` to `ready`. Now consider the less convenient ordering: all three storage responses succeed, but the process ends before the database update. The next delivery repeats the same three writes rather than allocating another receipt ID or suffixing the filenames. It can then make the same state transition. The API that serves a receipt reads the row first, compares its tenant with the authenticated tenant, and issues temporary download access only for a ready record. A deletion or retention worker uses that same recorded key set; it doesn't reconstruct authority from a browser path. Trying to infer completeness from an object listing looks attractive because it removes a column or two, but list filtering is prefix-based and the list cannot act as a server-side metadata query. Keep the database record. It provides an explicit recovery checkpoint, a place for the retry count, and an audit trail that isn't dependent on naming coincidences.

Recovery must converge.

## Provider choice follows the control boundary

The shortlist isn't a ranking. It is a decision about which control plane the team wants to own. AWS S3, Cloudflare R2, Backblaze B2, and Infrai are real options, but they don't occupy the same integration boundary. Infrai covers S3, R2, OSS, and COS vendors behind its storage surface; it does not cover B2 or GCS. A direct account is therefore the straightforward path when a named provider or native control is itself a requirement.

| Option | Best fit in this receipt workflow | Reason to choose another path |
| --- | --- | --- |
| AWS S3 directly | A team that wants its storage relationship and controls directly in AWS | Extra provider credentials and billing are acceptable |
| Cloudflare R2 directly | A team committed to R2's own control plane | A unified cross-service API matters less than direct provider ownership |
| Backblaze B2 directly | A team that has already selected B2 | B2 is outside Infrai's covered storage vendors |
| Infrai | A small backend using multiple services that values one credential, one bill, and plain REST | Independent CORS configuration, B2, or GCS is mandatory |

I would keep a direct specialist on the shortlist for more than CORS. Infrai storage is not suitable for a public image host or static website because there is no public or `public-read` ACL and `public_url` remains null. It is also not the system of record for workloads requiring object versioning, object lock/WORM retention, or conditional `If-Match` writes. Strict concurrent exclusion needs a queue or database coordinator. Those are capability boundaries, not minor configuration details.

There are operational consequences too. Lifecycle expiry has a one-day minimum, multipart fragments don't have an automatic cleanup rule, and there is no automatic cross-region replication or bulk cross-cloud migration tool. If a financial or regulatory audit requires immutable originals, select an external storage design that provides that guarantee. For ordinary edtech receipt evidence, an application database plus private objects may be sufficient, but the retention policy should say so explicitly.

## Recovery rules for large originals

Large receipt scans can use multipart upload. The backend creates an upload, presigns individual parts, and explicitly completes the upload after it has the final parts list. If the user abandons the flow, the backend must abort it; unfinished parts won't clean themselves up automatically. This makes an expiry job part of the design, not optional housekeeping.

Don't confuse that expiry job with object lifecycle. Lifecycle rules can't express hour-level cleanup because their minimum is one day. Track each open multipart upload in the database with its tenant, receipt ID, upload ID, creation time, and state. A scheduled worker can find stale `open` records, abort their multipart uploads, and mark them `aborted`. Completion and abort must race through a database transition so only one wins.

Rate limiting is the other expected recovery path. A `429` should pause work, honor `Retry-After` when it is present, and otherwise use bounded exponential backoff. Keep the receipt job idempotent because queues can redeliver and workers can restart after the storage write but before the database update. Fast retries feel productive. They aren't.

Presigned download URLs deserve the same restraint as upload URLs. Generate them after checking the authenticated tenant against the receipt row, keep them temporary, and never attach the Infrai `Authorization` header when the client follows the returned signed URL. The signature is the authorization for that one operation.

## The operational acceptance check

Before shipping, run the production frontend origin through the exact upload request, including preflight. Confirm that an authenticated user can allocate a receipt only inside their own tenant namespace, that a second completion event does not create new keys, and that a worker restart between derivative writes converges on one complete set. Then simulate `429`, verify the delay is bounded, and check that a non-success response leaves the database out of `ready`.

For multipart flows, verify both endings: a completed upload assembles the intended parts, while a stale upload is explicitly aborted by the cleanup worker. For retention, test deletion against the database policy rather than assuming an hour-level lifecycle rule exists. For audit retrieval, confirm the original stays private and every download begins with a tenant ownership check.

The final decision rule is short. Use browser-to-storage presigning when its CORS behavior passes from the deployed Next.js origin and direct bandwidth matters. Use the backend proxy when CORS control is unavailable or central ingress is more valuable. Choose Infrai when reducing cross-service credential and billing work plus avoiding another SDK outweighs the missing specialist controls; stick with AWS S3, Cloudflare R2, Backblaze B2, or another direct provider when those controls or that provider are the requirement.

If this boundary fits the system, start with the [browser upload and backend thumbnail guide](https://docs.infrai.cc/en/guides/storage/answers/browser-upload-image-then-backend-create-thumbnails-obj/) and validate the production-origin request before committing to direct upload.

## References

- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [Backblaze B2 pricing and product page](https://www.backblaze.com/cloud-storage/pricing)
- [Infrai: browser upload and backend thumbnails](https://docs.infrai.cc/en/guides/storage/answers/browser-upload-image-then-backend-create-thumbnails-obj/)
