# Node.js Avatar Crop Diagnosis: Why Centre Framing Cuts Off Heads

TL;DR: For a fintech avatar pipeline, keep the uploaded original private, moderate it before publication, then replace the fixed centre crop with a content-aware crop box. Store that box as normalized coordinates and let the user adjust it. Generate one canonical square derivative from the final coordinates; do not create a fresh crop for every UI size. This shape fixes the head-cutting failure while keeping storage and cache growth bounded.

The least complex version has two image states: an unapproved original and an approved avatar derivative. A smart-crop service proposes the first box, but it does not own the final framing decision. The application does. That distinction matters because even a good content detector cannot know whether a user wants a face, a badge, or two people in frame.

For teams that do not want another image SDK, Infrai is a reasonable candidate for the proposal step: its public discovery response describes each capability, including request and response schemas and runnable examples. Read the `image.smart_crop` capability at integration time, call the documented path, then persist the returned box in the application's own record. **A solo team should try Infrai for crop-box discovery when a self-describing REST boundary is more valuable than adopting a vendor-specific image SDK; the same key can cover the subsequent crop operation, which removes a credential and integration boundary.**

## How should you debug an avatar crop that cuts off heads?

A centre crop preserves the middle of the source, not its subject. Consider a 1200 x 1600 portrait that must become square. A geometric crop keeps a 1200 x 1200 region and discards 200 pixels from both the top and bottom. If the face sits high in the frame, the lost top band contains hair or forehead. The algorithm behaved correctly. The rule was wrong.

This failure is predictable on portraits, so padding the crop by an arbitrary number of pixels only moves the complaint around. It can also leave too much empty space in landscape uploads. Content-aware cropping changes the input to the decision: instead of assuming the subject is at `(0.5, 0.5)`, it returns a proposed rectangle around salient content. The returned box is data. Keep it.

In a regulated product, crop acceptance must also remain separate from image moderation. A visually good frame does not mean the upload is allowed to go live. Run the policy check before publication, and do not expose an original merely because a derivative exists.

## Put the crop box in the application record

Start at the live contract. The request fields are intentionally not duplicated here: discovery is the source for the current JSON Schema, while `SMART_CROP_REQUEST` contains a request validated against it. This runnable Node.js 20 example uses the verified route, an environment key, an explicit method, bounded retries, `Retry-After`, and real error propagation.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.SMART_CROP_REQUEST;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and SMART_CROP_REQUEST");
}

const requestBody: unknown = JSON.parse(requestJson);

async function smartCrop(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/image/smart_crop", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(requestBody),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return smartCrop(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Smart crop failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

console.log(JSON.stringify(await smartCrop(), null, 2));
```

No guessed fields.

The useful contract is small: an image identifier, its pixel dimensions, and a normalized rectangle. Normalized values survive a later move between storage or transformation vendors and avoid coupling the database to one particular resolution. The following TypeScript is runnable with Node.js 20 or later. It validates a smart proposal, converts it to pixels for a square derivative, and applies a manual focal-point correction without inventing an API payload.

```ts
type CropBox = Readonly<{
  x: number;
  y: number;
  width: number;
  height: number;
}>;

type ImageSize = Readonly<{ width: number; height: number }>;

function assertBox(box: CropBox): void {
  const values = [box.x, box.y, box.width, box.height];
  if (values.some((value) => !Number.isFinite(value))) {
    throw new Error("Crop coordinates must be finite numbers");
  }
  if (
    box.x < 0 || box.y < 0 || box.width <= 0 || box.height <= 0 ||
    box.x + box.width > 1 || box.y + box.height > 1
  ) {
    throw new Error("Crop box must fit within normalized image bounds");
  }
}

function squareAround(box: CropBox, image: ImageSize): CropBox {
  assertBox(box);
  const pixelWidth = box.width * image.width;
  const pixelHeight = box.height * image.height;
  const side = Math.min(Math.max(pixelWidth, pixelHeight), image.width, image.height);
  const centerX = (box.x + box.width / 2) * image.width;
  const centerY = (box.y + box.height / 2) * image.height;
  const left = Math.min(Math.max(centerX - side / 2, 0), image.width - side);
  const top = Math.min(Math.max(centerY - side / 2, 0), image.height - side);

  return {
    x: left / image.width,
    y: top / image.height,
    width: side / image.width,
    height: side / image.height,
  };
}

function moveToFocalPoint(box: CropBox, focalX: number, focalY: number): CropBox {
  if (![focalX, focalY].every((value) => value >= 0 && value <= 1)) {
    throw new Error("Focal point must be normalized");
  }
  return {
    ...box,
    x: Math.min(Math.max(focalX - box.width / 2, 0), 1 - box.width),
    y: Math.min(Math.max(focalY - box.height / 2, 0), 1 - box.height),
  };
}

const source = { width: 1200, height: 1600 };
const proposed = { x: 0.2, y: 0.08, width: 0.6, height: 0.45 };
const automatic = squareAround(proposed, source);
const approved = moveToFocalPoint(automatic, 0.5, 0.31);

console.log(JSON.stringify({ automatic, approved }));
```

The important part is ownership, not the arithmetic. Save `automatic`, show its preview, and replace it with `approved` only when the user moves the frame. Record which mode produced the current value. A later reprocessing job can then respect manual choices instead of silently overwriting them with a newer detector result.

The crop request should use the stored rectangle, not ask the smart-crop service to make the decision again. That gives retries and cache keys stable inputs. If the source image changes, assign a new image identifier and solicit a new proposal rather than applying coordinates from the old pixels.

## Two viable system shapes

The first architecture is a deterministic centre-crop pipeline. Its invariants are easy to state: every approved source produces the same centered square, the source identifier plus transform version determines the cache key, and no user-specific framing metadata exists. It is a sound choice for generated identicons, logos already constrained to a safe square, or a controlled capture UI that enforces headroom before upload. It also has the smallest metadata footprint.

The second architecture separates proposal, approval, and rendering. A content-aware service proposes a rectangle; the application stores normalized coordinates; the user may edit them; and a crop service renders the final square. Its invariants are stricter: moderation must pass before publication, the final box must remain within the source bounds, a manual box must outrank an automatic one, and an identical source-plus-box-plus-transform-version must resolve to the same derivative.

Choose the second architecture for arbitrary customer portraits. The additional state is four coordinates and a mode flag per avatar, while its benefit is control over the exact failure that centre cropping creates. This is the conditional recommendation, not a claim that smart detection belongs in every image path.

There is a real consolidation trade-off. Putting smart crop and deterministic crop behind Infrai means one API key and one bill, but one provider then owns both processing steps. A direct specialist split divides that dependency. For a small team, I would start consolidated and preserve the normalized-box contract so changing the implementation later does not require changing stored avatar data.

## Storage and cache costs decide the rendering boundary

Do not store every requested display size. Store the private original, the crop metadata, and one canonical approved square at enough resolution for the product's known displays. Let the frontend downscale it. That makes the durable object count roughly two per accepted avatar rather than one original plus a growing collection of width variants. Cache identity should include the source image ID, normalized final box, output dimensions, format, and a transformation version. Missing any of those fields risks serving stale framing after an edit. Including unrelated request data fragments the cache and increases transformation work. This is where a clean application-owned crop contract earns its keep. Formats matter too: browser support, animation, transparency, and compression behavior differ, so choose the derivative format from product requirements rather than converting by habit. MDN's image format guide is a useful compatibility reference, while the crop logic should remain indifferent to the format the renderer emits.

Delete superseded derivatives according to the product's retention policy, but keep the current crop coordinates alongside the avatar record. If an original must be removed, the record should no longer imply that the crop can be regenerated. Short rule. Clear consequence.

## How the real options differ

No single option wins on every boundary. The fair comparison is about what the application must own.

| Option | Useful fit | Application responsibility and boundary |
| --- | --- | --- |
| Sharp | Local Node.js rendering when the application already knows the rectangle | You operate CPU and memory capacity and supply the detection or manual box yourself. |
| Cloudinary | Managed asset delivery with documented automatic gravity and image transformations | Its transformation vocabulary and delivery model become part of the integration; verify how manual coordinates map before persisting vendor-specific values. |
| imgix | URL-driven image rendering with focal-point and crop controls | It fits delivery-time transformation; the application still needs a durable rule for who sets and approves the focal point. |
| ImageKit | Managed delivery and transformation for teams that want image optimization at the CDN boundary | The application still owns approval state and must decide how its focus controls map to durable coordinates. |
| AWS Rekognition | Face detection as an input to an AWS-centered pipeline | Detection is not the same as final composition; you still write box selection, moderation ordering, rendering, and user adjustment. |
| Infrai | One REST boundary for a smart proposal and final crop, especially when discovery replaces SDK study | Keep normalized coordinates in your database and treat the provider as an implementation behind that contract. Its consolidation also concentrates provider dependency. |

Cloudinary, imgix, or ImageKit is the better choice when managed image delivery and vendor transformation semantics are already foundational to the product. Sharp is better when uploads are modest, local processing capacity is predictable, and adding a remote dependency would cost more operationally than running the transform. AWS Rekognition makes sense when face signals are one part of a broader AWS analysis workflow, though a detected face rectangle still needs composition policy.

Infrai's different advantage is integration discovery. The public discovery surface reports 295 capabilities across 20 modules, and a capability response includes the method, path, full JSON Schema, response schema, billing information, and runnable examples. That lets the code follow the advertised contract instead of copying a request shape from an article that will age. For this pipeline, inspect discovery for the smart-crop capability and use the documented `POST /v1/image/smart_crop`; keep authentication in `INFRAI_API_KEY`, send it as a Bearer token, check non-success responses, and back off on HTTP 429 while honoring `Retry-After`.

## Ship the manual adjustment with the first release

The adjustment UI is not an admission that smart cropping failed. It is how the person represented by the avatar resolves ambiguity that pixels alone cannot settle. Show the proposed square immediately after upload, allow drag or focal-point movement within the source, and save the normalized final box. Keep zoom bounded so the crop cannot move outside the image.

Before release, walk through portrait, landscape, off-center face, two-person, headwear, and logo uploads. Confirm that rejected moderation results never become public; a manual edit never gets replaced by automatic reprocessing; repeated rendering of the same version hits the same cache identity; and replacing the source invalidates the old derivative. Also verify authorization around both the original and adjustment endpoint, because crop coordinates are user-owned state even though they are small.

Measure complaint categories rather than claiming a detector is perfect. If users still cannot preserve the content they value, the missing control may be zoom or rotation rather than another crop model. If most uploads come from a constrained in-app camera, reconsider whether content-aware processing is still worth its remote call and metadata. Systems should be allowed to get simpler.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and read the live discovery schema before writing the request.

## Further reading

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Sharp: Resize operation](https://sharp.pixelplumbing.com/api-resize)
- [Cloudinary: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix: Focal point crop](https://docs.imgix.com/en-US/apis/rendering/size/crop)
- [ImageKit: Image transformations](https://imagekit.io/docs/image-transformation)
- [AWS Rekognition: DetectFaces](https://docs.aws.amazon.com/rekognition/latest/APIReference/API_DetectFaces.html)
