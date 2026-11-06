# Node.js Leak Rehearsal: Containing Prepaid API Balance Auto Recharge and Manual Funding

A leaked support API credential should lose permission to spend before a low prepaid balance can trigger auto recharge. That is the acceptance condition for this engineering note: the drill passes only when the compromised credential cannot consume newly funded credit, including manual top-ups during recovery. The example below is a proposed control and a synthetic exercise, with no measured incident or benchmark claimed.

Short answer: use an auto-recharge threshold to decide when funding may be needed, a shared daily funding ceiling to limit how much may be added, and a separate credential admission budget to contain consumption. Manual top-ups belong under the same funding ceiling. For a small SaaS support feature, choose manual funding until the drill demonstrates that revocation, pending funding, and concurrent requests respect those boundaries. Automation is a later operational choice.

The cheap implementation is tempting: check the account balance, add credit below a threshold, and alert on unusual usage. Its failure in this threat model is structural. A stolen credential can keep consuming while the funding loop keeps restoring its fuel. Switching that loop to manual approval introduces a human checkpoint, but a person can still refill an account the attacker can access. The experiment therefore evaluates containment after the refill, rather than treating a successful top-up as recovery.

## What can one leaked credential actually reach?

Start with the customer support path: incoming ticket, queued summarization job, model request, saved draft reply. Give that path a dedicated runtime identity. The funding worker has a different identity, and neither credential belongs in a ticket attachment, model prompt, or diagnostic event. This is a proposed application boundary, grounded in the OWASP guidance on least privilege, secret lifecycle management, rotation, and revocation [1]. It doesn't assume that an account provider offers every desired scope.

Trace the bypasses.

If the exposed secret is an upstream account credential that can call the service directly, an application gateway budget cannot stop requests made outside that gateway. The drill must revoke the upstream secret and verify the resulting denial. If the exposed secret is an internal support token accepted only by a controlled gateway, that gateway can enforce the credential budget. These are different containment claims. A passing test for one says nothing about the other.

The central trade-off is shared credit versus isolated authority. A shared account makes funding simpler, but its balance may be reachable by several runtime identities. Separate accounting labels help investigate consumption; they should not be mistaken for enforced isolation. Before relying on a scope, test the operation that should be denied. In this design, the support identity cannot change recharge settings or submit funding requests, even when its runtime budget has room.

OWASP recommends handling secrets through their lifecycle and auditing their use [1]. For this drill, record the credential identifier, support queue, authorization decision, funding operation identifier, and incident cutoff time. Keep the secret value out of those events. Ticket text is unnecessary for proving that a revoked credential was denied, so the exercise can use synthetic tickets.

## Can a prepaid API balance trigger auto recharge after credential revocation?

Three controls need separate names. The trigger threshold is a balance level that makes the funding controller consider a top-up. The daily ceiling is an application policy limiting additions during an explicitly defined accounting day. The credential budget is a limit on work admitted for a particular runtime identity. None can stand in for the other two.

Here is the synthetic fixture. The support account starts with 120 credit units, the refill threshold is 40 units, each refill requests 80 units, and the shared daily funding ceiling is 160 units. These are exercise inputs, not prices, service defaults, or measurements. The selected numbers make it possible to test two allowed additions and a rejected third attempt without introducing a pricing comparison. Count the opening 120 units separately: a daily ceiling on new funding does not erase an existing balance.

Both manual and automatic additions debit the same allowance. If an automatic refill of 80 units has settled and a manual refill of 80 units is pending, the full 160-unit allowance is already committed. A second worker must not see only the settled amount and approve another refill. Reserve capacity before dispatching the funding operation. A timeout leaves that reservation pending until reconciliation establishes an outcome; it doesn't establish that no credit was added.

That reservation makes availability less convenient. An uncertain payment can block a later refill, and a legitimate support queue may have to wait while the outcome is checked. Releasing capacity optimistically makes the daily ceiling a promise the controller cannot enforce under concurrency. For this design, preserving the ceiling wins. Keep unanswered tickets queued for the existing human support workflow.

The threshold has a different job.

Estimate the balance needed while detection, funding, and settlement take place, then add a margin justified by the workload. Before choosing that estimate, measure the support queue's consumption during those intervals. Don't copy the fixture's 40-unit threshold into production. A higher trigger can give funding more time; it does not reduce what a stolen credential is authorized to consume.

## A focused admission gate for the support worker

The following TypeScript function demonstrates the credential side of the boundary. It models an atomic state transition with integer budget units. The caller supplies an upper-bound reservation for one support operation; the example does not estimate tokens, settle charges, or call a funding service. Production admission must serialize this transition with revocation and persist its result before dispatching work.

```ts
type CredentialState = {
  status: "active" | "revoked";
  incidentFrozen: boolean;
  limit: number;
  spent: number;
  reserved: number;
};

type Admission =
  | { allowed: true; next: CredentialState }
  | { allowed: false; reason: string };

function admitSupportWork(
  state: CredentialState,
  upperBound: number,
): Admission {
  const values = [
    state.limit, state.spent, state.reserved, upperBound,
  ];
  if (!values.every((value) =>
    Number.isSafeInteger(value) && value >= 0
  ) || upperBound === 0) {
    return { allowed: false, reason: "invalid-budget" };
  }

  if (state.status !== "active" || state.incidentFrozen) {
    return { allowed: false, reason: "credential-blocked" };
  }

  const available = state.limit - state.spent - state.reserved;
  if (upperBound > available) {
    return { allowed: false, reason: "budget-exhausted" };
  }

  return {
    allowed: true,
    next: { ...state, reserved: state.reserved + upperBound },
  };
}

const supportDraftCredential: CredentialState = {
  status: "active",
  incidentFrozen: false,
  limit: 60,
  spent: 42,
  reserved: 10,
};

const withinBudget = admitSupportWork(supportDraftCredential, 8);
const overBudget = admitSupportWork(supportDraftCredential, 9);
const afterLeak = admitSupportWork(
  { ...supportDraftCredential, status: "revoked" },
  1,
);
```

The fixture admits the 8-unit request, rejects the 9-unit request, and rejects the 1-unit request after revocation (the limit and existing reservations stay constant). Refilling the account does not alter any of these decisions because account balance is deliberately absent from this function. A separate check must establish that funding exists before dispatch; authorization and affordability must both pass.

The difficult part is around the function. Two workers reading the same state must not each reserve the last 8 units. Persist admission with a unique operation identifier inside the serialization boundary, and return the stored decision when that operation is retried. Revocation uses that same boundary. Otherwise, a worker can authorize against a stale active state after the incident controller believes the credential is blocked.

There is also a condition the code cannot supply: the reservation must bound the operation's actual liability. Enforce the applicable request limits and account for retries when constructing that bound. If the upstream service cannot offer an enforceable bound, don't describe this local budget as a hard financial cap. Reconcile completed work against reservations, and keep uncertain work reserved until its outcome is known.

## Run the leak through funding and recovery

Use a synthetic ticket asking for a refund-status summary. Capture the starting credential state and funding ledger, admit one job, then expose only the exercise credential to the test client. The goal is to demonstrate the boundary with controlled requests. Mark an incident cutoff and revoke that identity. OWASP's lifecycle guidance makes revocation and rotation part of secret management; this exercise turns them into observable acceptance conditions [1].

Continue the drill across these transitions:

1. Submit work using the revoked credential through each applicable entry point. Record denials without recording the secret. An upstream credential must also be tested directly against its upstream boundary.
2. Let a previously authorized funding operation complete, then attempt a manual top-up within the remaining daily allowance. Repeat the revoked-credential request after each event. Newly available credit must not restore its access.
3. Race two funding requests against the last available allowance and retry one uncertain operation. Check that pending reservations prevent excess authorization and that a retry cannot create a second logical addition.
4. Cross the defined day boundary with an operation still pending. Preserve its reservation under its original accounting day and reconcile it there, so a reset cannot silently discard an unresolved commitment.
5. Provision a replacement runtime identity through the normal secret distribution path. Confirm legitimate support work can resume and that the old credential remains denied.

A denial after revocation doesn't cancel work admitted earlier.

Capture those operations at the cutoff, reconcile their eventual usage, and report the residual exposure separately. This matters for support queues: replaying every unfinished job after rotating a key can authorize duplicate work unless the application preserves job identity and completion state. Recovery needs both a usable replacement secret and a trustworthy queue.

For a solo builder, this is where manual funding can earn its place. It can make the funding pause visible while the runtime boundary is still being established. The trade-off is delayed support work and operator interruption. Once the same containment checks pass with automatic funding, manual approval should be chosen for an explicit operational reason, rather than assumed to be a substitute for revocation.

## Measure the boundary before enabling automatic funding

Collect four observations from the exercise: time from incident cutoff to the last successful authorization using the old credential, liability already admitted at that cutoff, total settled plus pending additions for each accounting day, and time until legitimate support work resumes. Compare the ledger with the funding system's authoritative outcomes. A dashboard showing a low balance cannot prove any of these properties.

Keep latency and token consumption attached to support job identifiers so retries and duplicate drafts are visible. Otherwise, a successful credential rotation can conceal a recovery loop that keeps reprocessing the same ticket. This is also the practical vendor boundary: the application owns its queue, identities, and reservations, while the upstream service's documented revocation and accounting behavior determines which containment claims can be verified.

The decision rule is narrow. Enable auto-recharge only when revoked credentials stay denied after pending and manual funding, funding reservations survive concurrency and day rollover, and legitimate support work has a tested recovery path. Choose the trigger threshold from measured consumption during the funding delay. Keep manual and automatic additions under one daily ceiling, and maintain a separate credential budget. The useful result is a demonstrated limit on one leaked secret's authority.

## References

[1] OWASP, Secrets Management Cheat Sheet. Primary guidance for least privilege, secret lifecycle management, rotation, revocation, and auditing. The fixture values and admission design above are illustrative application policies, not requirements imposed by this source.

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

## Further reading

The References source above provides the secret lifecycle guidance needed to adapt this drill to a deployment:

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
