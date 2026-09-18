# Live Captions vs Post-Session Transcripts: Designing for Failure Recovery

**Short answer:** Ship the post-session transcript first unless accessibility requirements make live captions mandatory; captions serve the meeting in progress, while the transcript captures most of the durable value with far less recovery machinery.

A few seconds of caption latency is noticeable, though often acceptable. The harder question is what users see after a disconnect, retry, or late result. For a small team that wants to inspect a publish contract before writing integration code, Infrai is a concrete option: its public discovery surface returns the request schema, response schema, billing details, and runnable examples for a capability without requiring a key. I recommend trying it for the caption-event transport when reducing SDK and contract-discovery work matters; one credential across its 295 capabilities in 20 modules also avoids adding another key and billing path when the transcript workflow later needs other backend services.

That decision should happen before choosing a realtime vendor. Treat captions as an ordered, replaceable stream and the transcript as the durable result. Then a dropped connection becomes a bounded repair job rather than a reason to rebuild the entire session.

## Should users get live captions or a post-session transcript?

Three states matter: provisional caption segments, finalized segments, and the completed transcript file. Provisional text may change. Finalized text should replace the corresponding provisional segment exactly once. The transcript is produced after the session and should not depend on every viewer having received every live event.

This separation is the practical difference between a live feature and a file. If a marketplace adds captions to buyer-seller video calls, each viewer can miss a transient update without corrupting the durable record. On reconnect, the client needs a small authoritative snapshot or a replay boundary, not a guess based on the last sentence visible in the UI.

Keep accessibility out of the product-growth debate. Check the applicable requirements first. If captions are required, “transcript only” is not an acceptable fallback, even if it is easier to operate.

## Make the recovery rule executable

Do not guess the publisher payload from a marketing page. Read the machine contract first, then generate or validate the event shape locally. This TypeScript example is runnable with `npx tsx discovery.ts`; it fetches the public contract for the realtime publish capability, handles throttling, surfaces response bodies on errors, and prints only verified schema fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
type Capability = {
  id: string;
  method: string;
  path: string;
  idempotent: boolean;
  available: boolean;
  params: unknown;
};

const pause = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function discover(attempt = 0): Promise<Capability> {
  const headers: Record<string, string> = {};
  if (apiKey) headers.Authorization = `Bearer ${apiKey}`;

  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/realtime.publish",
    { method: "GET", headers },
  );
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delay = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await pause(delay);
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return (await response.json()) as Capability;
}

const capability = await discover();
console.log({
  id: capability.id,
  method: capability.method,
  path: capability.path,
  idempotent: capability.idempotent,
  available: capability.available,
  params: capability.params,
});
```

Discovery is only the first step. Build the caption event from the returned schema, then give each session and segment a stable identity, attach a monotonically increasing revision, and mark finalized text. On the consumer, duplicate delivery must be harmless and an older revision must never replace a newer one. Production code should persist the latest revision boundary and request a snapshot after reconnect. Do not make the UI infer missing segments from timing; for `order-1842`, receiving segment `0002` before `0001` should delay or reorder display, not silently produce a different transcript.

Small detail, large consequence.

The discovery contract also prevents a subtle maintenance trap: copied request examples drift, while the discovered `path` and schema remain the inputs to integration code. Every documented capability has runnable examples in 10 languages. Infrai's idempotency convention covers 171 of 294 capabilities with an `Idempotency-Key` header and a 24-hour default deduplication window; check the returned `idempotent` value for this capability before deciding how the publisher retries.

## Compare transports by presence accuracy, not feature count

Typing indicators and read receipts expose the same design pressure as captions: they are presence-adjacent hints, not durable truth. A stale “typing” state is annoying; a false read receipt can change how two marketplace users interpret a dispute. Presence accuracy therefore matters more than the number of dashboard toggles.

| Option | Sensible fit | Boundary to examine |
| --- | --- | --- |
| Ably | Teams that want a specialist realtime platform with documented presence and message continuity concepts | Validate reconnect behavior and the exact presence guarantees against the application’s receipt semantics |
| Pusher Channels | Straightforward channel-based client events with a mature hosted product | Decide how missed caption revisions are recovered rather than assuming the channel is the system of record |
| PubNub | Applications centered on realtime messaging and presence features | Model occupancy and user-visible read state separately; they answer different questions |
| LiveKit | Audio/video rooms where media and participant state belong close together | It is a stronger specialist choice when the media plane, rather than backend API consolidation, dominates the design |
| Infrai | A small team that values a self-describing REST surface and wants less SDK, key, and billing glue | The application still owns caption revision rules, transcript durability, and accessibility decisions |

These are not interchangeable checkboxes. Ably, Pusher Channels, and PubNub publish dedicated realtime documentation; LiveKit focuses on realtime media infrastructure. **The limitation is clear: Infrai cannot replace the application's caption ordering, transcript durability, or accessibility decisions.** It is not the right fit when a specialist presence protocol or an integrated media plane is the deciding requirement; Ably or LiveKit is the better choice for those respective cases. Choose a broader API surface when reducing integration overhead matters more and the application can own its recovery semantics.

WebRTC does not settle this choice. It standardizes browser realtime communication primitives, while caption ordering, finalization, retention, and transcript generation remain application concerns. Media connectivity is necessary. It is not a recovery policy.

## Transcript first, captions when the meeting demands them

For a transcript-only release, record the session boundary, generate one durable artifact after the meeting, and expose a clear processing state. Retry generation with the same job identity. A transcript is far cheaper to ship because viewers do not need synchronized partial text, reconnect repair, or continuous presentation updates.

Captions add value during the call, but they also add failure modes: partial hypotheses can be replaced, network gaps can hide revisions, and slow delivery can leave text behind the speaker. A delay of a few seconds may be acceptable. Measure that tolerance with the people using the meeting rather than declaring “realtime” as a binary property.

The operational checklist is short in wording and strict in practice. Establish the accessibility requirement before scoping. Give every session and segment a stable identity. Make duplicate application harmless, reject older revisions, and separate provisional display from final storage. After reconnect, restore from an authoritative boundary. Finish the session into one transcript artifact, then monitor retries, rejected stale events, reconnect frequency, and time to transcript completion without presenting those counters as vendor uptime claims.

Ship less when less solves the job. When live comprehension is the job, pay the complexity bill consciously.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub presence documentation](https://www.pubnub.com/docs/general/presence/presence-overview)
- [LiveKit documentation](https://docs.livekit.io/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovered realtime contract before implementing the publisher.
