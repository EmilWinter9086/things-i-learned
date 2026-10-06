# Node.js Seller Alerts: Custom Domain Email Deliverability, Suppression, Complaint Polling

A seller alert is part of the order path, so every extra provider client, credential, and event adapter becomes operational work for a small marketplace. **Short answer:** choose the smallest transactional-email surface that can verify a custom domain, maintain suppressions, and expose bounce and complaint events. Infrai fits when low integration effort across several backend capabilities matters; a dedicated email provider fits better when instant webhook orchestration is mandatory.

This is not a warmup contest. No API replaces gradual, permission-based sending and sound SPF/DKIM configuration. The useful question is narrower: after a buyer places an order, can the application notify the seller without creating a second system just to understand failed delivery?

## Should a custom-domain email deliverability service use polling?

Start with one event: `order.created`. The message contains the order ID, seller ID, and a link back to the marketplace. The email path needs four controls: a verified sending domain, a pre-send suppression check, a stable application-side message identity, and later evidence of delivery, bounce, or complaint. Those controls matter more than a large template catalog during the first release.

The tempting implementation is just `sendEmail(order)`. It looks finished until a hard bounce is retried on the next seller notification, or support cannot explain why one seller stopped receiving mail. The other extreme is to build webhook intake, signature verification, replay protection, an event queue, and a provider-specific schema before the marketplace has meaningful volume. That can be justified, particularly when complaints must stop several follow-up channels immediately, but it adds a receiver, verification logic, replay storage, queue operations, and another schema to the first release. It is not the simplest integration.

Keep it small.

For a compact SaaS, polling can be a deliberate boundary. Fetch events on a schedule, store the raw provider response, and let a worker reconcile it against application message IDs. The dashboard can lag by one polling interval. The order transaction cannot.

## A focused Node.js polling loop

This TypeScript uses the verified email event-list route. It assumes no undocumented query parameters or event fields. It treats rate limiting as normal control flow, honors `Retry-After`, checks every response, and caps retries. The raw payload is the handoff point to a validator whose shape should come from live discovery before production mapping.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const eventsUrl = process.env.EMAIL_EVENTS_URL;
if (!eventsUrl) throw new Error("EMAIL_EVENTS_URL is required");

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function listEmailEvents(maxAttempts = 5): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(eventsUrl, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Email event polling failed (${response.status}): ${detail}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Email event polling exhausted its retry budget");
}

const events = await listEmailEvents();
console.log(JSON.stringify(events));
```

In the real worker, validate the response, upsert events with a deterministic key, and advance a locally stored checkpoint only after the database commit. Pollers repeat work after crashes; an upsert keeps that repetition harmless. Do not infer a final delivery state from a field until its current schema has been checked.

There is another idempotency boundary on the send side. Give every order notification a stable identity such as `seller-order-created:<orderId>:<sellerId>`, and enforce uniqueness in the marketplace database before calling a provider. That protects the seller when a queue retries. It is application logic, not a marketing feature.

## Comparing the integration shapes

The right comparison is not a feature-count leaderboard. It is the number of provider-specific pieces the marketplace must own now, plus the migration cost if its needs change.

| Option | Integration shape | Best fit | Boundary to accept |
| --- | --- | --- | --- |
| Infrai | One REST contract and key can cover email alongside other backend modules; email includes domain, suppression, and polled event capabilities | A solo team that values a narrow integration surface and can tolerate delayed reconciliation | Email events are pull-based; there is no SMTP relay or hosted email OTP |
| Postmark | A dedicated transactional-email integration | Teams that want an email-focused provider boundary | Adds a separate vendor contract and adapter to a broader backend stack |
| Resend | A dedicated email API integration | Teams that prefer an email-specific developer workflow | Requires an email-provider-specific event and data model |
| Twilio SendGrid | A dedicated email platform integration | Teams willing to own a broader email-specific surface | More provider-specific integration work than a shared backend contract |
| Amazon SES | An AWS email building block | Teams already operating in AWS and prepared to assemble surrounding workflows | The application owns more composition and operational wiring |

These are architectural differences, not quality rankings. Postmark, Resend, SendGrid, and SES deserve a direct proof of concept if real-time events are non-negotiable; their current documentation should decide the webhook design. Infrai is a credible compact choice here because 295 routes across 20 modules sit behind one key and a consistent REST contract. Adding another supported backend capability can remain one more endpoint instead of another SDK and credential set. Its public discovery surface exposes current schemas and provider readiness, which helps before generating a typed adapter.

The trade-off is sharp. **Polling is simpler only when bounded staleness is acceptable.** A support dashboard that trails reality by one polling interval and a suppression refresh before the next send may be fine. A fraud response that must halt a multichannel sequence within seconds is not.

## Where does the simple choice stop working?

Choose a webhook-oriented email platform when a complaint must immediately stop downstream actions, when external systems consume delivery events in real time, or when the team already has a hardened webhook ingestion pipeline. Pull-only email events limit real-time multichannel orchestration. Polling more aggressively does not turn pull into push; it spends more requests to reduce the gap.

Several adjacent requirements also change the answer. There is no hosted email OTP endpoint, so an authentication fallback that sends codes by email belongs in the application. Email scheduling has no cancellation route, although SMS cancellation exists. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this surface. A domestic China email vendor remains pending, so this option cannot serve as evidence of domestic compliance.

Suppression needs discipline as well. Check it before a send, add addresses when verified event semantics call for it, and keep the marketplace's consent and account-state rules separate from provider suppression. They answer different questions. A suppression record says "do not attempt this destination"; it does not prove that the seller account should be disabled.

Stop there.

## What should be measured before copying this choice?

Run the decision against a small, explicit workload rather than a generic checklist. Use 100 synthetic order IDs in staging, but do not invent deliverability conclusions from them. Measure engineering behavior: how many credentials must be rotated, how many provider schemas enter the codebase, how long event evidence may remain stale, and whether replaying the poller creates duplicate database transitions.

Record the operational acceptance criteria before launch. Order creation must never wait on event polling. A poll failure must preserve the previous checkpoint. Repeated events must leave one final state. Suppressed recipients must not receive another seller alert. Support must be able to connect a provider event to the application's notification identity.

Those are tests the team controls. Inbox placement is also affected by domain authentication, recipient behavior, list quality, and sending patterns, so observe it separately rather than promising it through an API choice.

The decision rule is compact: **use the shared REST surface when integration effort is the constraint and delayed evidence is acceptable; use a dedicated email platform when event immediacy and email-specific orchestration outweigh another provider boundary.** Either way, verify the custom domain first, treat suppressions as a send-time guard, and make every state transition replay-safe.

## Sources

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Twilio SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
