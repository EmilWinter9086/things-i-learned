# Pricing Rule Analytics: 3 Checks Linking Metrics Dashboard Signals With Logs Search

A new fintech pricing rule behind a flag needs an evaluation constraint before it needs another chart: the exposed and unexposed cohorts must describe comparable pricing decisions. TL;DR: use a metrics dashboard for the rollout decision, then logs search to explain a specific discrepancy. For a Node.js SaaS admin backend, start with one explicit decision record and two read paths. Check cohort assignment, denominator completeness, and diagnostic coverage before treating a quiet dashboard as permission to expand exposure.

This note proposes a design, not a measured rollout result. The intended result is a dashboard whose signal survives investigation: an operator can explain which rule ran, which decisions counted, and what evidence remains for an unexpected outcome. The evaluation constraint is that sampled diagnostic evidence cannot stand in for complete business accounting.

## How should admin analytics connect a metrics dashboard with logs search?

Consider an illustrative pricing flag with two bounded variants, control and candidate. An admin page displays the share of pricing decisions that fail validation. A tempting first implementation searches logs for validation failures and charts the count. That answers how many matching records the search returned. It leaves the operator to establish which decisions were eligible, which variant actually ran, and whether the search covered all relevant records. A smaller count alone cannot establish that the candidate rule behaved better.

Keep the denominator.

The chosen design records the executed rule version and an explicit outcome for each logical pricing decision before deriving the dashboard view. Compare invalid decisions with all completed decisions within the same variant and evaluation window. Also track decisions that started but have no terminal outcome; otherwise an interrupted calculation can disappear from a completion-based denominator. These are proposed measurement contracts. Their value depends on enforcing them at the point where the application decides what happened, including its retry and error paths.

There is a second trap: flag assignment and executed behavior can diverge in the application's own control flow. For example, a proposed rule might apply only after an eligibility check. Grouping by the assigned variant alone would mix eligible candidate executions with decisions that still used the control rule. Preserve both assignment and execution in the decision record, and define the chart around execution. Keep eligibility visible as a separate count. That makes a rollout question answerable without pretending that every assigned request received the new calculation. More dimensions aren't automatically more signal. For this design, keep dashboard groupings bounded to variant, rule version, and outcome. Put an individual decision identifier in the diagnostic record. The trade-off is deliberate: fewer aggregate groupings make the rollout comparison easier to inspect, while the identifier preserves a route to one calculation when the aggregate needs explanation. Keep both the included count and the excluded eligibility count visible beside the comparison; hiding either forces the operator to reconstruct the cohort during an investigation.

## A focused decision record for the admin backend

The following TypeScript example defines the boundary after a pricing decision has finished. The records are synthetic fixtures, not production observations. The summary is an in-memory illustration of the aggregate read path; it is not a persistence adapter or a complete feature-flag implementation. Both read paths consume the same completed-decision contract.

```ts
type Decision = {
  decisionId: string;
  assignedVariant: "control" | "candidate";
  executedVariant: "control" | "candidate";
  ruleVersion: string;
  outcome: "valid" | "invalid";
};

const decisions: Decision[] = [
  {
    decisionId: "example-a",
    assignedVariant: "candidate",
    executedVariant: "candidate",
    ruleVersion: "candidate-rule",
    outcome: "invalid",
  },
  {
    decisionId: "example-b",
    assignedVariant: "candidate",
    executedVariant: "control",
    ruleVersion: "control-rule",
    outcome: "valid",
  },
];

function summarize(records: readonly Decision[]) {
  const groups = new Map<string, { completed: number; invalid: number }>();
  const seen = new Set<string>();

  for (const record of records) {
    if (seen.has(record.decisionId)) {
      throw new Error("Duplicate logical pricing decision");
    }
    seen.add(record.decisionId);

    const key = JSON.stringify([record.executedVariant, record.ruleVersion]);
    const group = groups.get(key) ?? { completed: 0, invalid: 0 };
    group.completed += 1;
    group.invalid += Number(record.outcome === "invalid");
    groups.set(key, group);
  }

  return Array.from(groups, ([cohort, counts]) => ({
    cohort,
    ...counts,
    invalidShare: counts.invalid / counts.completed,
  }));
}

const dashboardRows = summarize(decisions);
const diagnosticRecord = decisions.find(
  (record) => record.decisionId === "example-a",
);
```

The second fixture is intentional. It was assigned to the candidate but executed the control rule, so it belongs in the control execution group. The example also rejects a repeated logical decision identifier. In a deployed system, define that identifier's scope and enforce the corresponding uniqueness policy in durable storage; this local set protects only one invocation. Decide separately how retries and legitimate recalculations receive identifiers. Silently counting each retry would change the denominator.

Do not interpret either one-record cohort as evidence that a rule is ready. The fixture contains 2 decisions and 1 invalid outcome; these are test inputs, not measured rollout results. There is no time window in this function, either: its caller must supply the intended window. A real admin page should display that window and the latest included event time alongside the counts, so an operator can distinguish recent evidence from an old summary.

This is the smallest useful backend boundary for the proposed choice. Persistence, aggregation, and search can change behind it. Keep the decision vocabulary application-owned so changing an observability backend doesn't also redefine what the rollout means.

## Sampling changes what the evidence can prove

OpenTelemetry's sampling concepts distinguish head sampling, where the sampling decision happens before the whole trace is available, from tail sampling, where the decision can use information from the completed trace. That distinction matters when a pricing discrepancy becomes apparent late in a request. A head-sampling decision cannot inspect an outcome that has not happened yet. Tail sampling can consider that later information, with additional processing and buffering considerations described in the sampling guidance.

Use that distinction to set expectations for investigation. A sampled trace can explain a retained execution. Its absence does not establish that a pricing decision never happened. Trace sampling also does not, by itself, specify how an application's separate logs or business records are retained. Define those policies explicitly instead of assuming one telemetry setting governs everything.

For this design, preserve the complete decision denominator independently of sampled diagnostic traces. Then choose diagnostic retention around the questions operators must answer, including invalid outcomes and eligibility mismatches. This spends storage and processing on explanatory evidence while keeping the aggregate contract stable. It also introduces a reconciliation obligation: the completed decisions included in an aggregate must agree with the corresponding business records for the same window.

An LLM-generated explanation in the admin interface should inherit the same boundary. Let it summarize the counts and the records actually retrieved. Require it to distinguish missing diagnostic evidence from a successful pricing decision. A token budget can limit how much evidence the explanation reads; it should not silently redefine the cohort used to make the rollout decision.

## Measure these 3 checks before expanding exposure

First, test attribution. Feed synthetic cases through both variants, an ineligible assignment, a retry, and an interrupted calculation. Verify that assignment and execution remain distinguishable, and that a retry follows the chosen logical-decision policy. The interrupted case should remain visible outside the completed denominator until its status is resolved.

Second, reconcile completeness. For a declared window, compare the aggregate's completed and invalid counts with the underlying decision records. Measure arrival delay as well. A correct count that arrives after the operator makes the expansion decision has limited operational value. Set an acceptable delay and a minimum amount of evidence before the experiment begins; this note supplies neither a universal threshold nor a measured one.

Third, test investigation coverage. Select known synthetic invalid decisions and confirm that an operator can retrieve the intended explanation using the decision identifier. Repeat under the proposed trace-sampling and retention policies. Record which evidence survives and how long retrieval takes. This exposes the trade-off between diagnostic volume and the ability to explain a rare outcome before a real rollout depends on it.

Start narrow.

For this fintech rollout, the easiest useful backend choice is a metrics view for the bounded expansion question, backed by searchable decision details for investigation. If building both read paths delays shipping, persist the decision contract first and derive a small aggregate from it. Measure query time, data delay, retained volume, and diagnostic coverage before adding another service. Expand exposure only when the three checks support the decision; a clean chart alone is insufficient evidence.

## References

OpenTelemetry's sampling concepts support the head-versus-tail sampling distinction and its trade-offs. The decision schema, synthetic fixtures, and rollout checks above are proposed application design choices, not benchmark findings.

## Sources

- OpenTelemetry, Sampling: https://opentelemetry.io/docs/concepts/sampling/
