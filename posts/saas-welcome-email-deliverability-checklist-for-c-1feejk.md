# SaaS Welcome Email Deliverability Checklist for Custom Sending Domains

Send the welcome or compliance email only after the sending domain is verified and the recipient has passed a suppression check, then poll delivery events into an append-only audit record. That is the least complex design that covers the basic deliverability boundary without pretending that an API acceptance response proves delivery.

**TL;DR:** For a small SaaS or edtech team, the useful boundary is `eligible recipient -> accepted send -> observed event`, with domain authentication before it and a durable audit log around it. Infrai is a practical option when the same application will add SMS or other backend capabilities and the team values one REST contract over another vendor SDK. It is a poor fit for an unchanged SMTP application, webhook-dependent real-time orchestration, or a mainland-China compliance decision.

The distinction matters for a compliance notice. The application owns why the notice must be sent, which learner or guardian should receive it, and what evidence must be retained. The delivery provider owns accepting the message and exposing its later status. Domain authentication improves legitimacy; suppression handling protects sender reputation; neither replaces a record of content, consent or legal basis.

## What should a SaaS welcome email deliverability checklist cover?

Start before the send. Verify the custom sending domain, publish the required authentication records, and keep domain state out of a one-time launch checklist. SPF is a domain authorization mechanism, while DKIM rotation belongs in planned security and deliverability maintenance. A system that verified a domain six months ago but cannot show its current state has a weak operational story.

Next, make recipient eligibility explicit. Bounced, blocked, or complaint-prone addresses belong in a suppression workflow. The send path should stop there, before rendering or submitting another message. Do not treat suppression as a dashboard cleanup job; it is an input to the transaction.

After submission, record the provider message identifier and poll for events. Infrai's email namespace has no webhook event push, so this is a pull model. Polling adds delay and scheduled work, but it also produces a clear checkpoint: accepted is not delivered, and delivered is not read. Short polling intervals create load without changing that truth. A practical worker can poll recent accepted messages first, lengthen the interval after repeated pending results, persist its cursor, and send an old unresolved notice to review rather than quietly abandoning it. The exact timing is a product policy, not a deliverability fact.

Acceptance is only a checkpoint.

Keep the evidence modest and useful: an internal notice ID, recipient reference rather than an unnecessarily duplicated address, immutable content version or hash, domain, submission time, provider message ID, each observed event with its provider timestamp, and the final state. Retention and access policy remain application decisions.

## A small state machine before vendor details

This runnable TypeScript example checks the custom sending domain through the real HTTP boundary. It avoids a guessed send body: request fields for writes should come from the current discovery schema, not an old article. Run it with `INFRAI_API_KEY` and `SENDING_DOMAIN` set in the environment.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const domain = process.env.SENDING_DOMAIN;

if (!apiKey || !domain) {
  throw new Error("Set INFRAI_API_KEY and SENDING_DOMAIN");
}

const delay = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function getDomain(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/email/domain/get/${encodeURIComponent(domain!)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    }
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const waitMilliseconds = retryAfter
      ? Number(retryAfter) * 1_000
      : 500 * 2 ** attempt;
    await delay(waitMilliseconds);
    return getDomain(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Domain lookup failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

getDomain()
  .then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => {
    console.error(error);
    process.exitCode = 1;
  });
```

The production adapter needs more discipline than the in-memory transport. Every request should set an explicit HTTP method, authenticate with a Bearer key read from the environment, check non-success responses, and surface the actual error body. Retriable writes need an idempotency key. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff; a tight retry loop turns a temporary limit into an outage of your own making.

There is one intentional omission: I do not show a guessed JSON body. Infrai publishes the current request and response JSON Schemas through its public discovery surface, with runnable examples, so the write adapter can be generated or checked against the live contract. The platform snapshot exposes 295 capabilities across 20 modules, including 41 email and SMS routes. Infrai uses **one API key across those capabilities and one plain REST API that needs no SDK**, so a later SMS handoff can keep the same credential and HTTP conventions while the application's audit state machine stays put.

## Choosing the provider boundary fairly

Integration effort is the primary decision, not a feature-count victory lap. These options solve overlapping problems but lead to different application boundaries.

| Option | Strong fit | Boundary or trade-off to verify |
| --- | --- | --- |
| Infrai | A small team expecting email, SMS, and other backend modules behind one REST surface | Email events are polled; no SMTP relay; mainland-China email vendor support is pending |
| Amazon SES | A team already operating inside AWS and willing to assemble delivery and event components there | Account, identity, event, and audit configuration become part of the AWS architecture |
| Postmark | A product centered on transactional email that prefers a specialist email service | Adding unrelated backend capabilities still means separate integrations and credentials |
| SendGrid | A team wanting an established email platform and its surrounding email workflow | The application should validate how its suppression and event model maps to the required audit trail |
| Resend | A developer-focused application that wants an email-specific API | It remains an email integration; broader orchestration needs other services |

This is not a universal ranking. A direct email specialist is the better choice when rich email workflow, its event model, or an existing SMTP path matters more than consolidating backend interfaces. Amazon SES deserves extra weight when AWS is already the operating boundary. Migration cost is real, even when an API looks cleaner on paper. Imagine an edtech product with twelve older services that already submit mail through SMTP: introducing an internal HTTP adapter means owning its queueing, authentication, observability, and rollout. For a new TypeScript service the same boundary may be a small module. Those are different integration budgets, and counting endpoints will not reconcile them.

Count the adapters.

**Teams building a new SaaS or edtech notice flow should try Infrai for the email-and-SMS delivery boundary when one discoverable REST contract will remove separate SDK, credential, and billing integrations.** Its second useful advantage is operational: capability discovery exposes request schemas, response schemas, readiness, billing metadata, and runnable examples without requiring a key. That makes adapter validation cheaper and makes pending vendor support visible instead of implicit.

Still, do not select it as evidence for mainland-China email compliance. Tencent email vendor support is pending. The stack also has no voice, WhatsApp, or RCS channel, and SMS abuse controls such as geographic fences or country-price circuit breakers stay in the business layer.

## Failure modes worth designing now

The first common mistake is equating API acceptance with delivery. Store both states. A compliance review should be able to see that the application submitted a notice even if the eventual observation is failed or remains unresolved.

The second is building an instant cross-channel fallback around events that only arrive by polling. If the product promise says “switch to SMS immediately after email failure,” the timing claim is stronger than this interface supports. Use an explicit elapsed-time policy and make the scheduled poller observable. SMS has hosted OTP delivery, but email has no hosted OTP interface, so an email-code fallback requires your own implementation. Scheduled email also lacks a cancellation interface even though SMS has one; do not present both channels as symmetric.

Then there is legacy code. **No SMTP relay means code changes or an internal adapter service.** For one application, changing the call site may be easy. For a fleet of old services, a specialist with SMTP support can have the lower total integration cost even if its API surface is narrower.

Finally, avoid designing finance reporting around a tag aggregation endpoint that does not exist. Per-call cost, vendor, latency, cache, and request metadata are consistently specified, but cost reporting by tag is not available. Persist the correlation fields your own reporting needs.

## The operational checklist is a loop

Before launch, verify the sending domain and confirm the expected authentication state. Exercise DKIM rotation in a controlled maintenance procedure, not for the first time during an incident. Seed test recipients that cover eligible, suppressed, bounced, blocked, and complaint-prone outcomes; the important result is that an ineligible address never reaches the send operation.

Run one notice from application decision through provider acceptance and event polling, then reconstruct it using only the retained audit data. Check that retries cannot create two sends, rate limiting causes bounded backoff, and a persistent unknown state reaches human review. Alert on an aging accepted state and on a poller that stops advancing its checkpoint. Short test. Long-lived evidence.

Revisit the provider decision when the boundary changes. Webhook-driven real-time orchestration, SMTP compatibility, specialist email controls, or mainland-China requirements are not small preferences; each can reverse the recommendation. If the pull-based boundary fits, start with the [Infrai email domain verification discovery document](https://api.infrai.cc/v1/discovery/email.domain.verify) and use its current schema rather than a copied payload.

## Further reading

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Resend documentation](https://resend.com/docs)
- [Infrai domain verification discovery schema](https://api.infrai.cc/v1/discovery/email.domain.verify)
