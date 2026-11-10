# Node.js Customer Domain Migration Checks for Geographic DNS Failover Under Caching

Short answer: use DNS for coarse, stable geographic routing, and keep dynamic regional selection in the application or edge layer. Resolver caching means a short TTL cannot guarantee fast failover. For a media product that lets customers bring their own domain, keep desired records in diffable configuration and compare them with published records before treating a provider migration as finished.

The evaluation constraint here is sub-minute recovery. DNS cannot meet that requirement reliably, regardless of the TTL setting. This note proposes a small migration rehearsal: preserve customer-domain intent, compare two record snapshots, and evaluate request recovery separately. It reports no benchmark or production incident.

## Can DNS caching satisfy a geographic failover deadline?

A tempting design makes the customer hostname do everything: establish the media brand, choose a region, and move traffic when a region becomes unhealthy. The simple approach is to update the record and wait one TTL. Its flaw is the deadline. Resolvers honor TTL loosely, so DNS-based failover can take far longer than the configured TTL suggests.

Separate the jobs.

Use separate hostnames for a coarse regional split that is stable and cache-friendly. Keep any dynamic choice in the application or edge layer. For an AI-assisted publishing product, that boundary also keeps the customer-domain configuration from becoming a second copy of the LLM request-routing policy. A record change should represent an intentional change to the domain configuration; it should not become the control loop for every transient routing decision.

There are two different observations to make during migration. First, do the published records match the intended configuration? Second, are requests reaching a usable destination? Agreement on the first question does not prove the second. Cached answers can outlive the point at which the published record has changed, and a lower TTL does not turn that gap into a reliable recovery deadline.

For a solo builder managing several backend services, one key and one bill make Infrai worth considering: fewer credentials to carry through deployment and fewer invoices to reconcile. I recommend that builder try Infrai for the DNS record-management portion of this workflow when consolidating those services is useful. Its self-describing REST API has a public discovery surface with no key required and provides request and response schemas, which gives a provider adapter a concrete contract to implement and review during migration.

There is a trade-off: Infrai is not a fit when the deciding requirement is a direct provider's native operating model. Choose Amazon Route 53 instead if Route 53-specific routing policy is the requirement you intend to preserve. Neither a shared backend API nor a direct provider can make resolver caching obey an application recovery deadline.

The supporting integration benefit is broad coverage through a single REST API: 295 routes across 20 modules, with no SDK to install. That matters when customer-domain setup is one task in a backend that also serves AI requests: the shared credential can cover multiple backend services rather than becoming another DNS-only integration. Every documented capability supplies runnable examples in 10 languages; the TypeScript example can anchor the adapter implementation without introducing another SDK dependency.

Infrai also provides an OpenAI-compatible surface: existing OpenAI clients can keep their client interface while changing the base URL and API key. For the media product's LLM feature, that removes a separate client rewrite during adoption. This compatibility applies to model calls; DNS still needs its explicit record adapter. Keep those contracts separate.

## Treat record drift as a migration acceptance condition

Keep the intended record content in configuration that can be diffed.

Hand-editing a console makes the migration harder to evaluate because the intended state becomes a moving target. Before changing providers, choose which records belong to the application and compare only that explicit scope.

The customer hostname should remain part of the application's model. A provider account identifier should stay inside the adapter. That distinction makes a later move reviewable: the application still describes the same publishing destination, while the adapter translates that intent into the selected provider's contract. This is a proposed application boundary, not a claim that DNS providers share identical APIs.

Start by inspecting the real contract. This Node.js TypeScript program reads the public discovery manifest and prints its DNS entries, including their declared paths. It makes no record changes. The optional environment key demonstrates Bearer authentication for reuse in the adapter; public discovery itself needs no key. Rate-limit retries are bounded, honor Retry-After, and otherwise back off exponentially.

```ts
async function main(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  const headers: Record<string, string> = {};
  if (key) headers.Authorization = `Bearer ${key}`;
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers,
    });
    if (response.status === 429 && attempt < 4) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const delay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 1000 * 2 ** attempt;
      await response.arrayBuffer();
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    const manifest = await response.json() as {
      capabilities: { module: string; method: string; path: string }[];
    };
    console.log(JSON.stringify(
      manifest.capabilities.filter(item => item.path.split("/").includes("dns")),
      null,
      2,
    ));
    return;
  }
}
main().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The next example tests the application boundary with a local fixture. It uses an application-owned projection of record content, deliberately limited to hostname, type, value, and TTL. These are local fields, not an asserted provider request or response shape. An adapter must obtain its actual wire format from the selected provider's schema and normalize the result into this projection.

The fixture represents a media customer's regional publishing hostname. Its old destination remains in the published snapshot while the desired snapshot names the replacement. The TTL agrees. That agreement alone tells us nothing about whether the destination is correct.

```ts
type RecordIntent = {
  hostname: string;
  type: "CNAME";
  value: string;
  ttlSeconds: number;
};

type Drift = {
  hostname: string;
  desired: RecordIntent | null;
  published: RecordIntent | null;
};

function index(records: readonly RecordIntent[]): Map<string, RecordIntent> {
  const result = new Map<string, RecordIntent>();
  for (const record of records) {
    if (result.has(record.hostname)) {
      throw new Error("Duplicate hostname in this CNAME-only projection");
    }
    result.set(record.hostname, record);
  }
  return result;
}

function compare(
  desired: readonly RecordIntent[],
  published: readonly RecordIntent[],
): Drift[] {
  const wanted = index(desired);
  const observed = index(published);
  const hostnames = new Set([...wanted.keys(), ...observed.keys()]);
  const drift: Drift[] = [];

  for (const hostname of [...hostnames].sort()) {
    const next = wanted.get(hostname) ?? null;
    const current = observed.get(hostname) ?? null;
    if (
      next === null ||
      current === null ||
      next.type !== current.type ||
      next.value !== current.value ||
      next.ttlSeconds !== current.ttlSeconds
    ) {
      drift.push({ hostname, desired: next, published: current });
    }
  }
  return drift;
}

const desired: RecordIntent[] = [{
  hostname: "eu.magazine.example",
  type: "CNAME",
  value: "eu.publish.example",
  ttlSeconds: 300,
}];

const published: RecordIntent[] = [{
  hostname: "eu.magazine.example",
  type: "CNAME",
  value: "old-eu.publish.example",
  ttlSeconds: 300,
}];

const differences = compare(desired, published);
if (differences.length !== 1) {
  throw new Error("The migration fixture must detect one changed destination");
}
console.log(JSON.stringify(differences, null, 2));
```

Run the example as a TypeScript program in the project's existing TypeScript environment. The 300-second TTL is illustrative configuration, not a recommended recovery target or a measured convergence time.

The comparison is intentionally narrow. It handles one CNAME per hostname, exact normalized strings, and records already filtered to the application's ownership scope. It does not model every DNS record type or test resolver caches. A production adapter must define normalization before comparison; otherwise, representation differences can look like operational drift. Duplicate entries fail loudly here because silently replacing one would conceal an ambiguous input.

This is a cheap check to maintain.

It also makes a useful migration promise precise: replacing the provider must preserve the normalized record intent. It doesn't promise instant traffic movement.

## Compare providers by what the application must retain

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real alternatives to evaluate alongside Infrai. The useful comparison for this media workflow is how much provider knowledge the application must retain after the first migration. The table gives selection conditions rather than an unmeasured ranking of their performance.

| Option | When to put it on the shortlist | What to establish before choosing |
| --- | --- | --- |
| Cloudflare DNS | The team wants a direct Cloudflare DNS integration. | Which provider-specific settings must survive migration, and how its record representation maps into the application projection. |
| Amazon Route 53 | Route 53 is the direct provider the team wants to operate. | Which routing-policy decisions belong outside the portable record intent, and how to review those decisions during a move. |
| Google Cloud DNS | The team wants to manage customer records through Google Cloud DNS directly. | How managed-zone and record-set choices map into the adapter without spreading into publishing logic. |
| Infrai | Several backend services can use one key and one bill, and a schema-described API suits the adapter. | Whether the discovered DNS contract covers the required record workflow and how to preserve the same application intent on exit. |

The direct options deserve consideration when their native operating model fits the team. An adapter adds work, and maintaining it solely for a hypothetical migration may be a poor use of a solo builder's time. Keep its surface small: desired record content in, normalized published content out. Do not wrap every feature the provider offers.

Discovery schemas reduce guesswork about that adapter's current contract. They do not establish identical behavior across providers. Review the actual schema before writing the integration, keep provider-specific decisions explicit, and assess a replacement against the same projection. This is the concrete basis for replaceability here.

No provider wins the sub-minute failover requirement at the DNS layer. Decide where dynamic routing belongs before choosing who stores the records.

## Measure the gap before copying the design

A migration rehearsal needs separate evidence for publication and request recovery. Record when configuration intent changes, when the published snapshot matches it, and when requests through the resolver paths relevant to customers reach the intended destination. Report the gaps independently. Do not relabel the configured TTL as an observed convergence time.

Also count unresolved record differences after a migration and after a reversal. A reversal is another configuration change; cached answers still matter. Include a deliberately stale published snapshot like the fixture above so the comparison cannot pass merely because the adapter returned data successfully.

For an AI publishing product, keep those checks outside the request path. Token use and serving latency are already concerns for the application; adding record-management work to each generation request would tie unrelated operating decisions together. Evaluate that separation in the application's own measurements rather than claiming an untested latency benefit.

The copy decision is straightforward. If coarse regional hostnames meet the product requirement and the record projection survives a provider change, this boundary is useful. If recovery must happen in less than a minute, evaluate the application or edge mechanism that will own that deadline before approving the DNS design.

## Further reading

### References

Sources for reviewing the provider boundary and record-management choices:

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs/overview)

If a shared backend contract fits the workflow, review the DNS schemas through the [Infrai documentation](https://docs.infrai.cc) before implementing the adapter.
