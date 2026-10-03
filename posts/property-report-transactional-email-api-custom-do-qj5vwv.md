# Property Report Transactional Email API: Custom Domain and Suppression Failure Drills

Short answer: send each generated property report through a small, durable dispatch state machine. Store the report first, assign one delivery key, submit once, and reconcile provider events before retrying. For a beginner comparing MailerSend, Amazon SES, and a simpler transactional API such as Postmark, the decisive test is not the advertised entry price. It is whether custom-domain authentication, suppression handling, attachment limits, and delivery events fit this recovery path without hidden manual steps.

The flow is plain: a property manager approves a tenant-facing report, the application freezes that report as an immutable object, and a worker submits an email that references its checksum and dispatch ID. Provider events update delivery evidence later. The user-facing request never waits for the mail provider, and an uncertain response never triggers a blind second send.

Retries need evidence.

## Can a retry send the same inspection report twice?

Yes, unless the application owns idempotency. A network timeout only says that the caller did not receive a response; it does not prove that the provider rejected the message. Retrying immediately can produce two emails with the same attachment. For a welcome note, that is untidy. For a move-out inspection report, duplicated delivery can confuse a dispute timeline.

Treat submission and delivery as separate facts. `accepted` means the provider accepted a request. It does not mean the recipient mailbox accepted the message, and it certainly does not mean a person read the attachment. Keep those meanings separate in the schema and operations screen.

Names matter here.

The least complex useful model has six application states: `ready`, `submitting`, `accepted`, `delivered`, `failed`, and `suppressed`. A suppression event is not a generic failure; it should stop automatic retries until the address or consent state changes.

## Build the narrow boundary first

This TypeScript example keeps provider-specific code behind one interface. The durable store is intentionally abstract because the important property is the compare-and-set transition, not a particular database. The worker can crash after submission; on restart, it reconciles the existing attempt instead of manufacturing a fresh one.

```ts
type DispatchState =
  | "ready"
  | "submitting"
  | "accepted"
  | "delivered"
  | "failed"
  | "suppressed";

type ReportDispatch = {
  id: string;
  propertyId: string;
  recipient: string;
  reportUrl: string;
  reportSha256: string;
  state: DispatchState;
  providerMessageId?: string;
};

interface MailTransport {
  submit(input: {
    idempotencyKey: string;
    from: string;
    to: string;
    subject: string;
    attachment: { url: string; sha256: string; filename: string };
  }): Promise<{ messageId: string }>;
}

interface DispatchStore {
  claimReady(id: string): Promise<ReportDispatch | null>;
  markAccepted(id: string, messageId: string): Promise<void>;
  markFailed(id: string, reason: string): Promise<void>;
}

async function submitReport(
  dispatchId: string,
  store: DispatchStore,
  mail: MailTransport,
): Promise<void> {
  const job = await store.claimReady(dispatchId);
  if (!job) return;

  try {
    const result = await mail.submit({
      idempotencyKey: job.id,
      from: "reports@notices.example",
      to: job.recipient,
      subject: `Property report ${job.propertyId}`,
      attachment: {
        url: job.reportUrl,
        sha256: job.reportSha256,
        filename: `property-${job.propertyId}-report.pdf`,
      },
    });
    await store.markAccepted(job.id, result.messageId);
  } catch (error) {
    const reason = error instanceof Error ? error.message : "unknown error";
    await store.markFailed(job.id, reason);
  }
}
```

There is a deliberate trade-off here. The interface asks for an idempotency key, but the application still owns the claim record because provider behavior may differ. That extra row and transition cost less operational attention than investigating duplicate notices later. Keep the attachment checksum too: it lets an operator prove which frozen report the dispatch referenced without opening or regenerating the document.

Do not log the attachment body or full report URL. Log the dispatch ID, property ID, checksum, state transition, provider message ID, and event timestamp. Those fields can trace the pipeline while reducing the tenant data copied into logs.

Keep the payload out.

## How should a beginner test a transactional email API?

A feature grid makes three products look comparable while hiding the path that matters. Run the same drill against MailerSend, Amazon SES, and Postmark instead. Use a domain reserved for testing, one controlled recipient, and a harmless two-page PDF. Do not infer production reliability from a successful quickstart.

| Test | Evidence to retain | Reject the integration when |
| --- | --- | --- |
| Authenticate the sending domain | DNS records and a passing check | Ownership needs a shared manual account |
| Submit one report | Dispatch ID mapped to message ID | The mapping cannot survive later events |
| Repeat the same dispatch key | Message count and request history | The adapter silently creates a second attempt |
| Send to a suppressed test address | Suppression reason and state | The worker retries forever |
| Delay or reorder delivery events | Event ID, timestamp, final state | An older event moves a terminal state backward |
| Exceed the configured attachment ceiling | A classified permanent error | The job enters an unbounded retry loop |

This is a comparison of integration boundaries, not a ranking. Each candidate gets the same fixture and acceptance criteria. Record the setup steps that require console access, events available, how suppression status is exposed, and documented attachment constraints. Product documentation can change, so capture the date and link beside every observed result rather than copying a limit into permanent application logic.

Walk through the ugly case before scoring the products. The worker claims dispatch `rpt-1842`, submits it, and loses the connection before it can store the returned message ID. The row remains `submitting`. A second worker must not turn that uncertainty into another message; it should place the dispatch in reconciliation, where an event or provider-side lookup can connect the original attempt to the durable record. If the provider offers no way to carry or recover the application key, record that as an operational limitation in the comparison. Then run the same sequence with the recipient already suppressed. The desired outcome is different but equally strict: no attachment leaves the system, the dispatch becomes `suppressed`, and an ordinary retry timer cannot revive it. This single drill exposes more than a polished send screen because it tests ambiguity, duplicate control, evidence retention, and the boundary between transient and permanent failure together.

That is the expensive moment.

Cost belongs in the decision, but as a constraint after the drill. Estimate the full monthly workload: normal messages, retry attempts, stored event data, and operator time. A tiny difference in message price is weak compensation for a recovery path that needs routine manual repair.

## Make events safe to replay

Provider callbacks are input, not commands. Verify them using the selected provider's documented mechanism, store each raw event once under its event ID, and then project it onto the dispatch. Return success only after durable storage. If processing fails later, replay the stored event; do not ask an operator to reconstruct it from a dashboard screenshot.

Event ordering needs a rule. `delivered` must not fall back to `accepted` because an older acceptance event arrived late. A permanent suppression can stop future attempts for that recipient, while a transient submission error can return to a bounded retry schedule. The exact categories must follow documented provider responses, but application state names should stay stable across adapters.

Replays are normal.

Google's sender guidance makes domain authentication and responsible sending behavior part of the delivery baseline. Set up authentication before measuring inbox placement, keep the visible sender aligned with the domain the property company controls, and monitor rejection signals. A custom domain is part of the trust and ownership boundary.

One practical constraint remains: email and SMS are not interchangeable fallbacks. SMS length and segmentation vary with GSM-7 versus UCS-2 encoding, so a report email cannot be converted into a long text message without changing both content and delivery behavior. Use SMS, if required, as a short notification pointing the recipient back to an authenticated report experience. Never squeeze report contents into an automatic fallback.

## Operate the boring path

Before release, freeze a representative PDF, verify its checksum through generation and submission, and exercise accepted, delayed, suppressed, and permanent-failure outcomes. Confirm that two workers cannot claim the same `ready` row. Then replay every stored callback twice and in reverse order. The final state should remain correct.

In production, alert on age and state, not raw error volume: a `submitting` record that remains unresolved deserves attention, as does a growing queue of reports that never reach `accepted`. Review suppression changes separately from transient transport failures. Keep retry counts bounded and send exhausted jobs to an operator-visible queue with the evidence already attached.

Ship the smallest flow that passes this drill. A beginner-friendly dashboard may shorten setup, while a lower-level service may expose more knobs; neither quality answers whether a property report survives a timeout, replay, or suppression event. The durable dispatch record does.

The state wins.

## Further reading

- Google, Email sender guidelines: https://support.google.com/a/answer/81126
- Twilio, SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit
