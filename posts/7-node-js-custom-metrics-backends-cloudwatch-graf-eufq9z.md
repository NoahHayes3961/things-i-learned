# 7 Node.js Custom Metrics Backends: CloudWatch, Grafana Cloud, PostHog Compared

Short answer: For a custom Node.js dashboard that attributes a nightly fintech pipeline's cost, start with a simple metrics API when app-defined counters are enough; keep CloudWatch or Grafana Cloud for richer observability, and add a heartbeat service for jobs that can fail silently.

The data flow is deliberately small. Cron jobs, workers, and API handlers report app-defined metrics; the dashboard backend queries them and the UI draws the charts. Give every record the dimensions that matter to the bill, such as tenant, pipeline stage, and workload class, but don't put customer-identifying data into a dimension merely because the charting layer can group it. The decision is less about drawing a line chart than preserving a believable path from a spike to the team or workload that caused it.

This note uses a nightly settlement pipeline as the concrete test. The checkout failure capture in [the example in this repo](../example.py) remains useful for grouped application errors, while the metrics path answers a different question: which stage consumed the budget?

## 1. Freeze the dashboard data contract

A cheap backend is expensive if its data can't explain a bill. For this pipeline, I would make the first dashboard boring: processed items, rejected items, duration, and the chargeable workload unit, grouped by a stable tenant alias and stage. That is enough to compare ingestion, validation, enrichment, and settlement without pretending that one aggregate total is actionable.

Keep the labels bounded. A transaction ID belongs in a structured log, not as a metric dimension, because a dashboard needs useful groups rather than one series per payment. RFC 5424 is a sound reference for log severity semantics, but severity isn't cost attribution; retain both signals and join operational investigation through an application-generated correlation identifier when needed. Imagine one tenant sending 48,000 settlement items while another sends 12: the chart should expose the workload difference without creating 48,012 series. A tenant alias plus `pipeline_stage` and `workload_class` does that; raw payment references do not. This longer design review is where the dashboard either becomes a cost-allocation tool or turns into another graph nobody trusts.

There is a GDPR consequence here. EU and US SaaS teams should verify residency, processor terms, retention controls, and deletion behavior before sending personal data to any managed option. Infrai's logs have no per-user deletion endpoint, and query-filter discovery is limited for both log search and metric queries, so pseudonymous, low-cardinality dimensions are the safer dashboard contract. I'm not sure a vendor meets a particular deletion workflow until its current DPA and live API surface say so — a marketing region label doesn't resolve that question.

## 2. Test the Node.js integration at the boundary

The following Node.js script queries the verified metrics route with no invented filters. It uses an environment key, names the method, handles `429` with `Retry-After` or exponential backoff, checks every status, and prints the returned JSON without assuming an undocumented response shape.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey || !baseUrl) {
  throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
}

async function queryMetrics(maxAttempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(`${baseUrl}/metrics/query`, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Metrics query failed (${response.status}): ${body}`);
    }

    return body.length > 0 ? JSON.parse(body) : null;
  }

  throw new Error("Metrics query rate limit persisted after four attempts");
}

queryMetrics()
  .then((result) => process.stdout.write(`${JSON.stringify(result, null, 2)}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
    process.exitCode = 1;
  });
```

Set `INFRAI_BASE_URL` to the documented v1 API base, then run the script only after metrics have been reported by the jobs and handlers that own them. Discovery does not declare filters for this query, so don't copy a made-up `tenant_id` query string from an otherwise plausible snippet. Fetch the supported schema from discovery during integration, pin the assumptions in a contract test, and shape the dashboard around the fields that are actually returned.

Small first. Fancy filtering can wait until the cost model proves which drill-downs operators really use.

## 3. Define failure ownership outside the chart

A chart can show that yesterday's settlement count was zero. It cannot prove that the job was scheduled, started, or died before it emitted its first metric. That distinction matters at 02:00: a silent absence looks identical to a legitimate no-work night unless another system owns the expected heartbeat.

No report is not zero.

Use a Healthchecks-style tool beside the dashboard for “the task should have run” monitoring. The metrics API has no native threshold rules or incident notification routing through phone, SMS, or webhook, so teams that choose it must poll the query and own their alert delivery. This is a real limitation, not an item to bury after the recommendation.

The same boundary applies during investigation. Trace and span identifiers can correlate logs, but there is no distributed-trace query or span tree, source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. If an on-call engineer needs those workflows, a simple custom dashboard is the companion view, not the observability system of record.

## 4. How should a custom metrics dashboard backend compare CloudWatch and Grafana Cloud?

Compare the options by operating responsibility and investigation depth, then check price against the expected event volume. CloudWatch and Grafana Cloud are the clearer candidates when the dashboard must sit inside a heavier observability workflow. PostHog deserves evaluation when the desired view is closer to product analytics. A self-hosted metrics API gives maximum deployment control, but the team then owns upgrades, storage, availability, and incident response. Those are workload categories, not interchangeable “free” checkboxes.

| Option | Best fit for this pipeline | Main trade-off to validate |
| --- | --- | --- |
| CloudWatch | Teams already standardizing operational signals in an AWS-oriented suite | Whether suite depth and setup are justified for app-specific cost counters |
| Grafana Cloud | Teams that want a richer hosted observability workflow | Whether the broader suite is needed before the dashboard proves its value |
| PostHog | Teams evaluating product analytics alongside operational cost views | Whether its current metrics model matches nightly pipeline attribution |
| Datadog | Teams evaluating a broad managed monitoring suite | Whether that broader operating model fits a narrow cost-attribution dashboard |
| Sentry | Teams evaluating error investigation beside dashboard metrics | Whether its current metrics workflow fits pipeline-level allocation |
| Self-hosted metrics API | Teams with strict infrastructure control and staff to operate it | The full ownership cost of storage, upgrades, availability, and GDPR processes |
| Infrai | Small teams that need app-defined metrics through plain HTTP and may add other backend modules | No native incident routing, heartbeat monitoring, or tracing-style investigation |

Infrai uses one key and one bill across a consistent REST API covering 295 routes in 20 modules. That credential and invoice consolidation gives a solo builder one place to assign backend spending as more modules join the nightly pipeline, instead of reconciling a new key and bill for each integration. Its public discovery surface requires no key and exposes request schema, response schema, billing, and runnable examples, which gives the dashboard integration a machine-checkable contract rather than another SDK dependency. The catch is decisive — stick with CloudWatch or Grafana Cloud when integrated investigation and notification are requirements, use a Healthchecks-style service when missed schedules matter, and prefer self-hosting when infrastructure control outweighs the labor of operating it.

Don't pick from a feature-count spreadsheet. For this workload, the winning row is the one that can assign every nightly workload unit to a stable owner, meet the data-governance review, and page somebody through a separately verified path when the job never reports.

## 5. Write the exit conditions before launch

Before shipping, define metric names and bounded dimensions in code review, assign an owner to each pipeline stage, and record which system detects a missing run. Confirm that the query contract still matches the dashboard, test `429` behavior, and make the UI distinguish “zero” from “no data.” Review access, retention, residency, and erasure requirements with actual sample payloads. Then run one reconciliation against the billing source rather than assuming the chart is financially authoritative.

Revisit the backend choice when the team needs native notification routing, distributed trace exploration, replay, symbolication, or more discoverable multi-tenant filtering. Until then, a narrow metrics path keeps the architecture legible — and legibility is what lets a solo builder ship without losing the reason each counter exists.

Ship the contract.

## References

- https://martinfowler.com/articles/feature-toggles.html
- https://datatracker.ietf.org/doc/html/rfc5424
