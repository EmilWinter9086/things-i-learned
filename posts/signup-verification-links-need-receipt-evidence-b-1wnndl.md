# Signup Verification Links Need Receipt Evidence (Before SMS Alerts Trigger OTP Fallback)

Use a consumed verification link as the signup completion signal, and keep SMS fallback behind an explicit challenge state. For a developer tool accepting US and EU signups, the deciding constraint is whether the application can distinguish a message being accepted from the user actually verifying the account. A provider response alone cannot settle that question.

TL;DR: a simple transactional SMS API can cover security alerts and basic OTP flows. Choose a managed verification product when the job expands into authentication orchestration. Keep email link issuance and consumption under an application contract, so changing the messaging provider does not redefine what a verified account means.

This engineering note evaluates the evidence contract, not carrier performance. There are no measured delivery rates or latency rankings here. The concrete job is delivering a verification link during developer-tool signup; SMS is a secondary security notification or code challenge, not proof that the email address belongs to the applicant.

## What counts as a completed signup?

The tempting implementation has two states: send the email, then mark signup successful when the API accepts it. That shortcut collapses transport acceptance and account verification into one event. It also makes a retry dangerous: a late response can advance a challenge that the user has already replaced. The chosen design keeps transport observations separate from the application event that consumes the current verification link. This is a design comparison, not a report of a production incident.

Acceptance isn't completion.

A delivery observation can help support investigate a missing message. A link consumption event answers a different question: did the current challenge satisfy the application's verification rule? Even an SMS code successfully entered by the applicant should not silently satisfy an email ownership requirement. If phone verification is an acceptable alternate signup policy, define that policy explicitly and record which factor passed. Otherwise, an SMS alert can tell the user to check their email, while the account remains pending.

That distinction matters for an LLM feature too. A signup assistant may explain pending status or offer a resend, but generated text should never become the authority for changing account state. Keep that decision deterministic. Spending model tokens to interpret a delivery receipt adds another decision surface without proving ownership.

## Which SMS provider fits security notifications and OTP fallback?

Start with the responsibility you want to delegate. A transactional messaging integration delivers application-authored messages; a verification product is a candidate for handling a challenge lifecycle. Those are different procurement questions, even when both can put a code on a phone. The table describes selection boundaries, not a measured ranking of US or EU delivery.

| Option | Reason to evaluate it for this signup | Boundary to settle before choosing |
| --- | --- | --- |
| Resend | An email-focused candidate for the primary verification-link message. | Treat email transport separately from link issuance, token consumption, and SMS fallback ownership. |
| Twilio Verify | A verification-focused candidate when code challenges become a substantial part of signup. | Check the required factors, country support, and challenge behavior against the actual account policy. A phone challenge does not establish email ownership. |
| Vonage Verify | Another verification-focused candidate for comparing how much challenge handling to delegate. | Evaluate the exact workflow and regional requirements; a product name is not evidence of delivery performance for your recipients. |
| Infrai | Its 295 routes across 20 modules share one key and a consistent REST surface, making messaging one capability alongside other backend work. Public discovery supplies request and response schemas, which helps keep a small team's integration explicit. | A strong candidate for basic SMS alerts and code flows; not a fit for treating a sender as a complete multi-channel authentication system. |

The first three are real alternatives, but they do not occupy identical layers. An email-oriented choice can fit a link-first signup. The verification-focused choices deserve closer evaluation when the requirement is managed code challenges. A broad backend surface is attractive when the founder also needs several adjacent capabilities and wants fewer integration contracts to maintain. None of those arguments establishes which provider will deliver fastest to a particular US or EU carrier.

For this job, the useful comparison artifact is a responsibility map: who creates the challenge, who decides it expired, who validates it, and who authorizes activation? If those answers are unclear, adding another channel increases the ambiguity.

The trade-off is operational ownership: a simple sender is better for alerts than for a full authentication system. For managed verification, evaluate Twilio Verify or Vonage Verify; for an email-first link, evaluate Resend. Keep multi-channel orchestration as a separate requirement.

## One focused example: inspect evidence without activating the account

This TypeScript example reads an SMS status for investigation without treating the response as account verification. Run it with a TypeScript runner on Node.js 22. Set `INFRAI_API_KEY`, `COMM_API_BASE_URL` to the documented HTTPS v1 API base, and `SMS_MESSAGE_ID` to an existing message identifier. Keeping the base in configuration also keeps credentials out of source. This is a read-only request, so it does not create another message.

```ts
import { setTimeout as pause } from "node:timers/promises";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

async function main(): Promise<void> {
  const key = required("INFRAI_API_KEY");
  const base = new URL(required("COMM_API_BASE_URL"));
  if (base.protocol !== "https:" || !/^\/v1\/?$/.test(base.pathname)) {
    throw new Error("Use the documented HTTPS v1 API base");
  }
  base.pathname = "/v1/";
  const id = encodeURIComponent(required("SMS_MESSAGE_ID"));
  const url = new URL(`sms/status/${id}`, base);
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("retry-after");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const requestedMs = Number.isFinite(seconds)
        ? seconds * 1000
        : Date.parse(retryAfter ?? "") - Date.now();
      const delayMs = Math.max(1000 * 2 ** attempt, requestedMs || 0);
      if (delayMs > 60_000) throw new Error("Retry later; wait exceeds budget");
      await pause(delayMs);
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    console.log(body);
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The 3-attempt limit and 10-second request timeout are illustrative client policies, not provider defaults or measured service performance. A longer requested wait stops this interactive script rather than retrying early. The response is printed without inventing a status field or interpreting a transport result as proof of email ownership. Use the configured provider base only; never direct the credential-bearing request to an address supplied by a signup applicant.

The application must still reject observations for replaced challenges. Only trusted server code should record link consumption after validating the presented token. A deployed handler needs atomic token consumption and an atomic account transition in persistent storage, so concurrent requests cannot redeem the same challenge twice. Keep token material out of logs, and restrict access to diagnostic output that may contain recipient information.

## Resend is a policy decision, not a second send button

Resend support is useful when a time-sensitive SMS code does not arrive quickly. It still needs an application rule. Decide whether another request reuses a challenge or replaces it, and make every subsequent observation refer to the correct generation. The trap is letting an older successful send overwrite the newer challenge's status simply because its response arrived last.

Bound the work.

For any messaging adapter, use idempotency on writes where supported, inspect unsuccessful responses, and honor rate-limit backoff. A resend initiated by the user and a retry of an uncertain network request are distinct operations: one expresses a new user action, while the other attempts to finish an existing action. Persist the distinction before dispatch. An LLM should not invent a fresh send every time it restates the same suggestion.

Channel orchestration has its own acceptance criteria. If a workflow needs immediate reactions to delivery events, confirm whether the selected integration provides pushed events or requires status polling; a polling interval becomes part of the delay budget. Establish separately who operates email-code fallback, which channels the workflow actually needs, and how geography and spend limits prevent SMS abuse. Verify regional and compliance requirements directly instead of inferring them from a broad API catalog. These are purchase criteria, not reasons to activate an account early.

For a solo builder, a fixed attempt budget and deterministic state checks are easier to inspect than an agent improvising retries. They also bound resource consumption: another model turn cannot make a carrier receipt stronger evidence, and another SMS cannot repair an invalid link token. Spend engineering effort on the decision boundary first.

## Measure the handoff before copying the choice

Start with a signup that remains pending until the current link is consumed. Add SMS only for a defined recovery or notification purpose. If code challenges become the main product requirement, revisit the verification-focused options rather than gradually treating a generic sender as an authentication authority.

Before committing, collect timestamps for send acceptance, delivery observations when available, link consumption, and final activation. Compare completion time and abandonment for US and EU recipient cohorts separately; record resend frequency, stale-challenge rejection, duplicate dispatch, and requests that reach the attempt limit. Choose the observation window and success threshold before testing. No numeric threshold here is a claimed provider result.

Run a late-event case too: replace a challenge, then process an observation for the old one. The account should remain pending until the current verification rule is met. That test reveals more about signup correctness than a successful send in a console.

Pick the smallest provider responsibility that meets those measurements. The durable asset is an auditable explanation of why an account became verified.

## References

- Resend documentation: https://resend.com/docs/introduction
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify documentation: https://developer.vonage.com/en/verify/overview
- Anthropic tool-use documentation, for keeping model-selected actions behind explicit tool contracts: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
