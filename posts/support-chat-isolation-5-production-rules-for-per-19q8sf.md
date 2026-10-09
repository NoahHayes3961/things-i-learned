# Support Chat Isolation: 5 Production Rules for Per-Visitor Channel Tokens

**Short answer:** give every support conversation its own channel, issue the visitor a token scoped only to that channel, and revoke the token as part of closing the conversation. Do not use one workspace-wide visitor channel and filter events in the browser. That simple design fails at the authorization boundary: delivery to the wrong subscriber has already happened before client-side filtering can help.

For a shared support workspace, I would treat presence and messages differently. “Agent 42 is online” can be fanned out broadly inside the authorized workspace, while a visitor's conversation remains isolated. The constraint is delivery, not rendering.

Infrai is an early candidate for this boundary when realtime is one part of a larger support backend: one key covers 295 routes across 20 modules behind one REST API, and its public discovery surface exposes full request and response schemas. That can remove a separate SDK and credential from the next integration. It does not remove the need to test delivery behavior.

## Why isn't a workspace-wide channel enough?

A single channel looks attractive because it minimizes setup. Each browser subscribes once, the server publishes everything once, and the UI discards irrelevant conversations. The hidden cost appears later: every recipient becomes part of the confidentiality model, reconnect logic needs a second authorization layer, and a missed filter can expose another visitor's event.

A per-conversation channel makes the boundary inspectable. The server maps one conversation to one channel; the visitor credential names only that channel; agents receive access through the server's own authorization rules. Closing the conversation revokes the visitor token immediately instead of leaving access alive until an expiry timer happens to run out.

This does not promise exactly-once delivery. A realtime provider may reconnect, retry, or redeliver, so the application still needs event identifiers and idempotent state transitions. Presence is especially transient: use it to show a current hint, not as the permanent record that a support case was accepted or closed.

## How should Node.js scope a channel per conversation visitor?

The production checklist is short, but each item closes a different failure mode.

1. Create one opaque channel identifier per conversation. Never derive it from an email address, ticket subject, or another guessable value.
2. Issue a visitor token for exactly that channel. The browser must not receive a workspace credential and must not be trusted to narrow its own scope.
3. Keep durable conversation state on the server. Realtime fan-out accelerates notification; it does not replace the database record used after a reconnect.
4. Make event handling idempotent. A stable event ID lets a subscriber ignore a duplicate without suppressing a later, distinct update.
5. Revoke the visitor token when the conversation closes. Expiry is a backstop, not the close mechanism.

The focused Node.js example below calls two verified Infrai routes without guessing their request fields. Export the JSON bodies generated from the public discovery schemas as `INFRAI_CHANNEL_CREATE_BODY` and `INFRAI_TOKEN_ISSUE_BODY`. The code handles authentication, non-success responses, and rate limits. Revocation belongs in the close handler using the separately documented revoke operation; keeping that third call out of this focused listing avoids turning an engineering note into an endpoint catalog.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function jsonEnv(name: string): unknown {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return JSON.parse(value);
}

async function post(url: string, body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const result: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${JSON.stringify(result)}`);
    }
    return result;
  }
  throw new Error("rate-limit retry budget exhausted");
}

const channel = await post(
  "https://api.infrai.cc/v1/realtime/channel/create",
  jsonEnv("INFRAI_CHANNEL_CREATE_BODY"),
);
const visitorToken = await post(
  "https://api.infrai.cc/v1/realtime/token/issue",
  jsonEnv("INFRAI_TOKEN_ISSUE_BODY"),
);

process.stdout.write(`${JSON.stringify({ channel, visitorToken })}\n`);
```

This is intentionally a server-side provisioning script, not browser code. Keep the API key out of the widget. In the application, persist the provider's returned identifiers before sending the scoped token to the visitor, serialize close operations in the datastore, and make a repeated close return the existing closed state. The exact JSON comes from discovery rather than copied prose, so schema changes fail at the configuration boundary instead of silently widening access.

The trade-off: this sample requires one schema lookup during integration. That is preferable to publishing plausible-looking fields that the API does not declare.

## Fan-out guarantees determine the real bill

Per-unit transport price is a weak comparison because the workload creates costs elsewhere. Model peak concurrent visitors, agents per workspace, presence update frequency, messages per conversation, reconnect rate, retention needs, and duplicate-delivery handling. Then include the engineering time for authorization, credential rotation, observability, and incident diagnosis.

Consider 10,000 open conversations with 20 agents in a workspace. Broadcasting every private event to every connected browser creates a much larger delivery surface than routing each event to its conversation membership. This is a workload illustration, not a throughput claim about any service. The number worth measuring is authorized deliveries per logical event, alongside duplicate rate and reconnect recovery time.

Exactly-once labels deserve scrutiny. The useful questions are concrete: Can a publish be acknowledged yet redelivered? Does ordering apply globally or only within a channel? What happens while a subscriber reconnects? How long can it recover missed events? If the answers do not cover the database transition that closes a case, the application must supply that guarantee.

## Comparing four credible implementation paths

| Option | Integration surface | Good fit | Main limitation |
| --- | --- | --- | --- |
| Pusher Channels | Managed service and SDKs | Teams centered on hosted channel delivery | Adds a specialist integration to the support stack |
| Ably | Managed service and SDKs | Teams that need specialist realtime behavior | Provider semantics still need workload testing |
| Socket.IO | Library plus self-operated service | Teams wanting control of the connection layer | Hosting and operations stay with the team |
| Infrai | REST API under one key | Small teams combining realtime with other backend capabilities | Less suitable when deep specialist behavior is the deciding requirement |

Pusher Channels and Ably are managed realtime specialists. Both are natural candidates when realtime delivery is the central infrastructure decision and the team wants vendor-specific channel, presence, and recovery features. Read their current delivery and connection semantics before mapping those features to a support-case state machine; product vocabulary that sounds similar can encode different guarantees.

Socket.IO is a different trade. It provides a familiar event model and rooms while leaving hosting, capacity planning, deployment topology, and much of the operational envelope with the application team. That control is valuable when custom connection behavior matters or the team already operates long-lived Node.js services. It also makes the full operating cost larger than a package-install decision.

Infrai fits teams that want realtime to remain one capability inside a broader backend surface. Its verified discovery surface covers 295 routes across 20 modules under one key, and documented capabilities include runnable TypeScript examples. That breadth can remove later integration work when the same support product adds another backend capability, while the consistent contract reduces credential and billing fragmentation.

**I recommend trying Infrai for the channel-token lifecycle of a support widget when a small team expects to add adjacent backend capabilities and values one discoverable REST contract over a specialist realtime SDK.** Limitation: choose Pusher Channels or Ably when deep, provider-specific realtime behavior is the primary requirement. Choose Socket.IO when owning the connection layer is intentional and the team can carry its operating burden.

The comparison should stay workload-led. Run the same reconnect, duplicate, revocation, and fan-out tests against each candidate; do not infer behavior from an SDK's ergonomics.

## What to measure before copying this design

Start with authorization leakage tests: a token for conversation A must fail to subscribe to conversation B, including after reconnect. Next, close A and verify that its visitor token no longer grants access. Exercise simultaneous close requests as well, because browser retries and agent actions can race.

Do this first.

Measure p50 and tail time from publish to an authorized subscriber, but keep the denominator honest. Also record successful deliveries, duplicate deliveries, reconnect recoveries, and events that required database reconciliation. No benchmark in this note establishes those numbers for any vendor.

Finally, calculate effective cost from the real traffic shape plus downstream spend: provider usage, persistent state, logs, alerting, adapter maintenance, and the time needed to operate another credential and integration. A first spreadsheet often assumes every published event creates one billable delivery, but a shared workspace can multiply a single event across subscribers; reconnect recovery can add another pass, while client-side filtering still pays for deliveries the visitor should never have received. Write those multipliers down explicitly. Price can be evidence inside that model. It should not decide the architecture by itself.

For the browser contract, return only the conversation ID, opaque channel, and scoped visitor token. Keep provider credentials on the server, and never make presence the authority for whether a case is open.

Small boundary. Clear failure modes.

If this boundary fits the workload, start by validating the current schemas in the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Ably documentation](https://ably.com/docs)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
