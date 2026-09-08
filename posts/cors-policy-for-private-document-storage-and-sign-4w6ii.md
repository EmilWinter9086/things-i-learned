# CORS Policy for Private Document Storage and Signed Uploads

Choose the path by deciding when an uploaded byte becomes an application document. For a SaaS product storing user documents, direct browser upload can keep API capacity free, but the application must still authorize the upload and accept the object after checking it. A server proxy is the better boundary when bytes must be rejected before they reach durable storage.

## TL;DR

Use short-lived signed writes for large browser uploads when storage CORS is under your control and inspection can happen after the write. Use a streaming server proxy when the application must apply an inline control before persistence. In both designs, keep the storage location private, use opaque object keys, and make a server-side acceptance record the only source of download authorization.

The data flow is small: a signed-in client requests permission, the application records a pending upload with an opaque key, and bytes travel either directly to storage or through the API. A separate server action verifies the stored object and changes the record to accepted. Downloads consult that record, not a filename or a key supplied by the browser.

This distinction avoids a common mistake: treating a successful PUT as proof that a user owns a safe, available document.

## How should browser direct upload, CORS, signed URLs, and server proxy uploads protect private SaaS document storage?

A signed URL grants a narrow storage operation for a limited time. It should be issued only after the application authenticates the caller, checks tenant membership, applies a byte limit, chooses the object key, and creates a pending record. CORS is different. It tells a browser which origins, methods, and headers may make cross-origin requests; it does not establish document ownership and does not replace application authorization.

For a direct browser upload, allow the exact production origins, the PUT or POST method actually used, and only the headers the signed request requires. A broad origin rule makes debugging easy and policy review hard. The signed request binds the write; the session-bearing application request authorizes issuance of that signed request. Keep those decisions separate.

The browser should never select a durable key such as `tenant/filename.pdf`. Filenames collide, expose meaning in logs, and change when a user renames a document. Generate an opaque key, retain the display name in application metadata, and place pending objects under a distinct prefix or tag. A private bucket or equivalent private container matters on both upload paths: a stored object must not become readable merely because somebody knows its key.

The catch is that direct upload adds a distributed state transition. The browser can finish the transfer but close before it calls finalize; finalization can be retried; an object can be absent or differ from the authorized size and type. The server needs to make these cases explicit. It should accept the document only after a metadata lookup confirms the expected object, and a second finalize request should return the already-accepted result rather than schedule duplicate work.

With a proxy, the application receives every byte. That permits streaming checks, content limits, and backpressure before storage sees the complete object. It also means the API tier pays for bandwidth, open connections, and time spent serving slow clients. Buffering a large file in memory is a bad bargain. Stream it with a byte ceiling and a request deadline.

There is no universal cutoff by file size. Network reliability, inspection requirements, API concurrency, and client-network restrictions matter more than a round number. Your results will depend on the documents and connections your product actually sees.

## A TypeScript acceptance flow before the policy debate

The application contract below has no storage-vendor route in it. `ObjectStore` can represent any implementation able to create a time-limited write and inspect object metadata. The important part is the pending-to-accepted transition, which remains in the application's own data store.

```ts
import { randomUUID } from "node:crypto";

type UploadStatus = "pending" | "accepted";

type Upload = {
  id: string;
  tenantId: string;
  objectKey: string;
  expectedBytes: number;
  expectedContentType: string;
  status: UploadStatus;
};

interface ObjectStore {
  createSignedWrite(input: {
    key: string;
    contentType: string;
    expiresInSeconds: number;
  }): Promise<{ url: string; headers: Record<string, string> }>;
  getMetadata(key: string): Promise<{
    bytes: number;
    contentType: string;
  } | null>;
}

interface Uploads {
  insert(upload: Upload): Promise<void>;
  find(id: string, tenantId: string): Promise<Upload | null>;
  accept(id: string): Promise<void>;
}

export async function authorizeUpload(
  store: ObjectStore,
  uploads: Uploads,
  tenantId: string,
  input: { bytes: number; contentType: string }
) {
  if (input.bytes < 1 || input.bytes > 25 * 1024 * 1024) {
    throw new Error("upload_size_rejected");
  }

  const upload: Upload = {
    id: randomUUID(),
    tenantId,
    objectKey: `pending/${randomUUID()}`,
    expectedBytes: input.bytes,
    expectedContentType: input.contentType,
    status: "pending"
  };
  await uploads.insert(upload);

  const target = await store.createSignedWrite({
    key: upload.objectKey,
    contentType: upload.expectedContentType,
    expiresInSeconds: 300
  });
  return { uploadId: upload.id, ...target };
}

export async function acceptUpload(
  store: ObjectStore,
  uploads: Uploads,
  tenantId: string,
  uploadId: string
) {
  const upload = await uploads.find(uploadId, tenantId);
  if (!upload) throw new Error("upload_not_found");
  if (upload.status === "accepted") return { status: "accepted" as const };

  const object = await store.getMetadata(upload.objectKey);
  if (
    !object ||
    object.bytes !== upload.expectedBytes ||
    object.contentType !== upload.expectedContentType
  ) {
    throw new Error("uploaded_object_mismatch");
  }

  await uploads.accept(upload.id);
  return { status: "accepted" as const };
}
```

The client must send the returned headers exactly as signed. Adding an unplanned content header can trigger a preflight request, and changing a signed header can make the storage request invalid. Keep signed URLs out of logs, error reports, analytics events, and support screenshots; they are credentials for their brief lifetime.

The example leaves content inspection as a product policy. An application that accepts PDFs may validate a declared type, inspect file signatures, scan content, or delay availability until a separate processor completes. The state model should have a place for that policy, rather than pretending that a browser-supplied MIME type is enough.

## What fails after the happy-path upload?

Most production trouble sits between authorization and acceptance. Test the authorization endpoint with a caller from another tenant, guessed upload IDs, a stale session, and a filename designed to look like a key. Test finalization before an object exists, after it has been accepted, and after the declared size or type differs from stored metadata. Each test should either reject the transition or return the existing accepted state without creating another downstream task.

Direct uploads need a small CORS matrix in automated browser tests: the production origin succeeds; an unrecognized origin cannot read the response; required request headers are allowed; unused methods are absent. Also test an expired signature, a changed method, a changed content type, a partial body, and a client disconnect. These are separate conditions. Combining them into one broad browser test hides which policy changed.

One useful acceptance test is deliberately unglamorous. Create a pending upload for Tenant A, upload an object whose key was generated for that record, then attempt every follow-up as Tenant B: metadata lookup through the application, acceptance, download authorization, and deletion. Repeat the sequence with a valid session for Tenant A but a guessed upload ID, then with an object key copied from a diagnostic event. None of those inputs should turn into a document response. Next, run the normal path twice: authorize once, write once, accept twice. The first acceptance may enqueue the next policy-controlled stage; the second must observe the existing state. Finally, leave a pending object behind on purpose and confirm that cleanup sees the pending scope while accepted objects with a similar age remain untouched. This sequence catches an awkward class of defects where each individual endpoint appears protected, but the identifiers do not refer to the same tenant and state transition. It also forces a concrete answer to a product question teams often postpone: what is the user shown while inspection or processing is pending, and which action is safe to retry? The answer belongs in the application record, because browser success and object existence are evidence of transfer, not of availability.

Lifecycle rules deserve the same care. Object lifecycle management can transition or expire objects based on policy, so pending uploads need a scope that is distinct from accepted documents. A cleanup rule should identify abandoned pending objects without sweeping accepted records into the same deletion path. The database deletion workflow should likewise have a durable record of the storage deletion request and its result; deleting application metadata alone does not establish that document bytes are gone.

Cost often appears as retry behavior, duplicate processing, and retained pending objects rather than the line item someone first notices. Record attempted bytes, accepted bytes, pending age, and processing attempts against the upload ID. Those fields make it possible to separate a client retry pattern from an increase in actual document volume. Don't store signed query strings or document contents in those events.

## Operating the chosen boundary

For direct writes, release only after verifying the deployed CORS policy, signed-header contract, expiration behavior, tenant checks, finalization idempotency, and pending-object cleanup against a production-like environment. For a proxy, verify streaming backpressure, byte limits, connection timeouts, and the same tenant and acceptance checks. Both paths need metrics for authorization attempts, rejection reason, accepted bytes, pending age, cleanup volume, and latency from authorization to acceptance.

Keep the application database authoritative for tenant ownership and visibility. Keep object storage authoritative for bytes. This keeps document access decisions reviewable and prevents an object key from becoming a second identity system.

| Constraint | Direct signed write | Streaming server proxy |
| --- | --- | --- |
| API bandwidth | File bytes bypass the API tier | Every byte crosses the API tier |
| Inline controls | Apply after transfer, before acceptance | Apply while streaming |
| Browser policy | Requires a narrow storage CORS policy | Browser needs only the application origin |
| Unreliable large transfers | May need resumable semantics | API connection remains part of the transfer |
| Operational state | Signing, acceptance, and cleanup | Backpressure, limits, acceptance, and cleanup |

Use direct signed writes when upload traffic would compete with API latency and later acceptance is acceptable. Stick with a streaming proxy when inline control or client-network policy is more important than removing bytes from the API tier. The choice is a boundary decision, not a storage brand decision.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
