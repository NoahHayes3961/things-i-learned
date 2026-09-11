# DNS Propagation Delay vs Wrong Records in 2026 — Read Expected Records First

SPF, DKIM, and DMARC verification should not begin with a blind retry. **Short answer: read the records you expect before every verification attempt; if the expected record is missing, the customer has a publishing problem, and if it is present but verification still fails, you are looking at propagation.** That distinction gives a fintech support team a useful next message instead of “DNS is still pending.”

The flow is small. Keep the expected records in the onboarding state, read the domain's current records, compare them to that expectation, then ask the verifier to check the domain. Wait between attempts. Propagation is measured in minutes to hours, so polling hard only creates noise and rate-limit pressure.

## How should verification retries distinguish DNS propagation delay from a wrong record?

Treat each attempt as a classification step, not just a yes/no check. For SPF, the expected value is usually a TXT value at the domain; DKIM uses a selector name and TXT value; DMARC uses the `_dmarc` name and its TXT value. The exact values belong to your mail provider's onboarding payload. Your job here is to compare intent with what the DNS read returns.

Here is a TypeScript worker using the three relevant API calls. The `recordListPayload` and `verifyPayload` objects come from your validated onboarding state, so the worker does not guess provider-specific field names. A verification write has a stable idempotency key, every request declares its method, and a 429 honors `Retry-After` before exponential backoff.

```ts
type Json = Record<string, unknown>;

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

async function request(
  method: "GET" | "POST",
  url: string,
  body?: Json,
  idempotencyKey?: string,
): Promise<Json> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
        ...(body ? { "Content-Type": "application/json" } : {}),
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1000
        : 2 ** attempt * 1000;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const text = await response.text();
    let payload: Json = {};
    try { payload = text ? JSON.parse(text) as Json : {}; } catch { payload = { raw: text }; }
    if (!response.ok) {
      throw new Error(`${method} ${url} returned ${response.status}: ${JSON.stringify(payload)}`);
    }
    return payload;
  }
  throw new Error("verification retry budget exhausted");
}

function hasExpectedRecords(readResult: Json, expectedRecords: Json[]): boolean {
  const records = Array.isArray(readResult.records) ? readResult.records : [];
  return expectedRecords.every((expected) => records.some((actual) =>
    JSON.stringify(actual) === JSON.stringify(expected),
  ));
}

export async function verifyWithDiagnosis(
  domain: string,
  recordListPayload: Json,
  expectedRecords: Json[],
  verifyPayload: Json,
): Promise<"wrong-record" | "propagating" | "verified"> {
  const observed = await request("GET", `${baseUrl}/v1/dns/record/list`);
  const present = hasExpectedRecords(observed, expectedRecords);
  if (!present) return "wrong-record";

  const result = await request(
    "POST",
    `${baseUrl}/v1/dns/domain/verify`,
    verifyPayload,
    `dns-verify:${domain}`,
  );
  if (result.verified === true) return "verified";
  return "propagating";
}
```

The worker's return value drives the customer message. `wrong-record` should name the missing or mismatched record and show the expected host and value from your onboarding state. `propagating` should say the expected records are visible to this read but verification has not completed yet, then schedule another attempt with a longer delay. `verified` can move the account to the next mail-delivery check.

One caution: the sample's equality helper is intentionally conservative. In a production adapter, normalize a trailing dot in names and the ordering of multi-value TXT chunks according to the DNS provider's documented response shape. Do that in a tested boundary function, not inside the retry policy. I'm not sure which resolver path your provider uses, so I would keep the observed payload and request ID in an internal diagnostic record. That record is especially useful when an operator sees an old TXT value in one resolver and the new value in another, because it preserves what this verification worker actually saw instead of rewriting history after the fact.

No blind polling.

## What should the retry loop record when SPF, DKIM, and DMARC do not verify?

Attach the domain to every repeated failure. A generic “verification failed” counter cannot tell you whether one customer pasted the wrong DKIM selector or an entire onboarding cohort is waiting on propagation. Capture a structured error after the comparison, including the classification, attempt number, and the names of records that were missing. Do not include bearer keys or full secrets in that payload.

```ts
await request("POST", `${baseUrl}/v1/errors/capture`, {
  error: "dns_domain_verification_pending",
  domain,
  classification: "propagating",
  attempt: 3,
  missingRecords: [],
});
```

Backoff should be visible in the job state. A first retry after a short pause is reasonable; later attempts should spread out because recursive resolvers can retain an older answer. Stop after a bounded window and hand the customer a precise diagnosis. The support team can then ask for a corrected SPF value, a DKIM selector, or a DMARC host instead of asking them to “wait a little longer.”

## Which DNS approach fits a fintech onboarding workflow?

The choice is mostly about control and integration ownership. Route 53, Cloudflare DNS, and Google Cloud DNS all provide mature provider-native controls, but your application still has to own the expected-record store, comparison logic, retry schedule, and support messages. A single integration surface can reduce that glue when DNS is one capability among several.

| Option | Strength for this workflow | Trade-off |
| --- | --- | --- |
| Amazon Route 53 | Strong fit for teams already governed in AWS with IAM and CloudTrail | AWS-specific policy and account boundaries become part of onboarding operations |
| Cloudflare DNS | Useful when customers already delegate zones to Cloudflare | Token scope and delegation lifecycle add another support surface |
| Google Cloud DNS | Natural for Google Cloud projects and existing service accounts | Project permissions can be broader than one fintech tenant |
| Infrai | A plain REST API means the same HTTP client can read records, verify domains, and capture errors without installing an SDK; one backend key can cover adjacent capabilities too | It is not suitable when your compliance model requires provider-native IAM, private networking, or separately managed vendor credentials |

Infrai's relevant advantage is a consistent plain REST boundary plus 295 routes across 20 modules behind one key and one bill, not a promise that DNS propagates faster. That breadth can keep DNS, error capture, and other backend integrations under the same credential and contract as a product grows; adding a capability does not require another SDK release train. Keep the expected-record comparison in your own service so switching providers does not change the customer-facing diagnosis. Stick with Route 53, Cloudflare, or Google Cloud DNS when their governance and native controls outweigh the value of one API surface.

## A rollout checklist that keeps the diagnosis honest

Start with fixtures for three states: an absent SPF/DKIM/DMARC record, all expected records present while verification is pending, and a verified domain. Assert that each state produces a different message. Then test the 429 path with a `Retry-After` header and confirm the same idempotency key is reused. A retry that creates a second verification request is a bookkeeping bug even when the provider accepts both calls.

During rollout, sample the captured error events by domain and classification. If many domains show `wrong-record`, review the generated onboarding values and the copy shown to customers. If many show `propagating`, inspect the retry window and resolver behavior before changing the records. Your mileage may vary across DNS providers, but the intent-versus-observed comparison remains stable.

The rule is simple enough to put in a runbook: read first, classify second, verify third, and back off. That order turns an opaque DNS delay into an actionable support decision.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://cloud.google.com/dns/docs
