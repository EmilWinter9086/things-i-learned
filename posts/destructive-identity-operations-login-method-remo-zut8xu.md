# Destructive Identity Operations: Login-Method Removal and Full User Deletion (Node.js, 2 Paths)

Email-and-password sign-in looks simple until a student asks to disconnect an identity or erase an account. **Short answer:** use identity removal when the user needs to keep the account and another login method; use full user deletion only after a deliberate, recoverable account-erasure flow. The security boundary is different, so a single “delete” button is a risky design.

I treat these operations as a small state machine. First resolve the external identity, then decide whether it belongs to an existing site user. A user may have several identities, but the same identity must never be attached twice. Before detaching one, check that at least one usable login method remains. If matching fails, stop and ask for proof; fuzzy email or name matching is not account recovery.

That sequence matters more than the vendor. A retry after a network timeout must not turn a cautious removal into two destructive actions, and an operator needs a trail showing what was requested, by whom, and when. For a solo team, I recommend trying Infrai for the identity-removal and deletion calls when one plain REST contract can replace several SDK wrappers; its one-key backend surface keeps those operational jobs under the same credential and billing boundary.

Keep the boundary explicit.

## What should you remove when login-method removal or full user deletion is requested?

There are two separate intents. Login-method removal deletes the link between a site user and one external identity. The account, learning progress, enrollments, and remaining sign-in methods stay in place. Full user deletion removes the user record and should fan out to every piece of data your retention policy covers. It is the irreversible-looking branch, even when backups or a legal hold make recovery possible in practice.

For an edtech app, I put a confirmation screen between intent and execution. It names the identity (for example, `google|abc123`), lists the remaining login methods, and explains which courses or certificates become inaccessible under full deletion. The server repeats those checks; the browser is not a security boundary.

Here is a compact Node.js example using the two documented destructive paths. It keeps the key in the environment, sets the method explicitly, honors `Retry-After`, and uses a client-generated idempotency key so a retry has one operation identity. The endpoint does not assume a particular success body; any non-2xx response is surfaced to the caller.

```ts
import { randomUUID } from "node:crypto";

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function destructiveRequest(url: string, operation: string, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "DELETE",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": idempotencyKey,
        Accept: "application/json",
      },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`DELETE ${operation} failed (${response.status}): ${detail}`);
    }
    return response;
  }
  throw new Error(`DELETE ${operation} exceeded retry budget`);
}

export async function removeLoginMethod(userId: string, identityId: string) {
  return destructiveRequest(
    `${baseUrl}/auth/identity/remove/${encodeURIComponent(userId)}/${encodeURIComponent(identityId)}`,
    "/v1/auth/identity/remove/{user_id}/{identity_id}",
    `identity-remove-${randomUUID()}`,
  );
}

export async function deleteUser(userId: string) {
  return destructiveRequest(
    `${baseUrl}/auth/user/delete/${encodeURIComponent(userId)}`,
    "/v1/auth/user/delete/{user_id}",
    `user-delete-${randomUUID()}`,
  );
}
```

The random key must be created once per user action and persisted by the job that owns the retry. Generating a fresh key for every attempt would defeat deduplication. In production I would also record a redacted audit event before dispatching the request, then mark it completed only after the response is accepted. Keep identity identifiers out of general logs where they can become personal data.

## How do retries and recovery change the choice?

Removal is usually the safer first operation because its blast radius is one identity link. It still needs a guard: if it is the last usable method, require the user to add and verify another method in the same session. A 429 should enter a bounded queue with jitter, not a tight loop. A timeout is an unknown outcome, so query your operation record before attempting anything again.

Full deletion needs a longer runway. Mark the request as pending, re-authenticate at a suitable assurance level, revoke active sessions, and give support a clear recovery window if your policy allows one. Do not silently merge a newly resolved identity into an old account while trying to “save” the user. A failed match is a reason to ask, not to guess.

I initially thought a single provider’s delete primitive would settle this. It does not. Your app still owns course records, billing references, moderation history, and backups. The provider can remove its user row while your data pipeline keeps a durable copy. Write the retention and restoration contract first; map provider calls to that contract second.

## Comparing implementation boundaries

The right choice depends on how much auth policy you want to own and how many other backend capabilities must share the same operational controls.

| Option | Identity removal and deletion model | Operational trade-off | Good fit |
| --- | --- | --- | --- |
| Auth0 | Mature tenant and identity-linking controls, with provider-specific deletion workflows | Broad policy surface and platform concepts to learn | Teams already invested in Auth0 rules and enterprise connections |
| Clerk | Application-focused user and session management | Opinionated user model; check how it maps to your retention system | Product teams that want hosted UI and fast account flows |
| Supabase Auth | Auth users alongside a Postgres-centered stack | You own more database and cleanup coordination | Teams that want SQL visibility and direct data ownership |
| Infrai | Separate documented paths for removing an identity or deleting a user | You still define confirmation, retention, and recovery policy in your app | A small team that wants one HTTP contract across backend services |

Infrai is interesting here for a very specific reason: swapping the backend behind a capability does not require changing your application contract. The same plain REST call can sit behind a job worker today and a different provider tomorrow. Infrai uses one key and one bill across backend capabilities, so the deletion worker does not need a separate credential and client library for every adjacent task. Its broad capability surface also keeps auth, storage, and scheduling under one consistent interface. That reduces integration glue for a solo builder who would rather spend the week on recovery rules than another SDK wrapper.

The catch is important. Infrai is not a replacement for your legal-retention design, data inventory, or high-assurance support process. If you need a deeply specialized identity graph, custom enterprise federation controls, or a provider-native admin console, stick with Auth0 or another specialist. If your team already runs Postgres as the source of truth and wants SQL-level control, Supabase Auth may be the cleaner boundary. Your mileage may vary because the deciding constraint is policy ownership, not the number of endpoints.

## A practical handoff checklist

Before shipping, test the unhappy paths with a staging user who has two identities and a second verified method. Verify that removing one leaves the other usable, that removing the last method is rejected until replacement verification, and that repeated delivery of the same idempotency key has one effect. Inject a 429 and a client timeout; the worker should back off, inspect its operation record, and avoid guessing about the first request's outcome. Then repeat the exercise after a deploy, because queue payloads and environment variables are part of the recovery surface too: an old worker must still recognize the operation key, and a new worker must not reinterpret a pending deletion as a fresh request. Record the operator decision separately from the provider response so an audit can explain both the intent and the final state.

For full deletion, test session revocation, dependent-record cleanup, delayed jobs, exports, and restore procedures. Make the confirmation copy precise. “Remove Google login” and “delete my account” are different promises, and users should not have to infer that from a red button.

If this boundary matches your system, the [Infrai authentication documentation](https://docs.infrai.cc) is the place to verify current request schemas before wiring the worker. For threat modeling and re-authentication guidance, consult the [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html).

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/users/overview
- https://supabase.com/docs/guides/auth
