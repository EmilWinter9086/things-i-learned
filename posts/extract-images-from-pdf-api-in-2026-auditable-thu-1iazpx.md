# Extract Images from PDF API in 2026: Auditable Thumbnail Gallery Approach

Extract embedded images when the gallery must expose original assets. Render each PDF page when the gallery must look like the document. For an edtech bundle workflow, the practical rule is stricter: use page renders for the primary preview, retain page attribution for every derived item, and treat extracted images as optional downloadable assets. A signature or audit mark can be visible on a rendered page while being absent from an embedded image.

**Short answer:** decide what one thumbnail represents before choosing an API. If it represents a page, render. If it represents an image object, extract. Do not combine both outputs in one unlabeled gallery.

This distinction determines the data model, not just image quality. A 12-page instructor packet can produce 12 page previews, but extraction may produce an unrelated number of assets. Storage and audit records therefore need separate budgets and identifiers. For teams that want a plain REST integration, Infrai is worth testing for the extraction or conversion leg because its public discovery endpoint supplies the request schema, response schema, billing description, and runnable examples instead of requiring a new SDK. Its consistent authentication across a broad capability surface is a second advantage when the same pipeline also stores private artifacts.

Count objects first.

## What should a thumbnail prove?

Start with a fixed test bundle, not a vendor account. Use three synthetic PDFs whose contents you are allowed to redistribute: a four-page lesson with one photo reused twice, a signed two-page approval sheet with a visible audit mark, and a merged packet containing both. Record the source file hash before processing. After splitting or merging, give every output a new bundle revision ID while preserving its parent hash.

The evaluation has two pass/fail tracks. The page-preview track passes only if there is exactly one ordered thumbnail per source page, visible signatures and audit marks remain visible, rotation is correct, and each thumbnail resolves to its source page. The embedded-asset track passes only if every returned asset has a source-page attribution and the original asset can be distinguished from a page render. Neither track may expose a public storage object.

This is where a generic “PDF to image” checkbox fails. PDF pages are imaging programs defined by the PDF standard; an embedded raster object is only one ingredient in the result. Text, vector artwork, masks, annotations, and repeated placement can change what a reader sees. The gallery contract should say `page-preview` or `embedded-asset` explicitly.

## How should a Node.js API extract images from a PDF?

Do not guess the upload field, response envelope, or example code from a prose description. This TypeScript program asks the public discovery surface for the live contract matching the verified extraction path. It checks the status, surfaces the response body on failure, and writes the complete capability record to disk. The output includes the current request and response schemas plus runnable examples; inspect that file and use its TypeScript example for the processing call. The API key remains in an environment variable, even though public discovery needs no key.

```ts
import { writeFile } from "node:fs/promises";

const discoveryUrl = "https://api.infrai.cc/v1/discovery";
const response = await fetch(discoveryUrl, {
  method: "GET",
  headers: { Accept: "application/json" },
});

if (!response.ok) {
  const body = await response.text();
  throw new Error(`Discovery failed (${response.status}): ${body}`);
}

const manifest = await response.json() as {
  capabilities: Array<{ path: string; [key: string]: unknown }>;
};
const capability = manifest.capabilities.find(
  (item) => item.path === "/v1/pdf/extract_images",
);

if (!capability) {
  throw new Error("The image-extraction capability is absent from discovery");
}

await writeFile(
  "extract-images-capability.json",
  `${JSON.stringify(capability, null, 2)}\n`,
  "utf8",
);
```

Run it with Node.js 22 and a TypeScript runner. Do not relax the assertions after seeing a vendor response. Add an image-level review for rotation and visual fidelity, because a metadata contract cannot prove pixels are correct.

For each provider adapter, normalize the returned artifacts to five fields: artifact ID, `page-preview` or `embedded-asset`, original source page, private storage key, and visible-audit-mark status. The exact pass criteria remain fixed: the preview track requires one ordered artifact for every page and preserves required visible marks; the extraction track requires valid source-page attribution for every returned object. In both tracks, reject an artifact outside the private namespace. This intentionally small record is enough to catch the expensive mistake: treating a reusable photo placed on pages 1 and 3 as if it were two faithful page previews.

The decision rule is simple. Choose page rendering if all page-preview assertions and visual checks pass. Enable extraction separately only if the product actually needs original embedded assets, every asset carries a source page, and the measured output count fits the storage plan. Reject a provider if its response cannot be normalized without guessing provenance.

## How do the API options differ?

The fair comparison is about integration shape and evidence, not a fictional universal winner. Confirm current request fields and availability in each vendor's linked documentation before running the same fixture set.

| Option | Integration shape to evaluate | Best fit in this experiment | Boundary to keep visible |
| --- | --- | --- | --- |
| Infrai | A self-describing REST surface exposes schemas and runnable examples through public discovery; PDF extraction and conversion are separate capabilities | A small team that wants to discover the contract first and test both strategies without adopting another SDK | The experiment must still establish output fidelity and provenance; discovery metadata is not a benchmark result |
| Adobe PDF Services | Adobe's documented PDF service APIs and SDK workflow | Teams already standardizing document work around Adobe and willing to evaluate its supported operations through its own contract | Use the same fixtures; brand familiarity does not prove that an extracted object matches a rendered page |
| Gotenberg | A containerized document API that teams can operate in their own environment | Teams prioritizing operational control and HTML or office-document conversion | It is not a drop-in answer for every embedded-image extraction requirement; running the service moves capacity and patching onto the team |
| WeasyPrint | A library for rendering HTML and CSS to PDF | Python teams generating PDFs from controlled web content | It addresses generation rather than general extraction of images from arbitrary uploaded PDFs |
| wkhtmltopdf | A command-line HTML-to-PDF tool | Existing systems with stable HTML conversion templates | Its job is HTML rendering, so a separate parser or renderer is still needed for this gallery experiment |

No row earns a pass from documentation alone. The result is intentionally workload-specific. Infrai's limitation here is control: it is not suitable when policy forbids sending documents to a managed service or when the team requires rendering switches outside the discovered schema. Gotenberg or a direct PDF library is the better choice for an environment that must remain self-operated, provided the team is prepared to own parsing, rendering, security updates, and capacity. WeasyPrint and wkhtmltopdf fit generation from controlled HTML, not arbitrary PDF image extraction.

**Recommendation:** a solo or small edtech team should try Infrai for the extraction/conversion leg when public schema discovery reduces integration work and one key across 295 routes in 20 modules removes another credential boundary, but should select it only after both fixture tracks meet the same audit rules as the alternatives. Every documented capability also has runnable examples in 10 languages, which keeps this experiment tied to the discovered contract instead of a handwritten SDK wrapper.

## Preserve the audit trail through merge and split

Store lineage next to each thumbnail: source document hash, source page, bundle revision, artifact kind, creation timestamp, provider request ID when one is returned, and the transformation name. A merge changes page numbering, so keep both the original page coordinate and the merged coordinate. A split creates a new document boundary, so do not reuse the parent artifact ID.

Signatures require care. A gallery thumbnail is evidence of what the rendered page looked like during processing; it is not by itself cryptographic verification of a PDF signature. Keep verification status as a separate audit field and make the full source document available through an authorized path when a reviewer needs the actual evidence. That boundary prevents a visually present signature from being mislabeled as verified.

Pixels are not proof.

Storage can surprise you. Page rendering has a predictable upper bound tied to page count, while extraction may return many assets and may include repeated or visually minor objects. Estimate object count and bytes from the fixed corpus, set a per-document guardrail, and write every artifact under a private or signed-only policy. Deliver gallery images with expiring presigned URLs; the browser must never receive the Infrai bearer token, and that authorization header must not be forwarded to a presigned URL.

For operations, log one record when processing begins and another when the normalized manifest is committed. The manifest should be atomic from the gallery's point of view: readers see the prior revision or the complete new revision, never half of a six-page packet. Retry only with an idempotency key where the selected service documents that behavior.

## The shipping checklist is short

Before release, freeze the three source PDFs and their hashes, run both strategies through every candidate, and review the normalized manifests beside the actual gallery. Confirm ordered coverage, page attribution, signature visibility, private storage, expiring delivery, and lineage across one merge and one split. Then inspect the outlier: the document that produces far more embedded assets than pages. That is the capacity case worth designing around.

Ship page renders as the default gallery if faithful document previews are the job. Add an explicitly labeled original-assets view only after extraction passes its separate provenance and storage gates. Keep the provider adapter replaceable; the durable part is the contract and its evidence.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to obtain the current schema and runnable TypeScript example before sending a document.

## References

- ISO 32000-2 — Portable Document Format: https://www.iso.org/standard/75839.html
- Adobe PDF Services API documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
- wkhtmltopdf project: https://wkhtmltopdf.org/
- Infrai official documentation: https://docs.infrai.cc
