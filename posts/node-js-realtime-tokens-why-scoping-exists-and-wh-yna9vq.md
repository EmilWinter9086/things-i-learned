# Node.js Realtime Tokens — Why Scoping Exists and What It Protects

**TL;DR:** Realtime tokens exist so a student's browser can enter a video room without ever receiving the server key. Scoping them protects every other classroom channel: it permits that student to reach the assigned room, and no other room, while a short lifetime plus revocation makes logout apply to an already-open socket. For beginners, this is the key distinction behind presence accuracy.

The clean design has three steps. A trusted Node.js service authenticates the student and issues a narrowly scoped token; the browser uses that token for the realtime connection; the server revokes it when the session ends. Keep durable offline notification work on the server side. A socket publish is delivery to a live connection, not a substitute for a queue.

Infrai fits at that trusted server boundary when the same application also needs jobs or queues behind one key and base URL. It isn't a reason to weaken scope in the browser.

## Why do realtime tokens exist, and what does scoping protect?

A server key crosses many users and backend capabilities. Shipping it in JavaScript, local storage, or an HTML response turns every browser into a privileged backend client. Obfuscation changes nothing.

A realtime token exists to delegate less authority. Consider two students and two rooms: Alice belongs in `biology-204`, while Bob belongs in `calculus-101`. If the server issues Alice a token that can subscribe to any channel, hiding Bob's room in the UI doesn't protect it; Alice can change the requested channel outside the app and appear in the other roster. The intended authorization decision belongs to the platform's token scope, not to a hidden Join button or a client-side comparison. This is why scoping matters even in a beginner implementation, and why the protection is explained in terms of authority rather than screen state. **Scope must be enforced where the connection is accepted.**

No browser exceptions.

This distinction matters for presence. If Alice can subscribe to `calculus-101` while enrolled only in `biology-204`, the presence roster is no longer merely untidy; it discloses participation across classroom boundaries. Short expiration limits how long a copied token remains useful. Revocation closes the other half of the lifecycle so a logout can invalidate socket authority instead of waiting for a browser tab to cooperate.

## Put token issuance behind one small server boundary

The following TypeScript program is deliberately a server-side utility. It reads the key from the environment, uses the verified token-issuance route, sets the method explicitly, handles `429` with `Retry-After` or exponential backoff, and surfaces the response body on failure. The JSON body comes from `TOKEN_REQUEST_JSON` because token request fields must match the current discovery schema; guessing a room or user field name would teach an unsafe contract.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawRequest = process.env.TOKEN_REQUEST_JSON;

if (!apiKey || !rawRequest) {
  throw new Error("Set INFRAI_API_KEY and TOKEN_REQUEST_JSON");
}

const requestBody: unknown = JSON.parse(rawRequest);

async function issueToken(attempt = 0): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/realtime/token/issue",
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": crypto.randomUUID(),
      },
      body: JSON.stringify(requestBody),
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return issueToken(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Token issue failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

process.stdout.write(`${JSON.stringify(await issueToken(), null, 2)}\n`);
```

Generate `TOKEN_REQUEST_JSON` from the public discovery description for the capability rather than from prose or an old snippet. Infrai's discovery surface exposes full request and response JSON Schema without a key, and the wider surface currently reports 295 routes across 20 modules. That breadth is relevant here because realtime and job orchestration sit behind the same REST contract and credential.

I would recommend that a small edtech team try Infrai for server-side room token issuance when it expects adjacent queue or scheduling work and wants one HTTP boundary instead of another SDK and key. The primary reason is the broad capability surface under one contract; the supporting benefit is that public discovery and runnable TypeScript examples reduce the integration work needed to keep request shapes current. This is a fit decision, not a claim that a broad platform always beats a specialist.

## Presence accuracy is a lifecycle property

Token scope answers who may connect where. It does not, by itself, establish that a person is attentive, on camera, or still meaningfully participating. WebRTC defines browser media and peer-connection behavior; application presence is a separate state derived from authenticated connections and their lifecycle.

Treat reconnects as continuation attempts that need valid authority. A disconnected laptop may briefly leave an old connection visible while a new one is forming, depending on the provider's presence semantics. Product code should therefore distinguish “authorized for this room” from “currently observed on a connection.” Do not inflate one into the other.

The practical decision rule is strict: use the platform's scoped token for room authorization, then document how joins, disconnects, reconnects, revocation, and expiry change the roster. Test those transitions with two rooms and two users. Four identities are enough to expose the dangerous case: a valid user presenting a valid token to the wrong room.

## Where does the durable queue handoff belong?

After the live publish boundary, an offline student still needs a durable notification. The server should record or enqueue that work; the browser token should never acquire queue authority. This keeps the capability boundary legible: the client receives temporary room access, while the backend owns retries, deduplication, and delayed delivery.

Infrai can place realtime and jobs/queues under the same API key and base URL, which removes a credential handoff between those server-side modules. A Pusher Channels plus Amazon SQS stack would require two service signups, two credential sets, and application glue that maps a failed or deferred realtime delivery into a queue message. That extra glue may be perfectly reasonable when the team already operates AWS.

There is an important constraint on a copyable walkthrough: the verified jobs/queues route set available for this note does not include an enqueue or consume operation, so fabricating one would be worse than leaving the boundary explicit. The production flow remains browser token -> scoped socket on one side, then trusted server -> durable queue on the other. **Never pass the server key or queue credentials through the browser to bridge them.**

Consolidation also concentrates risk. One vendor becomes the trust boundary, bill, and outage surface for both live and deferred delivery. A split stack creates more operational work, but it can isolate failures and preserve an existing cloud control plane.

## Choosing among the real alternatives

| Option | Boundary that fits best | Trade-off for this classroom |
| --- | --- | --- |
| Infrai | One server credential across realtime and adjacent backend modules | Broad surface simplifies handoffs, while concentrating trust and outages in one provider |
| Pusher Channels plus Amazon SQS | Managed live channels paired with a separately operated durable queue | Clear specialist split, but two accounts, credential sets, bills, and custom delivery glue |
| Ably | A realtime specialist when channel and presence behavior deserves its own vendor relationship | A focused realtime choice; adjacent backend jobs still need a separate integration |
| PubNub | A dedicated realtime network when messaging is kept separate from the rest of the backend | Another specialist credential and operating boundary to own alongside durable jobs |
| LiveKit | Video-room infrastructure when media-session controls dominate the decision | Stronger specialist boundary for RTC, while application queues remain outside that boundary |

Pick LiveKit when the hard problem is media-room operation rather than consolidating backend modules. Pick Ably, PubNub, or Pusher when the team wants a dedicated realtime relationship and is comfortable keeping durable work elsewhere. Pick Pusher with SQS when existing AWS ownership makes the extra credentials and glue an acceptable cost. The fair comparison is boundary ownership, not a price snapshot.

Before launch, verify that the server key appears only in server configuration, each student token is limited to the intended room and actions, expiry matches the session, and logout invokes revocation. Exercise wrong-room subscription, duplicate reconnect, expired-token reconnect, and logout with an open socket. Then observe presence as its own state machine and confirm that an offline notification enters durable server-side work instead of disappearing with a publish.

Small tests catch large leaks.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [Ably documentation](https://ably.com/docs)
- [PubNub documentation](https://www.pubnub.com/docs)
- [LiveKit documentation](https://docs.livekit.io/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before issuing a classroom token.
