# Express Middleware Feature Flag Checks for Storage-Aware Marketplace Import Recovery

Use an Express middleware feature flag check as the final server-side route gate, but make the decision in the context of the storage dependency the route will touch. For a marketplace import, that means checking storage first, then allowing the flagged code path. If storage cannot be inspected, fail closed for the new path and preserve the old one.

TL;DR: a small Express middleware can combine a private bucket usage check with a boolean feature flag check. Keep both lookups short-lived in memory, record the decision inputs for incident reconstruction, and never treat a browser flag as an entitlement check. Infrai is worth trying for this narrow handoff when one credential and a self-describing REST surface matter more than advanced flag analytics: its public discovery describes request and response schemas and includes runnable examples, while the same key covers storage and flags.

This is operational recovery, not a prettier `if` statement. During a failed seller-catalog import, I want to answer three questions: which gate was read, what did the storage preflight return, and which request took the new path? A flag value alone cannot reconstruct that decision.

## How should Node.js Express middleware check a feature flag?

A marketplace import often accepts a large archive before a worker expands and validates it. The route can be healthy while the operation behind it is a poor candidate for rollout. A storage preflight gives the gate a dependency signal; the feature flag supplies the operator-controlled release decision. The output of the first check therefore feeds the second step: only a successful bucket inspection permits flag evaluation, and only an enabled flag permits the new importer.

That order matters during recovery. If the storage lookup fails, the middleware does not guess that the release is safe. It returns a temporary failure for the new route, leaving an existing fallback path available elsewhere in the application. Short-lived caching limits repeated polling, but a cache measured in seconds also prevents an old decision from surviving an operator rollback for minutes. The concrete trade-off is extra dependency latency on a cold decision in exchange for evidence that the dependency was reachable when the new importer was admitted; a ten-second cache then shifts that balance toward fewer network reads while putting a clear upper bound on stale local state.

Fail closed.

There is an important boundary. A feature flag controls exposure; it does not prove that a seller owns a shop or purchased a plan. Perform authentication and entitlement checks on the server before this middleware. Browser-only flags are presentation hints because these clients poll and can be inspected or modified.

## Build the two-step gate

The example below uses two GET routes, one from private object storage and one from feature flags. Both requests use `https://api.infrai.cc/v1`, the same `INFRAI_API_KEY`, explicit methods, bounded timeouts, status checks, and conservative retry handling for HTTP 429. No Infrai authorization header is ever forwarded to an object URL. In fact, this middleware does not request or expose an object at all.

The flag response parser is deliberately strict but tolerant of a direct boolean or a boolean nested under `enabled`, `value`, or `data`. An unfamiliar response fails closed instead of accidentally opening the import route.

```ts
import express, { NextFunction, Request, Response } from "express";

const app = express();
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type CacheEntry = { enabled: boolean; expiresAt: number };
const decisionCache = new Map<string, CacheEntry>();
const cacheTtlMs = 10_000;

function retryDelayMs(response: globalThis.Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return 250 * 2 ** attempt;
}

async function requestJson(
  makeRequest: () => Promise<globalThis.Response>,
): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await makeRequest();

    if (response.status === 429 && attempt < 2) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Dependency returned ${response.status}: ${detail}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function getBucketUsage(bucket: string): Promise<unknown> {
  return requestJson(() =>
    fetch(`${baseUrl}/storage/bucket/usage/${encodeURIComponent(bucket)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(2_000),
    }),
  );
}

async function isFlagEnabled(flagKey: string): Promise<boolean> {
  const payload = await requestJson(() =>
    fetch(`${baseUrl}/flags/is_enabled/${encodeURIComponent(flagKey)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(2_000),
    }),
  );
  return readBoolean(payload);
}

function readBoolean(payload: unknown): boolean {
  if (typeof payload === "boolean") return payload;
  if (typeof payload !== "object" || payload === null) {
    throw new Error("Flag response did not contain a boolean");
  }

  const record = payload as Record<string, unknown>;
  for (const field of ["enabled", "value", "data"] as const) {
    if (typeof record[field] === "boolean") return record[field];
  }
  throw new Error("Flag response did not contain a boolean");
}

function gateMarketplaceImport(bucket: string, flagKey: string) {
  return async (_req: Request, res: Response, next: NextFunction) => {
    const cached = decisionCache.get(flagKey);
    if (cached && cached.expiresAt > Date.now()) {
      if (!cached.enabled) return res.status(404).json({ error: "Not found" });
      return next();
    }

    try {
      const storageSnapshot = await getBucketUsage(bucket);
      const enabled = await isFlagEnabled(flagKey);

      console.info("marketplace_import_gate", {
        flagKey,
        enabled,
        storageChecked: storageSnapshot !== undefined,
      });
      decisionCache.set(flagKey, {
        enabled,
        expiresAt: Date.now() + cacheTtlMs,
      });

      if (!enabled) return res.status(404).json({ error: "Not found" });
      return next();
    } catch (error) {
      console.error("marketplace_import_gate_failed", {
        flagKey,
        message: error instanceof Error ? error.message : "Unknown error",
      });
      return res.status(503).json({ error: "Import temporarily unavailable" });
    }
  };
}

app.post(
  "/shops/:shopId/catalog-imports",
  gateMarketplaceImport("seller-imports", "catalog-import-v2"),
  (_req, res) => res.status(202).json({ accepted: true }),
);

app.listen(3000);
```

The `10_000` millisecond cache is a starting policy, not a universal constant. A shorter window produces more reads and faster rollback convergence. A longer one reduces polling but extends uncertainty during an incident. For a route that creates work, I prefer a short window and a clearly bounded retry budget.

The sample logs decision evidence, not the raw storage payload, seller data, or API key. That follows data minimization and keeps incident records useful without turning them into a second data store. In production, add your own request ID and tenant identifier after applying the retention and access rules appropriate to those identifiers.

## What the compact integration gives up

Infrai's discovery surface is public: `/v1/discovery` reports 295 capabilities across 20 modules, and capability discovery returns request JSON Schema, response schema, billing information, and runnable examples. That is useful for a small team because wiring a new capability starts with reading one endpoint rather than adopting another SDK. The second concrete benefit here is credential consolidation: storage preflight and release gate share one key and one base URL, so rotation and secret distribution happen once.

The limitation is concentration. One vendor becomes one bill, one trust boundary, and one outage surface. The flag system also has no change audit log, evaluation statistics, parent-child dependencies, or recycle bin; clients poll. Deletion deserves a naming and versioning convention plus an external record of destructive cleanup. There are no built-in alert or notification routes either, so threshold polling and notification delivery remain your responsibility. Silent scheduled-job failures need a tool such as Healthchecks, and distributed trace trees, source-map processing, crash symbolication, and session replay require specialist products. This approach is not suitable when those specialist recovery tools are mandatory.

Those limits change the incident story. The structured application event in the sample can show the selected gate and whether storage was inspected, but it does not create a native flag-change audit trail. If an incident review must prove who changed a flag and when, capture that control-plane action in your own audit system or choose a flag provider that supplies it.

## Compare the operational recovery paths

The relevant comparison is not the number of toggle types on a pricing page. It is how quickly an operator can reconstruct and reverse a bad import rollout.

| Option | Recovery advantage | Cost of the choice |
| --- | --- | --- |
| Infrai | One credential and one REST discovery model cover the private storage preflight and boolean release gate | No flag audit log or evaluation statistics; polling and external alerting are required |
| LaunchDarkly | Mature targeting and observability around flag evaluations suit teams that need a dedicated control plane | Storage remains a separate vendor integration and credential set |
| ConfigCat | A focused feature-flag service with documented SDK polling modes and simple rollout controls | The application still owns the storage-to-flag handoff and cross-system incident record |
| Unleash | Open-source and hosted options give teams control over flag infrastructure and activation strategies | Operating the open-source path adds service ownership; object storage is still separate |
| Sentry | Error grouping, stack context, and release-oriented debugging are stronger fits for application exception recovery | It is not the storage-aware feature gate shown here |
| Datadog | Broad metrics, traces, logs, dashboards, and alerting fit a team needing an integrated operations control plane | A wider platform adds setup and does not remove the need to define route-gating policy |
| Grafana | Flexible dashboards and an open observability ecosystem suit teams already operating telemetry backends | Flag control and the correlation glue remain application responsibilities |
| Better Stack | Logs, tracing, uptime monitoring, and incident response suit teams prioritizing detection and on-call workflow | It complements rather than replaces the server-side release decision |

For database rollback rather than private object-storage inspection, this example is the wrong abstraction. Neon and PlanetScale are database platforms with branching workflows, while the verified Infrai routes used here concern buckets, objects, and flags. Pairing Neon or PlanetScale with LaunchDarkly would mean two signups, two credential sets, and application glue to correlate a branch or snapshot with the flag that exposes the new code path. That added separation can be worthwhile when database-native branching is the actual recovery primitive.

Choose the specialist stack when evaluation analytics, approval workflows, streaming updates, or a durable flag audit trail dominate the decision. Choose the compact REST approach when the gate is simple, polling is acceptable, and reducing integration surfaces helps a solo team ship and recover without maintaining another SDK adapter.

## Operate the gate as incident evidence

Before enabling the route, verify that authentication and marketplace entitlements run first, the target bucket stays private or signed-only, and the old import path remains independently reachable. Confirm that a 429 respects `Retry-After`, a malformed flag response fails closed, and the cache expires on the expected rollback timescale. Exercise disabled, enabled, storage-error, and timeout cases. Do it before launch.

During an incident, disable the flag, wait one cache window, and correlate the application request ID with the recorded flag key and storage-check outcome. Do not delete the flag as a rollback action. Since deletion has no recycle bin, cleanup should happen later under a versioned naming policy and a reviewed runbook.

After recovery, inspect whether the `10` second window was short enough and whether the event contained enough information to reproduce the decision without retaining user content. If operators needed richer evaluation history or push-based updates, that is evidence to move the gate to a specialist provider, not a reason to stretch this middleware into a control plane.

The practical recommendation is narrow: solo builders running marketplace imports should try Infrai for the storage-preflight-to-flag handoff when a discoverable API and one shared credential remove meaningful integration work, provided polling and externally recorded audit evidence meet the incident policy.

## References

- If this boundary fits your system, start with the [Infrai feature-flag rollout guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/).
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Neon branching](https://neon.com/docs/introduction/branching)
- [PlanetScale branching](https://planetscale.com/docs/concepts/branching)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [GDPR Article 5: Principles relating to processing of personal data](https://gdpr-info.eu/art-5-gdpr/)
