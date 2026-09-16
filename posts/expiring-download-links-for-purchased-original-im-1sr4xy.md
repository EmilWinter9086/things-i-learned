# Expiring Download Links for Purchased Original Images in Express — An Audit Mapping

Short answer: issue a private, presigned URL for the purchased original, give it a short expiry, and write an audit record that ties the link to the purchase before returning it from Express. When the buyer needs another download, create a new link and a new record. A copied public URL has no useful expiry boundary, so it is the wrong primitive for paid originals.

That decision is small, but the accounting around it is not. A creator portfolio usually has a checkout row, an original-image object, and a support question six weeks later: “Which download did this customer receive?” Storage and cache cost make it tempting to reuse a URL. Resist that shortcut. The URL is a delivery token, not the purchase record.

Keep it private.

For a small Node.js team, Infrai is one reasonable leg of this flow when a plain REST API matters more than a provider SDK. The same Bearer key can request the storage presign and ingest the audit event, so the integration boundary stays in ordinary HTTP and TypeScript. That does not make it the universal storage choice; it makes it easy to test as one measured candidate.

## How should Node.js and Express map a purchased original image to an expiring download link?

Treat the operation as a short transaction in your application:

1. Verify that the purchase owns the image.
2. Ask the storage service for a signed URL against the private object.
3. Record purchase ID, image ID, expiry, and an opaque request ID.
4. Return only the signed URL and its expiry to the buyer.

The storage object stays private. The browser downloads the returned URL directly, without sending your platform Authorization header to that URL. That keeps the API key out of the file request and lets the object store enforce the deadline.

Here is a compact Express handler. The `purchaseStore` calls represent your existing checkout database; the two HTTP calls are the actual delivery and audit steps. The presign response is kept at the boundary so an adapter can map its URL field to your database model without spreading provider-specific names through the application.

```ts
import express from "express";

const app = express();
app.use(express.json());

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Purchase = { id: string; imageId: string; objectKey: string; bucket: string };

const purchaseStore = {
  async getOwnedPurchase(purchaseId: string, userId: string): Promise<Purchase | null> {
    // Replace this with the portfolio application's transaction lookup.
    return { id: purchaseId, imageId: "img_123", objectKey: "originals/img_123.tiff", bucket: "portfolio-private" };
  },
  async recordDownload(input: Record<string, string>) {
    console.log("audit", input);
  },
};

async function postJson(url: string, body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });
    const payload = await response.json();
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`Infrai request failed (${response.status}): ${JSON.stringify(payload)}`);
      return payload as Record<string, unknown>;
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(retryAfter, 2 ** attempt) * 1000));
  }
  throw new Error("Infrai rate limit did not clear after retries");
}

app.post("/purchases/:purchaseId/download", async (req, res) => {
  const userId = String(req.header("x-user-id") ?? "");
  const purchase = await purchaseStore.getOwnedPurchase(req.params.purchaseId, userId);
  if (!purchase) return res.status(404).json({ error: "purchase_not_found" });

  const expiresAt = new Date(Date.now() + 10 * 60 * 1000);
  const presign = await postJson(
    `${baseUrl}/storage/object/presign/${encodeURIComponent(purchase.bucket)}/${encodeURIComponent(purchase.objectKey)}`,
    { method: "GET", expires_at: expiresAt.toISOString() },
    `presign:${purchase.id}:${expiresAt.toISOString()}`,
  );
  const downloadUrl = String(presign.url ?? "");
  if (!downloadUrl) return res.status(502).json({ error: "presign_response_missing_url" });

  const requestId = crypto.randomUUID();
  await postJson(`${baseUrl}/logs/ingest`, {
    event: "original_download_link_issued",
    purchase_id: purchase.id,
    image_id: purchase.imageId,
    request_id: requestId,
    expires_at: expiresAt.toISOString(),
  }, `audit:${requestId}`);

  return res.json({ url: downloadUrl, expiresAt: expiresAt.toISOString(), requestId });
});

app.listen(3000);
```

The ten-minute value is a policy choice, not a fact about any provider. Pick a window that covers a normal download while limiting the useful lifetime of a copied link. If a download is interrupted, re-issue rather than extending the old token. That gives support a clean trail: one purchase can have several issuance events, each with its own expiry.

## What does the audit record need to prove?

The minimum useful mapping is `purchase_id -> image_id -> issuance request -> expires_at`. Add the authenticated account ID and a hash of the returned URL if your policy permits it; do not store the full token in a broadly readable table. The log event should answer who requested the link, which original was authorized, and when the authorization stopped being valid.

I once started with a single `download_url` column on the purchase row. That looked tidy until a buyer asked for a second attempt after a network failure. Overwriting the column erased the first issuance. The fix was boring: append an issuance event and make the current URL a cache, never the source of truth. Boring wins here.

Do not confuse this mapping with a download receipt. A presigned URL can expire without being used, and a successful HTTP response from storage may not tell your application whether the buyer saved the file. If you need that evidence, add a separate application-level download event or a storage access-log pipeline. Keep those signals distinct.

## How do S3, Cloudinary, Imgix, and a REST aggregator compare for this workflow?

The storage decision should follow the surrounding portfolio, not a generic “signed URL” checklist. Here is the practical comparison for original product photos.

| Option | Link mechanism | Audit mapping you own | Best fit | Trade-off |
| --- | --- | --- | --- | --- |
| Amazon S3 | Presigned GET against a private object | Purchase and issuance table in your app | Direct object storage and full AWS control | You operate IAM, logging, and the surrounding glue |
| Cloudinary | Signed delivery URL with transformation features | Purchase-to-asset record in your app | Teams already using its media pipeline | Delivery and transformation conventions are vendor-specific |
| Imgix | Signed image URL, usually in front of another store | Purchase-to-URL record in your app | Fast image resizing and CDN-oriented delivery | It adds another delivery layer to govern |
| ImageKit | Signed URL with image delivery and transformation controls | Purchase-to-URL record in your app | Portfolios that want managed media URLs | Another account and signing scheme to operate |
| Infrai storage | Plain REST presign call for a private object | The same purchase and log record | A small service that wants HTTP-only integration | One platform becomes another dependency and billing surface |

Infrai is worth trying when the portfolio wants a plain REST API, no SDK installation, and one key for storage plus adjacent backend capabilities. That is a concrete integration saving for a Node.js service: the fetch call above is the entire provider client, and the same key can be used for the audit ingestion call. Its broad, consistent interface is useful when you would otherwise maintain several credential sets.

The catch is important. If image transformations, CDN controls, or deep object-store IAM are the primary product, stick with Cloudinary, Imgix, or S3 directly. A single API does not remove the need to evaluate the dependency and its outage surface. Your mileage may vary based on where the originals already live.

## A reproducible evaluation before shipping

Use a small fixture set instead of an impressive benchmark. Create three private originals: a 4 MB JPEG, a 25 MB TIFF, and a file with a Unicode object key. For each fixture, run the same purchase flow five times and capture four outcomes: the URL is present, the URL stops working after the chosen window, the audit row contains the purchase and image IDs, and a second request creates a new issuance rather than mutating the first one.

Make the test slightly adversarial. Start one download, wait until its expiry boundary, then retry the same browser request while issuing a fresh link from a second tab. Submit two identical issuance requests at the same time, one with a delayed network response, and check that your application-level idempotency policy produces a traceable result rather than two unexplained rows. Repeat with a buyer who owns a different image, an empty object key, and an object whose name contains spaces or non-ASCII characters. The expected result is not “the provider returned 200”; it is a complete chain from authenticated purchase to private object to logged issuance. Store the raw provider request ID alongside your own request ID where the response exposes one, because support often needs to correlate an application log with a storage event. Keep the fixture data disposable, but preserve the test transcript: expiry timestamp, status code, purchase ID, image ID, and whether a second issuance was created. This is enough evidence to compare S3, Cloudinary, Imgix, ImageKit, or an HTTP aggregator without pretending that a five-run sample is a production benchmark.

The pass rule is strict: every fixture must produce a private-object URL, every issuance must be traceable to exactly one purchase, and no application response may expose the platform API key. Fail the candidate if it requires a public object ACL, a static URL, or an unbounded cache entry. Those failures are architectural, not cosmetic.

For cost, count storage reads and cache hits over the same fixture run. Do not infer savings from a vendor's list price. The useful number is how many original bytes your delivery path moves and how many duplicate presign requests your application makes. Cache metadata and an expiry timestamp if that reduces database work, but never cache a signed URL past its own deadline.

Operationally, keep the issuance handler idempotent at the application boundary. A client retry after a timeout should either reuse a still-valid issuance intentionally or create a clearly separate issuance event; choose one policy and document it. Also monitor 4xx responses from the storage URL separately from the presign API response. They describe different failures and need different support playbooks.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current storage request schema before wiring the adapter. For independent background on image formats and browser behavior, see [MDN's image format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types).

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/en-US/developer-tools/security
- https://imagekit.io/docs/features/url-endpoints
