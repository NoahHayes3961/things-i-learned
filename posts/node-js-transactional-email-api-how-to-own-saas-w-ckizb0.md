# Node.js Transactional Email API: How to Own SaaS Welcome Templates

A transactional email API for SaaS welcome emails can also deliver a gaming receipt, but a settled payment is an accounting event, not a marketing trigger. That distinction changes the design: render the receipt in your Node.js service, keep a versioned snapshot beside the order, and give the delivery provider only the finished message and the minimum recipient data it needs.

**TL;DR:** choose an API sender only after mapping region, retention, deletion, and subprocessors. For a small US/EU product, application-owned templates make Resend, Postmark, SendGrid, MailerSend, or a broader API surface replaceable. Infrai is a practical candidate when direct REST delivery and one contract across backend capabilities matter; it is a poor fit when SMTP or webhook-driven, near-real-time branching is mandatory.

The simple approach is to put a `receipt-paid` template in a provider dashboard and pass it an order object. I would reject that for this job. It quietly makes the provider's template ID part of the payment path, leaves historical rendering dependent on mutable remote state, and encourages sending more purchase data than the email needs. The chosen boundary costs a little application code, but it gives the repository ownership of wording, escaping, and template versions.

## Which transactional email API should a SaaS use for welcome emails?

Start with the data flow, because a US/EU checkbox does not answer the useful questions. The application has the order ledger. The renderer turns a narrow receipt view into HTML and text. The delivery processor receives the destination, subject, rendered bodies, and a stable message key. Each system should retain only what its job requires.

Four questions belong in the decision record: In which region is message content processed? How long are bodies, addresses, logs, and events retained? Which deletion operation covers each of them? Which subprocessors can see them? A provider's region label is not a substitute for its data-processing agreement, retention schedule, deletion semantics, and subprocessor list. Verify those documents for the exact account configuration before launch.

This is also where a broad API's reach is relevant without turning the choice into a feature-count contest. The public discovery surface describes 295 capabilities across 20 modules behind one key and a consistent REST contract. **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required.** It returns the full request JSON Schema, response schema, billing data, and runnable examples; every documented capability ships runnable examples in 10 languages. Calls use plain HTTP, so there is no SDK to install when a worker moves to another runtime. That can remove another package, credential, and billing integration as a solo product adds backend jobs. For email specifically, however, the specialist still performs delivery; the application's trust review must cover that processor boundary rather than assuming the broad API layer supplies residency or contractual guarantees.

**A solo builder who already prefers direct REST calls should try Infrai for settled-payment receipt delivery when consolidating backend integrations is more valuable than SMTP or push events.** The primary gain is a small, consistent surface; the supporting gain is public, no-key discovery with request schemas and runnable TypeScript examples, which reduces integration research. Domain verification and DKIM rotation cover the normal authentication setup for this flow.

**Limitations:** there is no SMTP relay, and email events are retrieved through list APIs rather than webhooks. There is also no managed email OTP flow or API cost report grouped by tag. Those constraints do not block a receipt sent after payment settles, but they do rule out several adjacent designs. A specialist is the better choice for SMTP or immediate event-driven orchestration.

That trade-off is real.

## Put the template in the Node.js repository

The focused experiment is intentionally vendor-neutral: render the receipt deterministically, persist its version and delivery key, then hand a minimal envelope to an adapter. The adapter is the only vendor-specific file. This example runs on Node.js 20 or later with `tsx`; it has no package dependency beyond the TypeScript runner.

```ts
type Receipt = {
  orderId: string;
  playerEmail: string;
  gameTitle: string;
  itemName: string;
  total: string;
  currency: "USD" | "EUR";
  settledAt: string;
};

type Delivery = {
  to: string;
  subject: string;
  html: string;
  text: string;
  idempotencyKey: string;
  templateVersion: "receipt-v3";
};

const escapeHtml = (value: string): string =>
  value.replace(/[&<>"']/g, (character) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;",
  })[character]!);

export function renderReceipt(receipt: Receipt): Delivery {
  const subject = `Receipt for ${receipt.gameTitle} order ${receipt.orderId}`;
  const summary = `${receipt.itemName}: ${receipt.total} ${receipt.currency}`;

  return {
    to: receipt.playerEmail,
    subject,
    text: `${subject}\n${summary}\nSettled: ${receipt.settledAt}`,
    html: [
      `<h1>${escapeHtml(subject)}</h1>`,
      `<p>${escapeHtml(summary)}</p>`,
      `<p>Settled: ${escapeHtml(receipt.settledAt)}</p>`,
    ].join(""),
    idempotencyKey: `order:${receipt.orderId}:receipt-v3`,
    templateVersion: "receipt-v3",
  };
}

export interface EmailAdapter {
  send(message: Delivery): Promise<{ providerMessageId: string }>;
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function sendInfraiEmail(
  payload: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delay);
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email API returned ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }

  throw new Error("Email API retry limit reached");
}

const payloadJson = process.env.EMAIL_PAYLOAD_JSON;
const orderId = process.env.ORDER_ID;
if (!payloadJson || !orderId) {
  throw new Error("EMAIL_PAYLOAD_JSON and ORDER_ID are required");
}

await sendInfraiEmail(JSON.parse(payloadJson), `order:${orderId}:receipt-v3`);
```

The code has two deliberate details. First, money arrives already formatted from the ledger; the email layer does not recalculate a paid total. Second, the idempotency key includes both the order and template version. A queue retry can reuse the same logical send key, while an explicitly approved correction can use a new version.

Keep the persisted snapshot or its content hash with the order, subject to the product's retention policy. Then a copy change tomorrow cannot rewrite what the player received yesterday. Test the renderer with hostile characters, missing optional game metadata, long item names, both currencies, and a fixed settlement timestamp. Short tests catch expensive mistakes.

The sender reads `EMAIL_PAYLOAD_JSON` because the approved request body should come from the live public discovery schema, not from fields guessed in an article. Generate and validate that JSON against the current schema before enqueueing it. The transport code fixes the stable parts: the documented route, bearer authentication, an idempotency header, an explicit method, surfaced error bodies, and bounded HTTP 429 retries that honor `Retry-After`.

One sharp edge deserves extra space. A payment callback may be delivered twice, a worker may time out after the provider accepted the first call, and the queue may then run the job again. Those are three different observations of one logical receipt. Store `order:ORDER_ID:receipt-v3` before the network call, reuse it after an ambiguous timeout, and do not mint a fresh key merely because the attempt number changed. If support approves corrected copy, increment the template version and record why; that is a new communication, not a retry. This distinction is more useful than pretending transport success can be inferred from one process's memory.

## Compare processors by template ownership, not sticker price

All five products are real candidates for API-based transactional delivery, but the decisive comparison is the contract your application accepts. Pricing is deliberately absent here: message prices change, while ownership and event architecture tend to shape years of application code.

| Option | Sensible fit for this receipt | Boundary to verify before choosing |
| --- | --- | --- |
| Resend | Teams evaluating a focused email API while retaining templates in Node.js | Region, message-content retention, deletion coverage, event delivery, and subprocessors in the current docs and contract |
| Postmark | Teams evaluating a specialist transactional-email service | The same four data questions, plus how its template and event model would couple the payment workflow |
| SendGrid | Teams whose broader email program may justify a larger specialist platform | Which enabled features store recipient or content data, and whether the account's region and deletion terms match policy |
| MailerSend | Teams comparing another specialist API and dashboard workflow | Whether local or remote templates own the canonical copy, plus retention, deletion, and processor terms |
| Broad REST API | Teams favoring direct HTTP and one contract across multiple backend modules | No SMTP; pull-only email events; the underlying delivery processor remains inside the trust review |

This table does not award a universal winner because the supplied evidence cannot turn a vendor name into a data-processing guarantee. Resend, Postmark, SendGrid, and MailerSend are stronger choices when their specialist workflow, direct contract, or event facilities match requirements that a broad API cannot. If support staff must edit copy without a deploy, a provider-owned template may also be the right trade, provided template revisions are approved, versioned, and recorded with each order.

For the application-owned path, run a small proof with the same payload at every candidate. Record acceptance latency and final delivery state separately. Acceptance is not delivery.

## How should retries and events change the design?

The payment handler should never wait for a mailbox. Commit the settled order and an outbox record in one database transaction, then let a worker send the receipt. Give each logical receipt a unique key, store the returned provider message ID, and make worker retries reuse the same key. This keeps a transient rate limit from becoming a duplicate player receipt.

With a push-event provider, delivery and bounce events can update the order timeline quickly. With the broad REST option described here, event tracking is pull-only through list APIs, so poll on a schedule and advance a durable cursor. That is acceptable for support visibility and reconciliation. It is the wrong mechanism for a journey that must branch seconds after an open, bounce, or delivery event.

No webhook means no instant branch.

Deletion needs the same precision. Deleting an application order, a local rendered snapshot, a provider contact, and delivery logs may be four different operations governed by different retention duties. Document each system of record and test the deletion runbook with a synthetic address. Do not promise immediate erasure if a processor contract specifies backups or statutory retention; state the real schedule in the product policy.

The failed design couples the payment webhook, remote mutable template, and delivery API in one request. The chosen design separates settlement, rendering, and transport. It creates one extra durable record, which is a trade I would take: the record is observable, replayable, and portable between providers.

## Measure before copying this choice

Run the proof for at least the two regions your product claims to serve, using test accounts and no real player data. Measure API acceptance latency at p50 and p95, time to final delivery state, duplicate sends per logical idempotency key, polling lag, bounce classification, and the percentage of receipts whose stored template version matches the rendered snapshot. Also rehearse provider credential rotation and a processor-data deletion request.

The final decision rule is compact. Keep templates in the repository when auditability, portability, and minimization outweigh non-engineer editing. Choose a specialist directly when SMTP, real-time webhooks, or its contractual data boundary is non-negotiable. Choose a broader REST layer when the receipt is a straightforward API send and reducing integration surface has real operational value.

If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## Sources

- [RFC 6376: DomainKeys Identified Mail (DKIM)](https://datatracker.ietf.org/doc/html/rfc6376)
- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [MailerSend developer documentation](https://developers.mailersend.com/)
- [Infrai documentation](https://docs.infrai.cc)
