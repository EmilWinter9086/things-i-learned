# Receipt Support Handoffs — SMS Alert API Integration and Delivery Evidence

Choose an SMS alert API by testing the support handoff for a settled payment: a receipt must have a traceable outcome, and a scheduled copy must have a clear cancellation outcome. The deciding constraint is integration effort across the whole receipt workflow, including the work needed to answer a customer who says the receipt never arrived.

Short answer: for a SaaS serving US and EU customers, evaluate delivery status polling and scheduled SMS cancellation with one receipt scenario before committing to an API. In a Node.js application, keep those operations behind a small TypeScript contract. Choose on the evidence your support workflow needs, rather than the number of lines in the first send example.

This experiment note proposes an acceptance exercise, not a measured winner. Its constraint is deliberately narrow: send an order receipt after payment settles, then make the result understandable to customer support. No provider performance or country coverage is assumed.

## What should SaaS support require from an SMS alert API?

Start with a synthetic case: order `order_125`, payment `payment_125`, receipt revision `1`. These are test identifiers, not customer records. Give the case a settlement event, a planned dispatch, a status lookup, and a cancellation request. The exercise should end with enough evidence to decide whether the receipt is waiting, dispatched, canceled, or unresolved. It should also preserve uncertainty when the external response doesn't settle the question.

Keep the receipt boring.

The proposed application rule is that settlement authorizes dispatch; a request to pay does not. Have the payment integration produce the settlement input according to its documented contract. The SMS adapter should consume that decision, not infer it from a customer message or an LLM-generated summary. If an AI support feature writes the explanation, give it the recorded outcome and require it to preserve unresolved states. Generating another explanation should never generate another receipt. For example, give the test support view two requests concerning `order_125`: the first asks where the receipt is, while the second asks to stop its scheduled copy after the customer has obtained the order details elsewhere. Both requests should point to receipt revision `1`, and neither should authorize dispatch just because the assistant has produced a helpful response. This is a proposed fixture, not an observed customer incident.

Keep those permissions separate.

For a US/EU service, repeat the exercise for each intended destination market. Ask for evidence of the relevant sender requirements, status meanings, scheduling behavior, and cancellation boundary. A successful test for one destination is not the acceptance criterion for another. This is an evaluation requirement; it makes no claim that any particular API supports either market.

## Replace the send-only experiment with a receipt case

A minimal experiment sends a message and saves the returned identifier. That is enough to inspect the submission response. It is insufficient for the support question in this exercise: what should happen when the customer asks for a copy while the original receipt is still scheduled? The missing part is the application's decision record.

Use a receipt case that owns the business identifiers and links them to the external message identifier. Record status observations with their observation time and original status value. Keep a separate cancellation outcome. Under this proposed design, asking to cancel does not itself make the receipt canceled; only evidence satisfying the adapter's documented cancellation contract can do that. An ambiguous response leaves an unresolved case for reconciliation.

That costs implementation time. The benefit is specific: support can inspect the pending receipt before deciding whether to create a new one. A send-only integration remains an acceptable prototype if its evaluation ends at submission, but it hasn't yet answered this receipt workflow's selection question.

Make the trade-off explicit in the acceptance exercise. One adapter might require an application scheduler; another might expose a scheduling operation whose cancellation semantics still need validation. Count the state management, deployment work, and support procedures each approach requires. Don't assign either capability to an unnamed service without documentation.

## A focused contract for the support view

The following TypeScript example is an application-owned interface, not an external API schema. It models one support inspection: read the receipt's current observation, optionally attempt cancellation, and return the evidence. Its literal states are local decisions; an actual adapter must map documented external responses without guessing.

```ts
type ReceiptCase = {
  orderId: string;
  paymentId: string;
  revision: number;
  externalMessageId: string;
};

type Observation = {
  observedAt: string;
  rawStatus: string;
  phase: "pending" | "terminal" | "unknown";
};

type Cancellation =
  | { outcome: "confirmed"; evidence: string }
  | { outcome: "rejected"; evidence: string }
  | { outcome: "unknown"; evidence: string };

interface ReceiptTransport {
  inspect(externalMessageId: string): Promise<Observation>;
  cancel(externalMessageId: string): Promise<Cancellation>;
}

async function inspectForSupport(
  receipt: ReceiptCase,
  transport: ReceiptTransport,
  requestCancellation: boolean,
): Promise<{
  receipt: ReceiptCase;
  observation: Observation;
  cancellation: Cancellation | null;
}> {
  const observation = await transport.inspect(receipt.externalMessageId);
  let cancellation: Cancellation | null = null;

  if (requestCancellation && observation.phase === "pending") {
    cancellation = await transport.cancel(receipt.externalMessageId);
  }

  return { receipt, observation, cancellation };
}
```

The trap is the gap between inspection and cancellation. The observation is a snapshot, so this function returns the cancellation result separately instead of overwriting the observation with an optimistic success. In the acceptance exercise, deliberately change the transport response between those calls. A support view that still announces successful cancellation fails the proposed contract. Make the assertion about the displayed outcome and saved evidence, because an assertion that merely checks whether the cancel function ran would allow an incorrect customer explanation to pass.

Check the explanation too.

This example does not implement dispatch, persistence, retries, or exception handling. Put those responsibilities in the surrounding application and test them explicitly. A rejected inspection promise should produce a recorded lookup failure and an unresolved support view; it should not trigger another send. Persist cancellation evidence before telling the customer what happened.

Keep the LLM outside this control path. This design needs no model call to interpret the three local cancellation outcomes, so a token budget is unnecessary for that decision. If generated explanations are useful, measure their latency and token use separately from receipt dispatch. The trade-off is less flexible automated wording in exchange for a smaller path to test.

## Measure the work before copying the choice

Run the receipt case through these proposed test conditions. They are test inputs, not claims about a provider's observed behavior.

| Test input | Required application behavior |
| --- | --- |
| Ordinary status observation | Save the observation and its original status value. |
| Unrecognized external status | Preserve uncertainty rather than infer success. |
| Confirmed cancellation | Save the confirming evidence before displaying confirmation. |
| Uncertain cancellation response | Leave the cancellation outcome unresolved. |
| State changes between inspection and cancellation | Use the cancellation response to evaluate the cancellation outcome. |

Retain the original responses.

Measure integration effort from settlement input to a support-readable outcome. Count adapter code, persistence changes, test setup, and manual decisions left for support. Also record the time until a useful status observation, the number of lookups per receipt, and unresolved cases at the end of the test. Choose a polling budget and stopping rule before the run; evaluate them against the selected API's documented limits and status contract. There is no justified universal polling interval in this exercise.

Test deployment too. Use synthetic receipts, keep message credentials outside the support view, and verify that restarting the application preserves pending receipt cases. Exercise a disabled-dispatch mode before enabling live delivery. If a status mapping changes, make its effect on unresolved cases visible. These are proposed acceptance checks whose implementation effort belongs in the comparison.

Two standards help define the edges of the problem. RFC 7489 specifies DMARC for email authentication policy and reporting; it does not define SMS delivery or scheduled cancellation. If this receipt workflow adds an email copy, evaluate that channel separately. NIST SP 800-63B discusses authentication, including restrictions on PSTN use for out-of-band authentication. A transactional order receipt is a different job: don't treat its delivery observation as proof that a customer has authenticated. Neither source establishes SMS provider coverage or cancellation behavior. Those claims require the candidate's own documentation and test evidence.

Finish the selection with the support case open beside the implementation estimate. Prefer the integration whose documented behavior and test evidence satisfy the required handoff with manageable application work. Until that evidence exists, a short send example is only a starting point.

## References

- RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC). https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B: Digital Identity Guidelines, Authentication and Lifecycle Management. https://pages.nist.gov/800-63-3/sp800-63b.html
