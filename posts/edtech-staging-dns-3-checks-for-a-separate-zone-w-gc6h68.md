# Edtech Staging DNS — 3 Checks for a Separate Zone Write Boundary

Customer-owned domains make a staging mail rehearsal useful, but they also put production authentication records within reach if the publishing credential spans the parent zone. Resolve that trade-off by giving staging its own delegated DNS zone and limiting its publisher to that zone. A production subdomain can be that separate zone; a different-looking hostname alone does not establish a write boundary.

TL;DR: Choose a separate staging zone when the job is to contain DNS writes. For an edtech app publishing SPF, DKIM, and DMARC, verify three things before enabling the publisher: which zone it can change, who controls the parent delegation, and whether the test message actually exercises the intended authentication domain. Customer-owned versus platform-owned zones changes who must authorize those steps. It does not remove them.

## Put the publisher behind a declared boundary

The proposed flow starts with a school authorizing a staging namespace. Its DNS administrator establishes the delegation, the app prepares authentication records inside that namespace, and a publishing worker receives credentials restricted to the child zone. After publication, a separate verification step checks DNS and a controlled test message. Production delegation and production authentication changes stay with their existing administrator. This is a proposed deployment design, not a claim that every DNS service exposes the same permission controls.

Consider an illustrative school domain, school.example, with trial.school.example reserved for staging. The school's administrator owns the parent-side change. The app publishes only into the child zone. A platform-owned alternative, school-trial.mail.example, can make that publishing workflow easier to automate, but it tests a platform namespace. Passing there does not demonstrate that mail using the school's production From domain will pass DMARC. RFC 7489 ties that judgment to identifier alignment.

Keep those tests separate.

The following TypeScript example is a runnable planning check for these two illustrative zones. It makes no DNS requests and contains no live credential. Run it with your existing TypeScript runner; its output is an approved record plan. The publisher must separately enforce the zone restriction through its actual authorization mechanism.

```ts
type Change = { owner: string; type: "TXT"; value: string };
type Plan = {
  zone: string;
  credentialZone: string;
  changes: Change[];
};

function name(value: string): string {
  return value.toLowerCase().replace(/\.$/, "");
}

function validate(plan: Plan, productionZone: string): Plan {
  const zone = name(plan.zone);
  if (zone === name(productionZone)) {
    throw new Error("Staging must not target the production zone");
  }
  if (name(plan.credentialZone) !== zone) {
    throw new Error("Credential scope must match the staging zone");
  }
  for (const change of plan.changes) {
    const owner = name(change.owner);
    if (owner !== zone && !owner.endsWith(`.${zone}`)) {
      throw new Error(`Record owner outside staging: ${owner}`);
    }
  }
  return plan;
}

const plan: Plan = {
  zone: "trial.school.example",
  credentialZone: "trial.school.example",
  changes: [{
    owner: "_dmarc.trial.school.example",
    type: "TXT",
    value: "v=DMARC1; p=none",
  }],
};

console.log(JSON.stringify(validate(plan, "school.example"), null, 2));
```

This intentionally prepares only a proposed DMARC monitoring record. SPF authorization and DKIM public-key material depend on the actual sending setup; inventing either would make the rehearsal misleading. Add those records to the same plan only after obtaining their real values. The suffix check includes a dot so that eviltrial.school.example cannot pass as a descendant of trial.school.example.

## Does a production subdomain create a separate zone?

A subdomain describes a position in the DNS namespace. A delegated child zone establishes a separate place to administer records. RFC 7489's DNS discussion distinguishes records held in the parent from records served through delegation. The choice “separate zone or production subdomain” is incomplete: trial.school.example can be both. The practical question is whether staging administration stops at that boundary.

Check 1 is the effective write scope. In the proposed deployment, the staging credential must be unable to modify school.example, its mail authentication records, or unrelated customer zones. The local check above can reject an accidental plan, but credentialZone is only a declaration. Prove the real restriction using the authorization system's supported evaluation mechanism or a controlled permission test against a designated harmless record. Never use a production SPF record as a test target.

Check 2 is parent authority. Delegating a child does not transfer control of the parent's delegation to the child publisher. Record which administrator can alter that parent-side configuration, and keep that authority out of the staging worker. If the same broad credential can rewrite both zones, the intended containment has not been established, even when the dashboard shows two zone names.

That distinction is the blast-radius decision.

Names aren't permissions.

For a solo app builder, an explicit zone allowlist adds configuration work. Accept that cost when the alternative gives a staging job permission to change school-wide mail settings. A narrower credential also makes a publishing error easier to reason about: the failure should be confined to the staging namespace. The proof must include the credential, not just the code path used on a successful run.

## Check the mail identity before calling the rehearsal complete

Check 3 is authentication alignment. DMARC evaluates the domain in the visible RFC5322.From field against authenticated SPF or DKIM identifiers. Under RFC 7489, a passing aligned result from either mechanism can satisfy DMARC.

Published records aren't proof.

For a lesson-reminder test with From set to notices@trial.school.example, inspect the actual SPF-authenticated domain and the DKIM signing domain. Record those identifiers with the test result. A message that authenticates a platform domain may still lack the alignment required for the school's From address. Conversely, relaxed alignment can permit related domains to align through their Organizational Domain; do not assume that adding a staging label guarantees an independent authentication identity. Strict alignment requires an exact domain match for the relevant mechanism.

DNS write isolation and DMARC policy inheritance solve different problems. RFC 7489 describes policy discovery at the From domain and, when applicable, at its Organizational Domain; the parent policy's sp tag can govern subdomains. Publishing an explicit policy for the test domain makes the intended rehearsal easier to inspect, but the separate DNS zone itself does not cancel DMARC's discovery rules.

The example uses p=none to request monitoring rather than a rejection policy. It does not ask receivers to send reports because it has no rua tag. If aggregate reporting is part of the test, configure an authorized destination and account for RFC 7489's authorization requirement when reports go to an external domain. Do not equate a DMARC pass with guaranteed inbox placement. The RFC explicitly leaves receivers room for local policy.

## Choose ownership around the evidence you need

A customer-owned child zone exercises the school's delegation and authentication namespace. The trade-off is coordination: someone with parent authority must authorize the setup, and the app needs a clear record of that approval. Use this path when the release decision depends on evidence about customer-domain sending. The design should let the school retain parent control while granting only the child-zone access needed for publication.

A platform-owned staging zone exercises the app's publisher with fewer customer-side dependencies. It is useful for checking plan generation, record placement, and the mail verification workflow. Its boundary is evidentiary: those checks cannot prove that a school's own delegation and From-domain alignment are correct. Treat customer-domain verification as a separate release condition.

For an app that generates lesson content with an LLM, keep this release condition deterministic. A model can draft a reminder's wording; it should not decide which zone a credential may modify. Repeated model calls add token cost without proving DNS authorization. A stored zone decision, a bounded record plan, and explicit verification results are easier to inspect and carry between implementations.

## Close the release with an operational record

Before publishing, save the intended zone, record owners, current values, and the administrator responsible for the parent. Check the worker's effective permissions, then compare the approved plan with what the publisher is about to send. Preserve the previous record values so that an authorized administrator can restore them if the new plan is wrong. Restoration is another DNS change and must use the same scope checks.

After publishing, verify the records and send a controlled lesson reminder. Keep the observed From domain, SPF domain, DKIM signing domain, and DMARC result together. If publication or verification fails, record which step failed and stop promotion until it is resolved; don't repeatedly submit the same plan without checking whether it already took effect. Bound retries as an operational policy so a failed rehearsal does not become an open-ended background bill.

The release record can be short. It needs to show who owns the namespace, what the worker can change, and what the message actually authenticated. Those three answers make a delegated staging subdomain a defensible write boundary while keeping the customer-domain delivery test honest.

## Sources

References: RFC 7489, especially Sections 3.1, 6.3, 6.6.3, and 7.1, and Appendix A, for identifier alignment, policy tags and discovery, external reporting authorization, and DNS background. The zone allowlist and release procedure above are proposed engineering controls.

- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
