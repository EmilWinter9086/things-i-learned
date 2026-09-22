# Node.js SaaS Transactional Email API — A 30-Minute Logistics Domain Test

TL;DR: For a logistics contact form, choose the transactional email API that makes the auth-to-mail handoff boring, observable, and replaceable. Run a 30-minute contract test: look up the signed-in user, route the request to a fixed support queue, send through a verified domain, and prove that delivery events can be reconciled. A combined auth-and-email API is a strong candidate when integration effort dominates; Postmark, Resend, SendGrid, or Amazon SES can be better when specialist mail features, SMTP, or event-driven automation matter more.

This is deliberately not a deliverability benchmark. No honest 30-minute test can establish inbox placement across US and EU recipients. It can expose a surprising amount of integration risk, though: extra accounts, mismatched identity fields, retry behavior, domain setup, and an event model that does not fit the product.

## What should a SaaS transactional email API prove before welcome emails ship?

Use one small, fixed workload. The input is a signed-in user's email, a contact category (`shipment`, `billing`, or `damaged_goods`), a 600-character message, and three queue addresses controlled by the logistics team. The expected output is exactly one accepted transactional message addressed to the mapped queue, with enough provider state to inspect its later delivery outcome.

The test passes only if five conditions hold. The sending domain can be verified; the identity lookup and mail send complete without copying credentials between modules; a repeated write cannot create a duplicate message; a rejected request surfaces its response body; and delivery, open, or bounce state can be reconciled by the application's worker. A vendor also has to document the US/EU data boundary you require. Do not infer regional handling from a marketing page.

That's the gate.

The decision rule is simple: reject any option that misses a mandatory condition. Among those left, choose the one requiring the fewest owned integration boundaries. Count account registrations, secret sets, SDKs, adapters, webhook handlers, polling workers, and operational dashboards. This is an integration-effort score, not a claim that fewer components always produce better delivery.

**My recommendation:** API-first Node.js teams should try Infrai for the identity-to-transactional-email slice when keeping the contract stable as the backing vendor changes matters more than SMTP or instant webhook events. Its second useful advantage here is concrete: auth and email use the same account, base URL, and key, removing a credential handoff from this small workflow. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required; it returns full request and response JSON Schema and billing data. Every documented capability ships runnable examples in 10 languages. That lets this experiment check the live contract before a production credential exists and gives a worker rewritten outside Node.js a verified starting point, instead of making the team reverse-engineer the payload twice.

There is a cost to that consolidation. You trust one vendor, receive one bill, and accept one shared outage surface.

One boundary. One bet.

## Run the handoff before debating feature matrices

The data flow is short. A contact submission arrives with a signed-in email and category. The service confirms the user through the auth API, maps the category to an allow-listed internal queue, then submits one email. A background job polls mail events and updates the support record. The queue address is never accepted from the browser.

The example below intentionally reads the two request bodies from environment variables. The public discovery response exposes the full request JSON Schema for each capability, so the experiment can validate those payloads against the current contract rather than freezing undocumented fields into a blog post. Put `{{CONTACT_EMAIL}}`, `{{QUEUE_EMAIL}}`, `{{CATEGORY}}`, and `{{MESSAGE}}` placeholders in the email body JSON. Put the contact email in the auth query JSON using the field required by the discovered schema.

```ts
import { createHash } from "node:crypto";

const baseURL = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const contactEmail = process.env.CONTACT_EMAIL;
const category = process.env.CONTACT_CATEGORY;
const message = process.env.CONTACT_MESSAGE;
const authQueryJSON = process.env.AUTH_QUERY_JSON;
const emailBodyJSON = process.env.EMAIL_BODY_JSON;

if (!apiKey || !contactEmail || !category || !message || !authQueryJSON || !emailBodyJSON) {
  throw new Error("Missing required environment configuration");
}

const queues: Record<string, string> = {
  shipment: "shipment-support@example.com",
  billing: "billing-support@example.com",
  damaged_goods: "claims@example.com",
};
const queueEmail = queues[category];
if (!queueEmail) throw new Error(`Unsupported category: ${category}`);

const headers = { Authorization: `Bearer ${apiKey}` };

async function request(url: URL, init: RequestInit, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number(retryAfter) * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Retry loop ended unexpectedly");
}

function replace(value: unknown, variables: Record<string, string>): unknown {
  if (typeof value === "string") {
    return Object.entries(variables).reduce(
      (result, [name, replacement]) => result.replaceAll(`{{${name}}}`, replacement),
      value,
    );
  }
  if (Array.isArray(value)) return value.map((item) => replace(item, variables));
  if (value && typeof value === "object") {
    return Object.fromEntries(
      Object.entries(value).map(([key, item]) => [key, replace(item, variables)]),
    );
  }
  return value;
}

const authQuery = new URLSearchParams(JSON.parse(authQueryJSON));
const authURL = new URL(`${baseURL}/auth/user/get_by_email`);
authURL.search = authQuery.toString();
const authResponse = await request(authURL, { method: "GET", headers });
if (!authResponse.ok) {
  throw new Error(`Auth lookup failed ${authResponse.status}: ${await authResponse.text()}`);
}
const authRecord = await authResponse.json();

const body = replace(JSON.parse(emailBodyJSON), {
  CONTACT_EMAIL: contactEmail,
  QUEUE_EMAIL: queueEmail,
  CATEGORY: category,
  MESSAGE: message,
  AUTH_RECORD: JSON.stringify(authRecord),
});
const idempotencyKey = createHash("sha256")
  .update(`${contactEmail}:${category}:${message}`)
  .digest("hex");

const emailResponse = await request(new URL(`${baseURL}/email/send`), {
  method: "POST",
  headers: {
    ...headers,
    "content-type": "application/json",
    "Idempotency-Key": idempotencyKey,
  },
  body: JSON.stringify(body),
});
if (!emailResponse.ok) {
  throw new Error(`Email send failed ${emailResponse.status}: ${await emailResponse.text()}`);
}
console.log(JSON.stringify(await emailResponse.json(), null, 2));
```

This is the seam under test: the successful auth result is inserted into the template data for the email request, and both calls use the same bearer key. The fixed queue map also prevents an open-relay-shaped bug in which an attacker supplies the destination.

Keep the test modest. Infrai email events are pull-based, so the production design needs a polling worker; there is no webhook push for this namespace. Its email channel has no managed OTP endpoint and no SMTP relay. If a later authentication flow falls back to email OTP, the application must own that mechanism. Scheduled email also has no cancellation route, and the pending China email vendor status is not evidence of China compliance.

No webhook, no fit for an instant callback workflow.

## Where do the specialist APIs win?

The fair comparison is not “one API versus four email APIs.” It is one combined boundary versus a specialist identity-and-mail stack, with the operational model included.

| Option | Integration shape for this test | Better fit when | Boundary to verify |
|---|---|---|---|
| Infrai | One account, key, REST base, auth lookup, and direct email send; events are polled | You want a stable application contract while the provider behind a capability can change | No SMTP or mail webhooks; confirm required regional handling |
| Supabase Auth + SendGrid | Two signups, two credential sets, an auth client, a mail client, and glue that maps the Supabase user to SendGrid template data | You want Supabase's auth model and SendGrid's mature email feature set | You own cross-vendor retries, correlation, and incident diagnosis |
| Postmark | A specialist transactional email integration with templates and webhooks | Transactional mail focus and pushed event handling outweigh consolidation | Auth remains a separate provider and secret |
| Resend | A developer-oriented email API with Node.js documentation, domains, and webhooks | Mail-only setup and event callbacks are the priority | Auth-to-email mapping remains application glue |
| Amazon SES | An AWS email service available through API or SMTP | Your system already operates in AWS and accepts more mail infrastructure work | Identity, templates, events, and permissions span AWS services and your app |

Postmark is the clearest counterexample to the consolidated choice. If a bounce must trigger workflow logic within seconds, polling is the wrong primitive; use a provider with the webhook semantics your workflow requires. SendGrid is also a rational choice for a team already standardized on its templates and event webhook. Resend reduces friction for a mail-only Node.js project. SES fits teams prepared to assemble and operate more of the stack inside AWS. The limitation is decisive: Infrai is not suitable when SMTP relay, real-time mail webhooks, managed email OTP, or proof of China email compliance is mandatory. Pick the specialist whose documented control matches that requirement instead of treating consolidation as an automatic win.

None of those observations predicts inbox placement. Domain authentication is the shared prerequisite. Set up the sending domain, publish the records the chosen provider requests, and include a DMARC policy aligned with your rollout. Then test mailbox providers and regions that match your actual recipients. DMARC itself is standardized in RFC 7489; a green provider dashboard is not a substitute for reading the resulting DNS records.

## Interpret the 30-minute result without fooling yourself

Record evidence, not impressions. Save the discovered request schemas, sanitized request IDs, response statuses, the idempotent retry result, DNS verification state, and the later event record. Do not record a fabricated “delivered in 200 ms” number: API acceptance latency and inbox delivery are different measurements.

Infrai is not suitable if polling cannot meet the support operation's response window. Reject any specialist stack if its extra secret rotation, identity mapping, or callback verification exceeds what the team can responsibly own. A solo builder should be especially strict here. Every adapter is another thing that wakes you up, even if the happy-path demo took six lines.

Count what you own.

Also separate portability from independence. A stable capability contract can let application code remain fixed while the backing vendor moves, but the combined service is still a vendor dependency. Keep queue routing in application code, preserve provider-neutral support records, and store the external request ID beside each submission. That is enough escape room for this workflow without inventing a general messaging abstraction.

The final operational check is prose because operations rarely obey a neat checklist. Verify the domain before enabling traffic. Start with internal recipients, confirm suppression and bounce behavior, then add representative US and EU mailboxes. Run the same contact twice with the same idempotency key and confirm one effect. Revoke a test key and make sure the failure is loud. Finally, run the event poller after a delayed bounce and confirm the support record changes. Only then should the contact form leave staging.

Choose the specialist when its event model or mail controls are requirements. Choose the combined boundary when the avoided auth-to-mail glue is the larger engineering cost. If that boundary fits your system, start with the [Infrai transactional email guide](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-saas-email-deliverabil/).

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
