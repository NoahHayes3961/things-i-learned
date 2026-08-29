# Game Training Artifacts: A Direct Browser Storage API Pattern for Auth and Retention

Short answer: for direct browser uploads of game training artifacts, make the API own tenant authorization and a versioned retention decision, while object storage owns the bytes; treat validation callbacks as repeatable evidence, not as permission to change ownership.

That boundary is more useful than choosing a storage API by upload speed alone. A replay bundle, checkpoint, or telemetry export can be large, but the dangerous part is usually smaller: one tenant's browser receives a key for another tenant, a retry creates two policy records, or a callback arrives after the retention rule has changed. The upload must survive those ordinary races and still be explainable months later.

I would test this design against three questions before shipping it: can the system prove who authorized the object, can it reconstruct the policy that applied, and can it safely repeat every callback? If the answer to any one is no, a direct upload is only moving the failure somewhere harder to see.

## Start with the retention decision, not the bucket

For a game studio, the artifact's identity should be an application record before it is a storage key. The record can contain a tenant ID, artifact ID, content type, byte limit, retention class, policy version, and an expected state such as `awaiting_upload`. The server generates the object key. The browser's filename stays display metadata.

This matters for reproducibility. A policy called `seasonal-training` is not enough if its meaning changes between two uploads. Store the policy version and the selected expiry timestamp with the artifact record. A later policy edit may affect new artifacts, but it should not silently rewrite the historical explanation for an old one.

The storage prefix is a useful secondary guard: `tenants/{tenantId}/artifacts/{artifactId}`. It is not the authorization model. Authorization belongs in the application database and in the short-lived grant issued after the authenticated request. A user-controlled prefix, even one that looks harmless, is not tenant isolation.

The key is generated once.

The smallest workable state machine is deliberately boring:

`awaiting_upload -> uploaded -> validated -> retained`

`rejected` and `expired` are terminal application states. A callback may provide evidence for a transition, but it should not be able to choose a tenant or retention class. That choice was made before the browser left the control plane.

## How should a Node and Express storage API handle browser auth, validation, and webhook callbacks?

Use three application-side operations with different responsibilities. The authorization operation authenticates the caller, checks tenant membership, validates the requested artifact type and size, creates the artifact row, and returns a short-lived upload grant. The browser sends bytes directly to object storage. The confirmation operation accepts only the artifact ID, checks that it belongs to the caller's tenant, and records that the browser has finished. A worker or storage notification then performs asynchronous validation and records its result.

The API should not accept a browser-supplied bucket, tenant prefix, expiry, or retention class as an authority. It may accept a requested class, but it must resolve that request against server-side policy. It also should not mark an artifact usable merely because the browser said "done". Verify the stored object's metadata, size, and content constraints through the storage interface available to your deployment.

Here is the application-side contract. It leaves the provider-specific upload grant out on purpose: the portable part is the ownership and state transition, while the grant payload differs between storage systems.

```ts
import { randomUUID } from "node:crypto";

type RetentionClass = "match-review" | "seasonal-training";
type ArtifactState = "awaiting_upload" | "uploaded" | "validated" | "rejected" | "expired";

type Artifact = {
  id: string;
  tenantId: string;
  objectKey: string;
  retentionClass: RetentionClass;
  policyVersion: number;
  expiresAt: string;
  state: ArtifactState;
};

type UploadRequest = {
  tenantId: string;
  retentionClass: RetentionClass;
  expiresAt: string;
};

export function createArtifact(input: UploadRequest, policyVersion: number): Artifact {
  const id = randomUUID();

  return {
    id,
    tenantId: input.tenantId,
    objectKey: `tenants/${input.tenantId}/artifacts/${id}`,
    retentionClass: input.retentionClass,
    policyVersion,
    expiresAt: input.expiresAt,
    state: "awaiting_upload",
  };
}

export function acceptValidation(
  artifact: Artifact,
  observedTenantId: string,
  observedObjectKey: string,
  valid: boolean,
): Artifact {
  if (artifact.tenantId !== observedTenantId || artifact.objectKey !== observedObjectKey) {
    throw new Error("artifact ownership mismatch");
  }

  if (artifact.state === "validated" || artifact.state === "rejected") {
    return artifact;
  }

  return { ...artifact, state: valid ? "validated" : "rejected" };
}
```

That last function is intentionally idempotent. A webhook can be delivered twice, a worker can retry after a timeout, and a deployment can replay a queue. The same artifact must not acquire a new expiry or cross a tenant boundary on the second delivery. If validation needs more detail, store an append-only event with the provider event ID and make the state projection repeatable.

Use an explicit conflict response for a confirmation that names an artifact outside the caller's tenant. I use `404` when revealing that the artifact exists would itself leak information, and `409` when the caller is authorized to see the record but its state does not permit the requested transition. The choice is a security policy, not a storage feature. I'm not sure which response is right for every game platform; threat-model the distinction with the team that owns account privacy.

## What does a reproducible object-storage retention workflow measure?

The useful test is a failure matrix, not a happy-path upload. Run it with two tenants and the same human filename, then repeat each operation after a dropped response. For example, let tenant A start a replay-bundle upload, let the browser retry its confirmation twice, and then send a delayed validation event whose object key uses tenant B's prefix. The expected result is that the artifact ID, object key, policy version, and expiry remain stable, while the mismatched event is rejected without changing either tenant's record or exposing the existence of the other object; a retry must produce the same state transition as the first accepted event.

Measure authorization latency separately from byte-transfer time. Also capture the delay between storage completion and validation, callback age, duplicate-event count, rejected-object count, and time spent deleting or expiring objects. A percentile such as p95 is more actionable than an average when a training export blocks a release job. Keep request IDs and artifact IDs in logs, but avoid logging bearer grants or raw object contents.

The browser is an untrusted coordinator. It can disappear after uploading, refresh after receiving a grant, or submit the same completion twice. Those are normal states. Design the database transition so a lost response can be retried with the same artifact ID, and give the worker a durable inbox or equivalent deduplication record before doing expensive validation.

Do not infer retention success from a database timestamp alone. A scheduled deletion job needs a reconciliation pass that finds records whose expiry has passed, checks the storage object, and records the result. Conversely, an object discovered without a matching application record should enter quarantine or an investigation queue; it should not be adopted into a tenant merely because its prefix looks familiar.

## The trade-offs behind direct browser uploads

Streaming every file through Express gives one obvious control point, and it can be the right answer for small artifacts or an environment where storage grants cannot be configured safely. The cost is that the application process now carries upload duration, connection pressure, and failure recovery. Direct upload reduces that data-path burden, but it adds a grant lifecycle, browser-origin policy, confirmation state, and cleanup work.

The pattern is not suitable when the application must inspect every byte before storage, when clients cannot reach the object-storage endpoint, or when tenant-specific retention rules cannot be represented and audited outside the provider. In those cases, keep the stream behind a service you control or choose storage whose governance primitives match the requirement. Stick with a proxy when a compliance boundary requires the API to be the only network path for the content.

There is another limit that is easy to miss: an expiry timestamp in your database does not guarantee deletion at that exact instant. Provider lifecycle scheduling, queue delay, clock discipline, legal holds, and failed cleanup all need an operational policy. If the game requires immutable evidence, retention enforcement and legal hold belong in a design that has explicit support for those controls; a naming convention cannot substitute for them.

## A pre-ship checklist for the storage boundary

Before copying this pattern, verify the following against the actual storage product and deployment:

- The grant is scoped to one generated object key, one tenant context, an allowed method, and a short lifetime.
- The API validates content type and size twice: once before issuing the grant and again after the object arrives.
- The application stores the policy version and computed expiry, rather than recalculating historical records from current configuration.
- Confirmation and webhook processing are idempotent, authenticated where applicable, and safe to replay.
- CORS, encryption, audit retention, lifecycle behavior, and deletion guarantees are tested in the target environment.
- Metrics distinguish authorization, transfer, validation, notification, and cleanup failures.

The decision rule is simple: direct browser uploads are a good fit when the control plane can prove tenant ownership and the storage plane can provide the evidence needed for validation. If either side is vague, add a service boundary or select a different storage design. The bytes are the easy part.

## Further reading

- RFC 9110: HTTP Semantics — https://www.rfc-editor.org/rfc/rfc9110
- Federal Risk and Authorization Management Program — https://www.fedramp.gov/
