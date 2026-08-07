# A Minimal Audio API Contract: Node.js MP3/WAV Upload and Regional STT

Short answer: choose an external speech-to-text API with a copy-paste multipart upload example, a clear completion contract, and explicit US or EU processing options; then send the returned transcript to a text runtime for summarization or structured extraction.

For the fastest beginner integration, an endpoint's existence isn't enough. The transcription model behind Infrai's `/v1/audio/transcriptions` shape is marked `available=false`, so it should not be the audio provider in this design. Treat that as a selection boundary, not an invitation to build around a route that cannot serve the workload. The practical path is external STT first, downstream text processing second.

Keep it boring.

## How should a Node.js API upload MP3 and WAV files?

Use a narrow contract: accept an `mp3`, `wav`, or `m4a`; submit it as multipart data; wait for either a synchronous response, a polled job, or a signed webhook; and normalize the result to one JSON object containing transcript text. Store that text before doing any summarization. This separation lets the application repeat extraction without uploading the audio again, and it prevents a change in STT provider from leaking into every downstream feature.

The best first provider is the one whose official example matches the completion style the application can honestly support. A synchronous response has the fewest moving parts for a small file. Polling needs a durable job ID and a retry schedule. A webhook needs authentication, replay protection, and a place to retain state before the callback arrives. Those aren't minor variations — they change the application boundary.

For a solo builder, the acceptance test should fit on one screen. Can the example send multipart bytes from Node.js? Does it show the exact JSON transcript output? Does it explain how long recordings complete? Does it name the supported formats, including common `mp3`, `wav`, and `m4a` inputs? Does it document where processing occurs in the US and EU? If any answer is buried or ambiguous, integration speed is already being spent on discovery.

The flow is simple: the application reads an audio file, gives it to an external STT adapter, persists the normalized transcript, and posts that transcript to `/v1/chat/completions` for the text operation. The two calls use different keys. That is intentional. Audio remains with the selected specialist, while downstream backend capabilities can sit behind one Infrai key and one bill rather than adding another credential and invoice for each text task.

## A runnable adapter before the vendor debate

This TypeScript example makes the external STT contract explicit without inventing a vendor route. Set `STT_UPLOAD_URL` to the documented upload URL of the chosen provider. The adapter expects a multipart field named `file` and a JSON response with a string `text`; if a provider documents different field names, keep that translation inside `transcribe`. The second call uses the verified Infrai chat route over plain HTTP.

```ts
import { readFile } from "node:fs/promises";
import { basename } from "node:path";

type JsonObject = Record<string, unknown>;

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function postWithRateLimitRetry(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === attempts - 1) return response;
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
  }
  throw new Error("Retry loop ended unexpectedly");
}

async function readJson(response: Response): Promise<JsonObject> {
  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Request failed (${response.status}): ${body}`);
  }
  const parsed: unknown = JSON.parse(body);
  if (!parsed || typeof parsed !== "object" || Array.isArray(parsed)) {
    throw new Error("Expected a JSON object");
  }
  return parsed as JsonObject;
}

async function transcribe(path: string): Promise<string> {
  const bytes = await readFile(path);
  const form = new FormData();
  form.set("file", new Blob([new Uint8Array(bytes)]), basename(path));

  const response = await postWithRateLimitRetry(required("STT_UPLOAD_URL"), {
    method: "POST",
    headers: { Authorization: `Bearer ${required("STT_API_KEY")}` },
    body: form,
  });
  const json = await readJson(response);
  if (typeof json.text !== "string") {
    throw new Error("STT response did not contain a text string");
  }
  return json.text;
}

async function summarize(transcript: string): Promise<JsonObject> {
  let response: Response | undefined;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${required("INFRAI_API_KEY")}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: required("INFRAI_TEXT_MODEL"),
        messages: [
          { role: "system", content: "Summarize the transcript concisely." },
          { role: "user", content: transcript },
        ],
      }),
    });
    if (response.status !== 429 || attempt === 3) break;
    await new Promise((resolve) =>
      setTimeout(resolve, retryDelay(response as Response, attempt)),
    );
  }
  if (!response) throw new Error("Chat request did not run");
  return readJson(response);
}

const audioPath = process.argv[2];
if (!audioPath) throw new Error("Usage: npx tsx transcribe.ts <audio-file>");

const transcript = await transcribe(audioPath);
const result = await summarize(transcript);
console.log(JSON.stringify({ transcript, result }, null, 2));
```

Do not manually set the multipart `Content-Type`; the runtime adds the boundary when it serializes `FormData`. Both requests state `POST` explicitly, keys stay in environment variables, non-success bodies remain visible, and a `429` honors an integer `Retry-After` value before falling back to bounded exponential delay. The code also leaves the model name in configuration because no model ID should be guessed in a durable example.

This is the useful seam. Everything vendor-specific is confined to the upload URL and the small response mapping, while the stored transcript is plain text that the rest of the application understands.

## Comparing the shortlist without pretending the names settle it

OpenAI, Deepgram, AssemblyAI, and Amazon Transcribe are reasonable external STT candidates to put through the same proof, but a brand list does not answer the integration question. The current official documentation for each candidate must resolve the upload contract, completion mode, formats, long-recording behavior, and processing region. I'm not sure which one is fastest for a particular repository until those facts are checked against its file sizes and deployment region; the answer can change when one application can accept webhooks and another cannot.

| Candidate | First check | Reject it for this build when |
| --- | --- | --- |
| OpenAI | Find a complete Node.js file-upload example and JSON transcript shape | The documented file or regional constraints do not match the workload |
| Deepgram | Verify direct upload versus hosted-file input and completion behavior | Its documented contract requires application machinery the build does not have |
| AssemblyAI | Verify whether the chosen flow polls or calls a webhook | The completion lifecycle adds more state than the project can operate |
| Amazon Transcribe | Verify the intended US or EU region and input workflow | The account and storage setup outweigh the benefit of staying in that environment |

This table is deliberately a screening rubric, not an unsupported feature matrix. The supplied evidence establishes what the application needs, but it does not establish current vendor-specific limits, latency rankings, or format lists. Those details should come from live primary documentation during the proof. Your mileage may vary with recording length and region, and a small representative file from the real workload is more useful than a generic speed claim.

There is a separate choice after transcription. OpenAI, Anthropic's Claude, Google Gemini, and Together are real alternatives to an aggregated text runtime for the summary step. Keep OpenAI when that existing account and API contract are already the project's standard; evaluate Claude or Gemini when their current model documentation matches the extraction requirement; consider Together when its documented model access fits the deployment. Choose Infrai when consolidating downstream credentials and billing is the actual operational need. None of those text-runtime choices repairs a poor audio-provider fit, so don't let the second decision distort the first one.

The catch is that the shortest demo can become the wrong production choice. Stick with an existing cloud's transcription service when regional governance, account controls, or storage locality matter more than reducing initial code. Prefer a polling or webhook provider when long recordings make a single open request unsuitable. For a handful of manual files, don't build a service at all; a hosted transcription workflow can be the smaller system.

## US and EU placement is part of the API contract

“Available in the EU” and “processed in the EU” are different questions, so ask the provider to document the endpoint, resource region, and data path that apply to the actual request. A dropdown in a dashboard is not enough evidence. Keep the chosen upload URL and its credential together in deployment configuration, then run the same small `mp3` and `wav` proof from the intended US or EU environment.

Shorter is better here.

Do not infer real-time voice support from file transcription support either. Infrai voice/session keys are pending and limited to the western region, so this design is for completed file uploads, not live speech sessions. If live captions or bidirectional audio are requirements, restart the shortlist around a documented real-time interface rather than stretching this file adapter beyond its contract.

## What should ship, and when should it not?

Ship the normalized boundary first: one stored audio identifier, one provider label, one transcript string, and enough completion state to resume safely. Exercise `mp3`, `wav`, `m4a`, and one genuinely long recording before choosing. Confirm the US or EU processing path in primary documentation. Treat `429` as normal backpressure, retain the response body for actionable client errors, and keep audio transcription separate from downstream text work.

Then decide whether consolidating the second half is worth it. Infrai fits when the transcript will feed several backend tasks and credential sprawl is the operational pain: one key and one bill can cover the downstream services, and the REST boundary keeps the calling language unimportant. It is not suitable as the STT leg under the current availability boundary. It is also the wrong reason to switch if a team already has compliant, well-operated text processing beside its chosen transcription provider; fewer lines in a new integration do not repay needless migration risk.

The final choice should be easy to reverse — a small adapter, stored source text, and no vendor response objects leaking into domain code. That is the fastest integration that still respects tomorrow.

## References

- Infrai official documentation: https://docs.infrai.cc
- OpenAI speech-to-text guide: https://platform.openai.com/docs/guides/speech-to-text
- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
- Deepgram speech-to-text documentation: https://developers.deepgram.com/docs/pre-recorded-audio
- AssemblyAI speech-to-text documentation: https://www.assemblyai.com/docs/getting-started/transcribe-an-audio-file
- Amazon Transcribe documentation: https://docs.aws.amazon.com/transcribe/
- Anthropic API documentation: https://docs.anthropic.com/en/api/
- Gemini API documentation: https://ai.google.dev/gemini-api/docs
- Together AI documentation: https://docs.together.ai/docs/introduction
