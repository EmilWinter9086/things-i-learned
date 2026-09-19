# Fair Bid Ordering with Authoritative Timestamps and Explicit Tie Breakers

Short answer: order accepted bids by an authoritative server timestamp, then by a server-assigned sequence number. Treat message arrival at a bidder's browser as presentation timing, never as the auction result. In a live logistics capacity auction, the same rule keeps a carrier's delayed dashboard from rewriting who won while device-status updates continue on a separate, less trusted path.

The decision rule is deliberately small: the authority that accepts a bid also fixes its order. A client may display a local estimate, but it cannot supply the timestamp, sequence, or closing decision that settles money. This costs an extra server round trip before a bidder sees final acceptance. Take that trade. Client clocks and network paths sit outside the trust boundary.

## Should bids follow a server timestamp or message arrival?

Fairness must be stated before transport is chosen. For this system, a bid is eligible if the auction authority receives and validates it before the close recorded by that authority. Eligible bids sort by `acceptedAt`, then by `sequence`. The second key makes equal timestamp values deterministic instead of leaving the outcome to database scan order or a JavaScript sort assumption.

Arrival is not one event. A bid can reach an edge, an application process, durable storage, a broadcast relay, and each bidder at different times. Ordering by the last of those points rewards the network path to an observer. Ordering by a timestamp created on the bidder's device is worse: that device is untrusted and its clock can be wrong or deliberately altered. The useful timestamp is created inside the acceptance transaction, at the boundary named in the auction rules.

This does not make latency disappear. A carrier on a slow link still has less time to submit. It makes the adjudication rule auditable: one authority, one cutoff, and one deterministic tie breaker. If the business instead wants sealed bids or a grace window, encode that policy explicitly. Do not smuggle it in through arrival order.

That is the line.

The data flow is straightforward. An authenticated bidder sends an intent containing an auction ID, amount, and unique request ID. The authority checks token scope, auction state, and request uniqueness; it then assigns acceptance metadata and commits the bid. Only after commit does the fan-out layer publish the accepted record to dashboards. Vehicle telemetry uses a distinct topic and narrower credentials, so permission to report a truck's status cannot become permission to place a bid.

## Put acceptance before broadcast

The following TypeScript example keeps the ordering decision in one in-memory authority so the invariant is visible. Production storage must provide equivalent atomicity for request deduplication, sequence allocation, close checking, and insertion. The transport callback is downstream of the commit and therefore cannot affect rank.

```ts
type BidIntent = {
  auctionId: string;
  requestId: string;
  amountCents: number;
};

type Claims = {
  subject: string;
  scopes: ReadonlySet<string>;
  auctionIds: ReadonlySet<string>;
};

type AcceptedBid = BidIntent & {
  bidderId: string;
  acceptedAt: number;
  sequence: number;
};

class AuctionAuthority {
  private sequence = 0;
  private readonly accepted = new Map<string, AcceptedBid>();

  constructor(
    private readonly closesAt: number,
    private readonly now: () => number,
    private readonly publish: (bid: AcceptedBid) => void,
  ) {}

  place(intent: BidIntent, claims: Claims): AcceptedBid {
    if (!claims.scopes.has("bid:place") || !claims.auctionIds.has(intent.auctionId)) {
      throw new Error("bid not authorized");
    }

    const duplicate = this.accepted.get(intent.requestId);
    if (duplicate) return duplicate;

    const acceptedAt = this.now();
    if (acceptedAt > this.closesAt) throw new Error("auction closed");
    if (!Number.isSafeInteger(intent.amountCents) || intent.amountCents <= 0) {
      throw new Error("invalid amount");
    }

    const bid: AcceptedBid = {
      ...intent,
      bidderId: claims.subject,
      acceptedAt,
      sequence: ++this.sequence,
    };
    this.accepted.set(intent.requestId, bid);
    this.publish(bid);
    return bid;
  }
}
```

There is a deliberate trap in this compact example: a process-local counter is not enough once several writers can accept bids. At that point, sequence allocation and acceptance need one serialization mechanism in the durable write path. The public contract stays the same; storage must guarantee it. Sharding by auction is a practical boundary because bids in different auctions do not need a shared order.

The unique request ID handles a common retry failure. A bidder can time out after the authority commits but before the acknowledgement returns. Retrying the same intent should return the accepted record, not create a later second bid. A real implementation must bind that identifier to the authenticated bidder and original payload, rejecting reuse with different terms.

This design has limits. A single acceptance authority is a poor fit when the auction rules require several independent organizations to agree on an event before it becomes final, or when bids must remain secret until a coordinated reveal. A server timestamp also cannot make unequal network access fair; it only makes the cutoff consistent at the named authority. Those cases need a different protocol and governance model, such as sealed submissions with a published reveal procedure, rather than a faster broadcast channel. There is an operational trade-off too: serializing each auction simplifies the audit trail but caps write concurrency within that auction. Partitioning by auction restores parallelism across sales, while a single very busy sale may require batching or a formally specified allocation rule. The right answer then depends on peak bid rate, acceptable acknowledgement delay, and the legal meaning of the close. None of those can be inferred from message arrival at a dashboard.

## Keep transport semantics out of the verdict

A live dashboard may use a reliable ordered channel, but its observed order is still only the order visible on that connection. W3C WebRTC 1.0 defines data channels over SCTP and exposes ordered delivery behavior; it also permits configurations involving unordered delivery and retransmission limits. Those controls change delivery behavior between peers. They do not establish a global auction order across reconnects, relays, or multiple recipients.

Every broadcast bid therefore carries `acceptedAt` and `sequence`. A reconnecting dashboard requests a snapshot, replaces its local view, and then applies records with later sequence values. If it sees a gap, it pauses the animation and fetches state rather than guessing. Fast motion is cosmetic. Correct rank is not.

Never guess.

Keep device status separate. A tractor's location or availability update can tolerate coalescing because a newer state may supersede an older one; an accepted bid is an immutable decision record. Giving both streams the same retry, retention, and authorization policy expands the damage a leaked telemetry credential can do. Mint narrowly scoped, short-lived credentials for the exact auction or device set, enforce those claims at the authority, and assume every browser can be inspected by its operator.

The browser gets enough information to render and submit. It does not get a secret that can mint broader credentials, select its own bidder identity, or write acceptance metadata.

Keep that boundary boring.

## Test the boundary, not the animation

Start with a deterministic clock and inject competing requests around the close. Test an accepted request retried after a simulated lost acknowledgement, two accepts with the same clock value, a late request, a reused request ID with altered content, and a token scoped to another auction. Then run the same cases through multiple accepting processes backed by the production serialization mechanism. The invariant to assert is the committed `(acceptedAt, sequence)` order, not callback order in a test client.

Network tests should reorder and delay broadcasts independently for two dashboards. Both views must converge after snapshot recovery even if their animations differ. Disconnect one client across several accepted bids, reconnect it, and verify that a gap triggers recovery. Also test revocation and expiry on bid credentials separately from telemetry credentials.

Observability follows the same boundary. Record the request ID, authenticated subject, auction ID, acceptance timestamp, sequence, and decision reason in an append-only audit trail with access controls appropriate to bid data. Measure submit-to-accept and accept-to-display separately. Combining them into one latency number hides whether the authority or fan-out path needs work, and it tempts teams to move adjudication toward the client for prettier charts.

## Operational rule for closing day

Before traffic arrives, write the acceptance point and tie rule into the auction terms, verify that every writer uses the same durable serialization boundary, and confirm that server clock monitoring is alerting. Exercise idempotent retries and snapshot recovery in deployment, not only in unit tests. During the auction, watch rejected-scope counts, duplicate request IDs, sequence gaps, and the two latency stages independently. After close, freeze the accepted ledger and retain enough authenticated decision data to reproduce the ranking without browser logs.

**Server acceptance order is the verdict; delivery order is a view.** That distinction survives transport changes, slow carrier links, dashboard reconnects, and the addition of live vehicle telemetry. It also gives an operator a result that can be explained without trusting a bidder's clock or a bidder's screen.

## References

- https://www.w3.org/TR/webrtc/
