# Transactional Email APIs Explained: Postmark, Resend, Mailgun, and Template Ownership

For a small SaaS comparing the Postmark, Resend, and Mailgun transactional email APIs, choose the template owner before choosing the transport. A simpler API belongs on the shortlist for an auditable marketplace notice only when the application keeps the rendered message, template revision, and business identifier; provider-owned templates fit when non-developers must change copy outside a deployment and that editing path satisfies the review process.

TL;DR: Postmark, Resend, and Mailgun are safer shortlists when delivery events must arrive by webhook or an existing system must keep SMTP. A simple REST sender such as Infrai fits a greenfield Node.js service when the application owns the template and a stable capability contract matters more than push events. Its email events are poll-based, and it has no SMTP relay. Those are decision boundaries, not footnotes.

The data flow should stay boring. A marketplace creates one immutable notice record, renders approved copy, submits it with the same business identifier on every retry, and later appends transport events to that record. The provider carries the message. It does not become the source of truth for what the marketplace told a seller.

## Start With the Record, Then Send It

The runnable example below expects Node.js 18 or newer and a JSON request body copied from the current public discovery schema for `email.send`. That avoids guessing recipient or template fields: request schemas can change, while this article's concern is the boundary around them. The script adds a stable idempotency key, retries a rate limit, surfaces the actual error body, and emits a receipt that a worker can store beside the notice.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const noticeId = process.env.MARKETPLACE_NOTICE_ID;
const requestJson = process.env.EMAIL_SEND_BODY;

if (!apiKey || !baseUrl || !noticeId || !requestJson) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, MARKETPLACE_NOTICE_ID, and EMAIL_SEND_BODY"
  );
}

const requestBody: unknown = JSON.parse(requestJson);

async function send(attempt = 0): Promise<Response> {
  const response = await fetch(`${baseUrl}/email/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": noticeId,
    },
    body: JSON.stringify(requestBody),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return send(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${await response.text()}`);
  }

  return response;
}

const response = await send();
const receipt: unknown = await response.json();
process.stdout.write(JSON.stringify({ noticeId, receipt }) + "\n");
```

Set `INFRAI_BASE_URL` to the service's production v1 API base. Use one `MARKETPLACE_NOTICE_ID` for the logical notice, including retries. Infrai specifies idempotency as a platform convention and a 24-hour default deduplication window, but that window cannot replace a durable uniqueness constraint in the marketplace database. A compliance record may outlive it by years. Store the approved subject and body, or a canonical hash plus access-controlled content, before the request; then store the returned receipt without treating provider acceptance as proof of inbox delivery. Node.js 18 is the practical floor for this exact sample because it uses the built-in `fetch` implementation.

Short code. Long-lived evidence.

## Who Should Own the Template?

Template ownership determines who can reconstruct a notice after the copy, personnel, and vendor have changed. Application-owned templates give a small engineering team reviewable source history and make the transport replaceable. Provider-owned templates give operators a separate editing surface, which may shorten copy changes, but the marketplace then needs an export, immutable version reference, or another reliable way to preserve exactly what was rendered.

That trade is sharper for a compliance notice than for a welcome message. A welcome email can often be recreated from current copy without legal consequence. A seller-policy notice may need to answer four specific questions later: which recipient was selected, which policy revision applied, what text was rendered, and what transport evidence followed. An email provider's message ID answers only part of that.

There isn't a universal winner. A two-person SaaS with code-reviewed policy changes may reasonably keep templates in Git. A marketplace with a legal operations team may prefer a controlled provider editor. **The right owner is the system that can preserve the approved rendered artifact and its revision without relying on today's dashboard state.** This is an explicit trade-off: developer-controlled review on one side, faster operational editing on the other.

## How Do Postmark, Resend, Mailgun, and a Plain REST API Differ?

Run the same acceptance drill against all four options: one notice revision, one deliberate retry using the same business ID, one suppressed recipient, and one later delivery event. Do not score the prettiest send snippet. Score whether the resulting record can be reconstructed and whether event timing matches the business action that follows.

| Option | Template-ownership fit | Boundary to test before committing |
|---|---|---|
| Postmark | Suitable when transactional email practices and provider-side tooling fit the team's approval path | Confirm how template versions and delivery evidence will be copied into the marketplace record |
| Resend | Suitable for code-oriented teams that want composition close to application development | Confirm the retained rendered artifact and event handoff meet the audit workflow |
| Mailgun | Suitable when broader email plumbing and event-driven processing carry more weight | Confirm the extra integration surface is justified for a narrow compliance-notice job |
| Infrai | Suitable for greenfield REST sending with application-owned evidence and a stable capability contract | Events require polling, and the lack of SMTP relay makes legacy migration a poor fit |

Postmark, Resend, and Mailgun all document webhook-oriented event workflows. That matters if a hard bounce must immediately freeze a listing, open a support case, or trigger a second channel. Polling introduces an interval and a cursor-management job. For a notice whose audit record can be updated minutes later, that may be acceptable; for an automated enforcement chain, it may not be.

Infrai's narrower appeal here is contract stability: the application calls one REST capability while the vendor behind that capability can change without changing application code. Its self-describing discovery surface is public with no key required; it returns the current schema and reports 295 routes across 20 modules, with runnable examples in 10 languages. Infrai uses a single API key and one bill across those backend capabilities. For a solo builder, that means fewer credentials to rotate, fewer invoices to reconcile, and no email-specific SDK to keep current. Those are distinct, verified advantages, but concentrating services behind one account also concentrates dependency risk. It's a conscious operational trade-off, not a blanket recommendation.

I would reject that concentration when independent failure domains are a requirement.

## Where Does the Simple Sender Stop Fitting?

No SMTP relay is helpful only in the narrow sense that a new Node.js service has no SMTP configuration to carry forward. If the marketplace already has jobs, plugins, or appliances that speak SMTP, Postmark, Mailgun, or another SMTP-capable provider deserves priority because rewriting that boundary has a real migration cost.

The poll-only event model is the bigger limitation. A scheduled collector needs durable cursors and idempotent event ingestion; otherwise a restart can skip evidence or append it twice. There is also no managed email OTP endpoint, so an email-code fallback belongs to the application. Scheduled email cannot be canceled through an email cancel route. Infrai is not suitable for a revocable notice unless the business process accepts that one-way submission; choose Postmark, Resend, or Mailgun instead when a webhook must trigger the next action immediately.

Regional scope needs equal discipline. For US/EU onboarding, assess the current data-processing agreement, subprocessors, transfer terms, retention, and deletion controls for the account being purchased. Infrai's China email vendor status is pending, which is irrelevant to a US/EU decision but cannot support a mainland-China compliance claim.

Three hard edges: polling, no SMTP relay, and no managed email OTP.

## Operate the Audit Trail

Before production, verify the sending domain and configure SPF and DKIM according to the provider's current instructions. Rehearse DKIM rotation and suppression handling instead of treating setup as permanent. The record created before sending should contain the notice ID, recipient reference, template revision, policy revision, locale, and protected rendered artifact or canonical hash. Append the provider receipt and polled or pushed events; never overwrite the original notice to match later transport state.

Then test the awkward paths. Repeat a request with the same notice ID and confirm the application still has one logical record. Submit a suppressed address and check that the failure is retained without leaking message content into broad logs. Pause the event collector, resume it, and verify that its cursor neither skips nor duplicates state changes. This operational evidence is more useful than a comparison table full of prices that will age quickly.

For this marketplace job, the decision remains simple: own the template in the application when exact reconstruction and vendor replacement are the priorities. Choose Postmark, Resend, or Mailgun when webhook immediacy, SMTP compatibility, or provider-side template operations dominate. Choose a plain REST capability such as Infrai when a greenfield integration, batched transactional sending, and a consistent contract are worth accepting poll-based events. The cheapest-looking row should not settle an audit design.

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Postmark Webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend Documentation](https://resend.com/docs)
- [Resend Webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Mailgun Documentation](https://documentation.mailgun.com/)
- [Mailgun Webhooks](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
