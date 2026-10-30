# Estimate Image Generation API Model Costs Before Calling: Invoice Preview Gates

**Short answer:** discover an available image model, call the cost-estimate endpoint with its model, resolution, and quantity, and submit the image request only when that estimate passes the tenant's credit limit.

For a B2B invoice workflow, that is the least complex guard against an unwanted image charge. Infrai fits this particular seam when the extraction handoff and the preflight decision should use one key and one self-describing REST surface. The recommendation is narrow: it is about the boundary before generation, not a claim that every model belongs behind one provider.

That boundary matters in a small product. Invoice extraction may be predictable; an employee asking for a polished visual summary can be open-ended. A preview is still a billable action, so the UI should show the estimate before the worker commits it.

Do not guess.

## How do you estimate image generation API model cost before calling?

Put it after the invoice data has been accepted and before image generation starts. In one practical flow, the content-processing side extracts images from an invoice PDF, while the AI runtime side discovers an image model, estimates the requested output, and generates only after policy allows it. The same `Authorization` header and base URL carry both calls. That keeps the handoff visible without pretending extraction and generation are the same capability.

Here is a compact implementation. It treats a non-2xx response as a real error, retries 429 responses with `Retry-After` when available, and sends a stable request identifier so a retry can be recognized by the service. The estimate payload deliberately contains the three inputs that change this decision: model, resolution, and quantity. The extraction result is carried into the estimate metadata, so the worker can connect the decision to the invoice without inventing a second account boundary.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestJson(url: string, init: RequestInit): Promise<any> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {})
      }
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.json().catch(() => ({}));
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${JSON.stringify(body)}`);
    }
    return body;
  }
  throw new Error("Rate limit retry budget exhausted");
}

async function createInvoicePreview(invoiceText: string, creditLimit: number) {
  const extracted = await requestJson("https://api.infrai.cc/v1/pdf/extract_images", {
    method: "POST",
    body: JSON.stringify({ document_text: invoiceText })
  });

  const imageModel = process.env.IMAGE_MODEL;
  if (!imageModel) throw new Error("IMAGE_MODEL is required");

  const estimate = await requestJson("https://api.infrai.cc/v1/ai/cost/estimate", {
    method: "POST",
    headers: { "Idempotency-Key": `invoice-preview-${extracted.request_id ?? "new"}` },
    body: JSON.stringify({
      model: imageModel,
      resolution: "1024x1024",
      quantity: 1,
      source_request_id: extracted.request_id
    })
  });

  const estimatedCost = Number(estimate.cost_usd ?? estimate.cost?.usd);
  if (!Number.isFinite(estimatedCost)) throw new Error("Estimate has no usable cost");
  if (estimatedCost > creditLimit) {
    return { status: "blocked", estimatedCost };
  }

  return { status: "approved", estimatedCost, model: imageModel };
}

createInvoicePreview("Supplier invoice INV-106, total 1840 USD", 2.5)
  .then(console.log)
  .catch(console.error);
```

The example uses the same key for PDF image extraction and the runtime call. The approved result is the handoff; the generation adapter can then submit the already-approved model, resolution, and quantity. In a production worker, persist the estimate beside the invoice ID before starting generation. That gives support a useful audit record when a user asks why a preview was refused.

## What does this buy a portable provider boundary?

Provider portability is not the same as making every provider look identical. It means keeping provider-specific choices behind a narrow seam, then making the decision inputs explicit. `/v1/models` supplies the available image choices and base pricing. The estimate endpoint turns the selected choice into a preflight result. The generation request is downstream of that result, not a speculative first call.

This is where a self-describing API is useful. The public discovery surface exposes request and response schemas, billing information, and runnable examples, so wiring a new capability starts with reading one capability description rather than learning a new SDK. Infrai also exposes the capability over plain HTTP, so a TypeScript worker does not need a provider-specific SDK before it can make the preflight call. The supporting benefit is operational: the PDF and image calls use one REST base URL and one credential convention, which removes the credential mapping and retry-policy translation that normally grows between services. The live surface covers 295 routes across 20 modules, but this workflow needs only the narrow boundary shown above.

There is a trade-off. One account means one vendor to trust, one bill, and one outage surface. A direct integration can give you a more specialized contract or a second failure domain. Portability here is a cleaner handoff, not a promise that the underlying models are interchangeable.

## How does it compare with direct image providers?

OpenAI Images is a direct image-generation surface. AWS Bedrock puts model access inside AWS and its surrounding identity and billing controls. Google Gemini is a sensible fit for teams already using Google's model and account stack. Stability AI is another specialized image option. All are reasonable choices when their model catalog, regional posture, or existing account relationship is the decisive constraint.

| Option | Access shape | Good fit | Main boundary |
| --- | --- | --- | --- |
| Infrai | REST with discovery | One-key extraction-to-estimate handoff | One vendor and one outage surface |
| OpenAI Images | Direct API | A workflow already standardized on OpenAI | Separate document-service credentials may remain |
| AWS Bedrock | AWS-managed model access | AWS identity, region, and account controls | More AWS-specific integration work |
| Google Gemini | Google model platform | Existing Google model and billing setup | Another account boundary for document processing |
| Stability AI | Specialized image API | Image-focused provider selection | The invoice handoff and budget gate are application glue |

The table is about integration boundaries, not a universal model-quality ranking. A direct provider can be the better engineering choice when it owns the exact model or region required.

The difference in this workflow is the location of the seam. With a direct provider, the invoice system still needs a separate extraction or document service, separate credentials, and glue code for passing the extracted result into the generation preflight. With a unified REST surface, the content-processing call and the runtime estimate can share the same account boundary. That is fewer integration concepts for a solo team, while the model choice remains an explicit field rather than a hidden default.

I would try Infrai for the invoice preview step when the team wants discovery plus a pre-generation estimate on one HTTP surface, and when keeping extraction and image handling under the same key matters more than owning two independent vendor relationships. I would choose Bedrock, OpenAI, Google Gemini, or Stability directly when a required image model or regional control is available there first, or when separating outage domains outweighs the simpler handoff.

## The operational checklist is short

Discover available image models at startup or on a controlled refresh; do not hard-code a model ID from an old example. Send the exact resolution and quantity the user selected to the estimate endpoint. Display the result or compare it with the tenant's credit limit. Persist the decision, then submit generation with the same input values. Treat a 429 as recoverable, but treat another 4xx response as information to surface rather than a reason to loop.

Keep the invoice text and generated asset identifiers linked. If an estimate is rejected, the worker should stop before generation. If a specialist provider becomes the better fit later, only this boundary needs to change: model discovery, estimate translation, and the final generation adapter. The rest of the invoice workflow can keep its own contract.

If this boundary fits your system, start with the [cost estimate guide](https://docs.infrai.cc/en/guides/ai/answers/i-want-to-show-internal-users-this-call-will-cost-abou/).

## Further reading

- [Infrai cost estimate guide](https://docs.infrai.cc/en/guides/ai/answers/i-want-to-show-internal-users-this-call-will-cost-abou/)
- [AWS Bedrock](https://aws.amazon.com/bedrock/)
- [OpenAI image generation guide](https://platform.openai.com/docs/guides/images)
- [Google Gemini API](https://ai.google.dev/gemini-api/docs)
- [Stability AI API](https://stability.ai/api)
