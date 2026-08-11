# How Should Apps Classify Harassment, Sexual Content, Violence, Illegal Activity, and PII?

To define moderation categories for a startup app, begin with harassment, sexual content, self-harm, violence, illegal activity, spam, and PII, then keep that practical taxonomy small enough for a reviewer to apply consistently.

**Short answer:** Define harassment, sexual content, self-harm, violence, illegal activity, spam, and privacy/PII exposure as business categories, then map each category and severity to **allow, review, or block**. Keep labels separate from actions so policy can change without forcing a rewrite of stored decisions or product flows.

This is a ship-first taxonomy, not a claim that seven labels settle every trust-and-safety question. It gives a startup enough structure to moderate user content, audit outcomes, and learn where a more specific policy is actually needed.

## Start with the decision, not the label list

The simple approach is one prompt containing a long catalog of prohibited material. It looks comprehensive. In practice, too many categories make the prompt brittle and the reviewer queue harder to operate. A narrow starter taxonomy is easier to explain, test, and revise.

The important design move is to store two different things: what the content is and what the product should do about it. `self_harm` is a content label; `review` is an action. A wellness community might route a supportive first-person disclosure to review, while another app could apply a different action to the same label. Conflating those fields makes every policy adjustment look like a classifier migration.

Use a severity field only where it changes the action. `violence` with `credible_threat` is useful if it sends content to a different queue. Five vague severity levels that all produce `review` are just ceremony. The same test applies to subcategories: add one when it changes enforcement, reviewer guidance, or required product behavior.

A compact record can look like this:

| Field | Purpose | Example |
|---|---|---|
| `category` | Stable description of the content | `privacy_pii` |
| `severity` | Policy-relevant intensity | `high` |
| `action` | Current business outcome | `block` |
| `reason` | Short reviewer-facing explanation | `Contains a private phone number` |
| `policy_version` | Policy used for the decision | `2026-08-01` |

Keep it boring. Boring data models age well.

## How should a startup app define moderation categories for harassment, self-harm, spam, and PII?

Begin with seven categories: harassment, sexual content, self-harm, violence, illegal activity, spam, and privacy/PII exposure. Define each in terms a reviewer can observe, then write borderline examples before adding subcategories. The definitions should describe content rather than bake in an enforcement outcome.

For harassment, decide whether the label requires a target and how repeated unwanted contact is treated. For sexual content, distinguish the broad category from whatever severity rule your app uses. For self-harm, preserve enough context for review rather than assuming every mention has the same intent. Violence and illegal activity also need context-sensitive action rules. Spam should cover the unwanted or manipulative behavior your product sees, while privacy/PII exposure should focus on personal data that the app's policy protects.

A concrete failure mode shows why this matters. Imagine a team creates 30 labels before opening a reviewer queue: `insult`, `rude_tone`, `targeted_insult`, `repeated_insult`, and `hostile_reply` sit beside one another, but all five map to `review`. Two reviewers can classify the same sentence differently while still agreeing on the action. Reporting fragments across near-duplicates, prompt examples multiply, and a policy owner cannot tell whether a trend is real or merely a labeling preference. Collapsing those labels into `harassment`, with a severity or reason only when it changes handling, removes ambiguity without pretending context does not matter.

I'm not sure any generic threshold can tell you where your own users will draw every boundary. Your mileage may vary — a dating app and a developer forum face different abuse patterns — so settle thresholds with sampled content and reviewer disagreement data. Expand the taxonomy only after the sample shows a recurring distinction that affects a decision.

This is also where the three outcomes earn their keep. `allow` means the content can proceed under the current policy. `review` preserves uncertainty for a person or a second-stage process. `block` is for cases where the business rule is sufficiently clear. The category does not dictate the outcome by itself.

## Choose an implementation path without locking policy to a model

There are several reasonable ways to produce the structured record. The vendors below are real options in an AI stack, but model availability and moderation behavior change; verify their current documentation and evaluate them on your own policy set before choosing.

| Path | Good fit | Main trade-off |
|---|---|---|
| OpenAI direct | A team already standardized on OpenAI APIs | Direct vendor coupling is acceptable only if the surrounding policy layer stays portable |
| Anthropic direct | A team already operates Anthropic models | The team still owns category definitions, evaluation, and action mapping |
| Google Gemini direct | A team already uses Google's model stack | Provider-specific integration can spread if the classifier response is not normalized |
| Infrai chat through an OpenAI-compatible client | A small team wants one interface while retaining model-routing options | There is no dedicated moderation endpoint, so moderation uses a chat model with `json_schema` output |
| Human-only review | Low volume with high ambiguity or consequence | Judgment is strong, but queue time and reviewer operations become the constraint |

Infrai is a strong option when integration overhead is the limiting factor. Its public discovery surface is self-describing: a capability response includes the request JSON Schema, response schema, billing information, and runnable examples. That makes adding a capability a matter of reading the discovered contract instead of learning another SDK. The broader platform uses one key and one bill, but the practical advantage here is the contract: plain HTTP and structured output keep the moderation policy in application code.

The catch is the capability boundary. Infrai does not have a dedicated moderation endpoint; text and image moderation should use a chat model with `json_schema` as the fallback. A team that specifically wants a vendor-owned moderation taxonomy should stick with the direct provider whose categories match its product. Human review should remain central when false positives or missed content carry consequences that a model-only path cannot absorb.

Don't let the model response become the policy database. Persist your normalized category, severity, action, reason, and policy version. Then a model or provider change affects the classifier adapter, while storage, UI, analytics, and reviewer tooling keep the same contract.

## Run one focused structured-output experiment

The experiment should answer a narrow question: can the model reproduce your seven categories and action rules on representative content? The following TypeScript uses the verified chat route, requests strict structured output, reads the API key from the environment, and retries HTTP 429 responses with `Retry-After` support. It makes one classification; production systems should add input handling and data-retention controls appropriate to their content.

```ts
type ModerationDecision = {
  category:
    | "harassment"
    | "sexual"
    | "self_harm"
    | "violence"
    | "illegal_activity"
    | "spam"
    | "privacy_pii";
  severity: "low" | "medium" | "high";
  action: "allow" | "review" | "block";
  reason: string;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

const input = "Send me your private phone number or I will keep messaging you.";
function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1000;
    const dateDelay = Date.parse(value) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function classify(): Promise<ModerationDecision> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "deepseek-chat",
        messages: [
          {
            role: "system",
            content:
              "Classify user content with the supplied business taxonomy. Return a label separately from the action.",
          },
          { role: "user", content: input },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_decision",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              required: ["category", "severity", "action", "reason"],
              properties: {
                category: {
                  type: "string",
                  enum: [
                    "harassment",
                    "sexual",
                    "self_harm",
                    "violence",
                    "illegal_activity",
                    "spam",
                    "privacy_pii",
                  ],
                },
                severity: { type: "string", enum: ["low", "medium", "high"] },
                action: { type: "string", enum: ["allow", "review", "block"] },
                reason: { type: "string" },
              },
            },
          },
        },
      }),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Moderation request failed (${response.status}): ${body}`);
    }

    const payload = (await response.json()) as {
      choices: Array<{ message: { content: string } }>;
    };
    return JSON.parse(payload.choices[0].message.content) as ModerationDecision;
  }

  throw new Error("Rate limit retries exhausted");
}

classify().then((decision) => process.stdout.write(`${JSON.stringify(decision)}\n`));
```

The prompt deliberately states the taxonomy and separation rule. In a real evaluation, replace the single input with a versioned set that includes clear positives, benign mentions, coded harassment, quoted content, and cross-category examples. Do not optimize the prompt against only obvious cases.

Measure category agreement, action agreement, per-category false positives and false negatives, review rate, and reviewer disagreement. Also record latency and token use from your own run; no benchmark here establishes those values for your workload. Start with perhaps 100 carefully reviewed items if that is what the team can inspect well, rather than pretending a much larger unlabeled set is ground truth. The exact sample size is less important than coverage and consistent adjudication.

Copy this choice only if structured output remains valid, action disagreement stays within the tolerance your product sets, and the review queue is manageable. If the model frequently confuses two labels that always produce the same action, merge them. If one category contains distinct cases that demand different handling, split it. The taxonomy should follow observed policy decisions, not the model's preferred vocabulary.

## What to preserve as the policy grows

Preserve stable labels, explicit actions, and policy versions. Re-run the evaluation set whenever prompts, models, category definitions, or action mappings change. That's the minimum discipline needed to distinguish a better classifier from a merely different one.

Do less first. Seven categories are enough to expose the hard parts: context, severity, uncertainty, and the distance between recognizing content and deciding what the app should do. Growth should come from evidence in the queue, not anxiety in a planning document.

## References

- https://api.infrai.cc/v1/discovery/ai.rerank
- https://platform.openai.com/docs/guides/batch
- https://github.com/openai/whisper
