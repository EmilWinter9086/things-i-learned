# Node.js Speech-to-Text API: Empty Transcript, Null Text, and Malformed JSON Validation

Short answer: treat a speech-to-text response as untrusted input, even after HTTP 200. Check the content type, parse JSON in a narrow boundary, validate the shape, and make empty or null text an explicit application state instead of silently turning it into a successful transcript.

The data flow is small: an upload worker sends audio, receives bytes, and maps those bytes into a typed result. The dangerous assumption is that a successful transport means a usable transcript. A provider can return an empty string for silence, null for an absent field, HTML from a proxy, or JSON that is truncated in transit. Those outcomes need different operational decisions.

The status code is only one signal.

## What should a Node.js client do with empty speech-to-text text and malformed JSON?

Put one defensive boundary around the response. Keep provider-specific decoding there, and let the rest of the application consume a small internal union. This TypeScript example distinguishes an empty transcript from a protocol failure without pretending that a cast is validation.

```ts
type TranscriptResult =
  | { kind: "text"; text: string }
  | { kind: "empty"; reason: "blank" | "null" }
  | { kind: "error"; reason: "http" | "content-type" | "malformed-json" | "schema"; detail?: string };

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

export async function readTranscript(response: Response): Promise<TranscriptResult> {
  const raw = await response.text();
  if (!response.ok) return { kind: "error", reason: "http", detail: `status=${response.status}` };
  const contentType = response.headers.get("content-type")?.toLowerCase() ?? "";
  if (!contentType.includes("application/json")) return { kind: "error", reason: "content-type", detail: contentType || "missing" };
  let payload: unknown;
  try { payload = JSON.parse(raw); } catch { return { kind: "error", reason: "malformed-json" }; }
  if (!isRecord(payload) || !("text" in payload)) return { kind: "error", reason: "schema", detail: "missing text field" };
  const value = payload.text;
  if (value === null) return { kind: "empty", reason: "null" };
  if (typeof value !== "string") return { kind: "error", reason: "schema", detail: "text is not a string" };
  if (value.trim() === "") return { kind: "empty", reason: "blank" };
  return { kind: "text", text: value };
}
```

There is a practical reason to read the body once with `response.text()`: a later call to `response.json()` cannot recover a body that was already consumed, and it gives logging code a controlled place to redact or cap raw bytes. Never log the audio or an unrestricted response body just to diagnose parsing.

## How do validation, retries, and observability change the transcript pipeline?

Empty text is usually a business result, not a transient network error. Silence, an unintelligible clip, or a language mismatch may all produce no words. Record the reason and audio metadata, then decide whether the product should show “no speech detected,” request another clip, or accept an empty result. Retrying every empty response wastes latency and can multiply usage.

Malformed JSON is different. A bounded retry can make sense for a retryable transport or gateway failure, but it should be limited, jittered, and keyed by an idempotency token when the upstream supports one. A parser exception alone is not evidence that repeating the same request will help. Keep the original status, content type, request identifier, byte count, and parser reason in structured telemetry. Redact transcript text unless the data policy explicitly allows it.

I keep a three-word log message for the common path: “transcript empty.” Then I attach the machine-readable reason. Tiny signals are easier to aggregate than a dashboard full of stack traces.

Contract tests should feed the client fixtures for a normal string, `text: ""`, `text: "   "`, `text: null`, a missing field, a number, invalid JSON, a non-JSON content type, and a non-2xx status. For each fixture, assert the union kind, the retry decision, and the telemetry reason; then run the same matrix through the queue worker so a parser result cannot be accidentally converted into a completed database row. Include a fixture whose body is HTML with a 200 status, one whose JSON is valid but has an array where an object is expected, and one whose `text` contains only Unicode whitespace. Assert the union kind, not an incidental error-message string. The test suite should also verify that a response body is consumed once and that sensitive payloads never reach logs.

## Which transcript contract is safe to expose to downstream code?

Do not pass `unknown` or a provider-shaped object through every layer. Normalize it at the edge into the union above, persist the state transition, and make downstream code handle all three branches. A search index can skip `empty`; billing can count only `text` after a policy check; an alert can page on repeated `malformed-json` but not on ordinary silence.

The catch is that this client cannot infer semantic quality from a non-empty string. “okay” might be a valid utterance, a hallucinated segment, or a low-confidence result. If the application needs confidence thresholds, timestamps, language detection, or speaker labels, extend the schema contract and validate those fields explicitly. For a privacy-sensitive or regulated workflow, a managed transcription API may be unsuitable unless its retention, residency, and deletion guarantees match the requirement; a self-hosted model or a provider with documented controls is the better fit.

Your mileage may vary: response envelopes differ, and some APIs use a different field name or return plain text by design. That is a contract difference, not a reason to weaken validation. Adapt the adapter, keep the internal result stable, and pin fixtures to the documented contract.

A ship checklist is short prose: verify status before decoding, require the expected media type, parse once, validate every field used, classify blank and null deliberately, cap retries, redact logs, and test the ugly payloads. That discipline keeps a green HTTP status from becoming a false product success.

## References

- https://owasp.org/www-project-top-10-for-large-language-model-applications/
- https://developer.mozilla.org/en-US/docs/Web/API/Response/json
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- https://github.com/pgvector/pgvector
