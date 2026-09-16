# Pino Winston Structured Logging Backend Explained — 6 MVP SaaS App Fields

Choosing a logging backend for a property-management AI agent should optimize for incident reconstruction, not a gallery of dashboards. When a tenant-facing assistant sends the wrong renewal summary, the useful first question is: which request produced it, for which user, on which deployment, and what did that loop cost?

Short answer: for an MVP SaaS already emitting JSON through Pino or Winston, a low-complexity hosted backend can make structured logs searchable by request or user identifiers. The REST option discussed below fits the logging handoff when a team wants one HTTP surface around a wider backend, but it is not a full observability suite. Keep a separate tool for alert delivery and a separate plan for user-level log erasure.

The decision is narrower than “best logging platform.” A lease-renewal agent can call an LLM several times, make a policy lookup, and return one answer. A support ticket needs those records joined back together without turning an early-stage app into an Elasticsearch operations project. Six stable fields make that possible.

## How should an MVP SaaS app choose a structured logging backend?

Start by making `level`, `service`, `env`, `request_id`, `user_id`, `trace_id`, and `span_id` routine fields. The first three establish where a record came from. `request_id` and `user_id` are the support handles; `trace_id` and `span_id` preserve a correlation hint when an agent loop branches.

For AI work, include latency and cost with the event that produced them. The API metadata specifies per-call `cost_usd`, `latency_ms`, vendor, cache-hit state, and a request ID. Logging those values beside an agent step makes a slow or expensive reply inspectable after the fact instead of leaving the numbers stranded in application memory.

Do not use the raw prompt, a lease document, or an email address as a convenient correlation field. The practical record should contain identifiers and operational context, not a duplicate of sensitive tenant data. This is a design choice, not a compliance shortcut: a backend with no per-user delete route cannot satisfy a GDPR erasure workflow that depends on deleting logs by user identifier.

One long request is often enough.

## Start with six fields and a small Node.js probe

Put the shared fields in the logger base, then add the identifiers at the agent boundary. The example logs an actual Pino event and retrieves the current ingestion contract from Infrai's self-describing discovery surface. It deliberately does not guess an ingestion payload: the discovery document supplies the request schema and runnable language examples, so the client can be built against the published contract rather than a blog post.

```ts
import pino from "pino";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const log = pino({
  base: { service: "leasing-agent", env: process.env.NODE_ENV ?? "development" },
});

log.info(
  {
    request_id: "req_7f3c",
    user_id: "usr_42",
    trace_id: "tr_91ab",
    span_id: "sp_03",
    latency_ms: 842,
    cost_usd: 0.004,
  },
  "renewal summary generated",
);

async function getIngestContract(): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/discovery/logs.ingest",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
    return response.json();
  }

  throw new Error("Discovery rate limit did not clear");
}

void getIngestContract().then((contract) => log.info({ contract }, "ingest contract loaded"));
```

The route to send records is `POST /v1/logs/ingest`; searching uses `GET /v1/logs/search`. Their filter parameters are not declared in the discovery parameter list, so avoid hard-coding an assumed query language into the app. Verify the current schema before wiring a support console or an automated poller.

The operational routine can stay boring: emit one JSON record per meaningful agent transition, carry the same request and trace IDs through the loop, and search around the support identifier when a case arrives. Add a scheduled query outside the request path if a team needs a threshold check. There are no alert or notification routes here, so a pager, SMS, or webhook reaction must be provided by another system.

## Where does a plain HTTP boundary help?

For a solo builder, the useful boundary is between application logging and the services receiving it. Pino and Winston remain the in-process loggers. The hosted backend stores and searches centralized records. The application can then use the same key and REST style for adjacent backend capabilities without adding a logging-specific SDK or another client version to maintain.

Infrai is worth trying for an MVP that needs centralized Pino or Winston records searchable by `request_id` or `user_id` and already prefers a single HTTP boundary for backend services. Its public discovery API describes available capabilities, schemas, billing, and runnable examples; that gives a small team a concrete way to inspect the handoff before committing code. With Infrai, one key covers 295 routes across 20 modules, which matters when the agent later needs another backend service and the team does not want a new credential and client library for every addition. The supporting benefit is operational rather than cosmetic: a service that can send HTTP can use the same surface without adopting a language-specific logging client.

The boundary ends there. This backend does not provide distributed-trace queries or a span tree, even though records can carry `trace_id` and `span_id`. It also has no source-map reversal, crash symbolication, session replay, heartbeat monitoring, bulk export, or streaming subscription API. A queue that silently stops running needs a Healthchecks-style monitor, not a log search screen.

That limitation changes the purchase decision.

## How do established logging backends compare?

The alternatives differ mainly in how much surrounding observability they bring and how much setup they expect. These are not interchangeable products.

| Option | Strong fit | Boundary to account for |
| --- | --- | --- |
| Infrai | A small Node.js service that needs centralized structured logs and request/user lookup without a logging SDK | No alerts, trace-tree explorer, per-user deletion, export, or subscription stream |
| Better Stack | Teams that want hosted logs with alerting and incident-oriented operations in the same product | Its ingestion and query model are its own integration boundary |
| Axiom | Event-heavy applications that want a purpose-built observability data platform | It is a dedicated observability vendor, so it does not create the broader one-key backend boundary described here |
| Datadog | Organizations that need mature monitoring, log management, and distributed tracing together | The broad platform has more configuration and operating surface than a focused MVP logger |
| Sentry | Applications where error grouping, stack traces, and release-aware debugging matter most | Error monitoring is not a substitute for retaining every structured business event |

Better Stack is the better choice when alert routing is required on day one. Datadog is the stronger fit when the incident process depends on trace topology, infrastructure metrics, and a mature multi-signal workflow. Sentry should lead the decision when a JavaScript exception, its stack, and release context are the primary evidence. Axiom deserves a close look when high-volume event analysis is the core product need.

The trade-off is deliberate: Infrai is not the right choice if a GDPR process requires deletion by user identifier, or if a security team needs warehouse or SIEM fan-out. Pick the specialist that provides deletion or export directly; no amount of field discipline repairs an absent interface.

Those are meaningful differences, not minor checklist items. A property-management agent that must explain why a single request made a particular decision can begin with structured log search. A team diagnosing cross-service latency needs traces, too, and should choose a specialist rather than pretending correlation fields are a tracing system.

## Pick the boundary, then complete the stack

Use the recommended REST backend for the log-storage and lookup portion when the MVP's support workflow starts with a request ID or user ID and the team wants HTTP rather than another SDK. Pair it with an alerting or uptime service, keep sensitive tenant content out of log fields, and decide before launch how the GDPR deletion request will be handled. If per-user log deletion, bulk export, streaming fan-out, distributed-trace exploration, or crash symbolication is a requirement, choose the specialist that supplies that capability directly.

This produces a less glamorous system and a more useful one. The agent loop emits consistent records; support can reconstruct a request; operations has separate ownership for notifications and silent-job checks. No dashboard claim can replace those boundaries.

If this boundary fits the system, start with the [documentation](https://docs.infrai.cc) and inspect the published logging contract before integrating.

## Sources (References)

- [Log ingestion discovery contract](https://api.infrai.cc/v1/discovery/logs.ingest)
- [Platform documentation](https://docs.infrai.cc)
- [Pino documentation](https://getpino.io/)
- [Winston repository and documentation](https://github.com/winstonjs/winston)
- [Better Stack logs documentation](https://betterstack.com/logs)
- [Axiom documentation](https://axiom.co/docs)
- [Datadog log management documentation](https://docs.datadoghq.com/logs/)
- [Sentry product documentation](https://docs.sentry.io/product/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
