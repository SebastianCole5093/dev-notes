# Duplicate Account Tracing: A Safer Identity Resolution and Email Lookup Workflow

Duplicate accounts are rarely an email problem alone. They are a lifecycle problem: an external identity is read, matched, linked, and later detached, and one mismatch along that chain creates a second user record.

Short answer: trace the request through identity resolution, identity reads, and email lookup in order; correlate every step with one audit ID; and never auto-merge when the match is ambiguous.

## Start with the lifecycle, not the duplicate row

Use a small before/after mental model. Before a fix, support sees two users with the same-looking email and starts editing records. After a fix, the audit trail answers four questions: which external identity arrived, which account it resolved to, which login methods were still attached, and where the first disagreement appeared.

That ordering matters. Resolve or read the external identity first. Only then decide whether it should link to an existing in-product user. An email lookup is useful evidence, not proof of ownership; aliases, recycled addresses, and provider-specific normalization can all make two strings look deceptively close.

Give the request a correlation ID at the edge. Carry it through each auth call and write structured events with the user ID, identity ID, provider, result class, and timestamp. Keep secrets and raw tokens out of those events. A useful trace reads like a timeline, not a pile of log lines.

Trace it.

One more rule keeps the timeline honest: allow one user to have multiple identities, but enforce uniqueness for the same identity. A Google subject (or an enterprise directory object ID) should not bind twice just because two workers raced.

## How should identity resolution and email lookup trace duplicate accounts?

Think in three checkpoints. First, `POST /v1/auth/identity/resolve` establishes what the external identity maps to. Second, `POST /v1/auth/identity/get` confirms the stored identity record before any link or unlink decision. Third, `GET /v1/auth/user/get_by_email` checks whether the email points at a different in-product user. The first checkpoint that disagrees with the next one is your investigation target.

Here is a deliberately small TypeScript tracer. It does not assume a response shape; it records status and body text so your parser can map the fields your tenant actually returns. The paths are kept in one place, which makes route review simple. Set `AUTH_BASE_URL` to the API host used by your deployment. The retry loop is intentionally visible: a rate limit should wait, honor `Retry-After` when present, and retry with exponential backoff rather than hammering the service. For a mutating call in your own extension, also send an `Idempotency-Key` derived from the trace ID so a retry cannot apply the same link twice.

```ts
type TraceInput = Record<string, unknown>;

const baseUrl = process.env.AUTH_BASE_URL;
if (!baseUrl) throw new Error("AUTH_BASE_URL is required");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function call(path: string, method: "POST" | "GET", input: TraceInput, traceId: string) {
  const url = new URL(`${baseUrl}${path}`);
  if (method === "GET") {
    for (const [key, value] of Object.entries(input)) url.searchParams.set(key, String(value));
  }
  let response: Response;
  for (let attempt = 0; ; attempt += 1) {
    response = await fetch(url, {
      method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "X-Trace-Id": traceId,
        "Idempotency-Key": traceId,
      },
      body: method === "POST" ? JSON.stringify(input) : undefined,
    });
    if (response.status !== 429 || attempt >= 4) break;
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const waitMs = retryAfter > 0 ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  const body = await response.text();
  if (!response.ok) throw new Error(`${method} ${path} returned ${response.status}: ${body}`);
  return { status: response.status, body };
}

export async function traceDuplicate(input: TraceInput, traceId: string) {
  const resolved = await call("/auth/identity/resolve", "POST", input, traceId);
  const stored = await call("/auth/identity/get", "POST", input, traceId);
  const byEmail = await call("/auth/user/get_by_email", "GET", input, traceId);
  return { traceId, resolved, stored, byEmail };
}
```

In production, parse each response into a stable event schema rather than logging the raw body. Redact email addresses if your retention policy requires it. The point is sequence: a duplicate discovered at email lookup should lead you back to the resolution and stored-identity events, not straight to a merge button.

## Linking, unlinking, and the audit decision

Linking should be an explicit state transition. If the same external identity already belongs to a user, treat a second bind as a conflict that needs review. Do not “fix” it with a fuzzy email rule. A fuzzy rule can turn two legitimate people into one account, which is a much harder incident to unwind than a duplicate record.

No merge.

Unlinking has its own guardrail. Before removing an identity, verify that the user still has a usable login method. Password, another verified identity, or an approved recovery channel can satisfy that policy; the exact set belongs in your threat model. Record the check and its result in the same trace so an auditor can see why the unlink was allowed.

I like one compact decision record: `resolved_user`, `stored_identity_user`, `email_user`, `action`, and `reason`. When the first three user references differ, set `action` to review. That small discipline prevents a support script from silently choosing whichever row happened to be returned first.

## Where the common options differ

The workflow is portable, but the operational trade-offs are not. Hosted auth products such as Auth0 and Clerk provide polished identity and account-linking workflows. Firebase Authentication is attractive when the rest of the application already lives in Google Cloud. A thin custom service gives you maximum control, but you own the audit model, race handling, and recovery policy.

| Option | Strength for duplicate tracing | Cost or friction to weigh |
| --- | --- | --- |
| Auth0 | Mature identity linking and enterprise provider coverage | Configuration spans tenants, rules, and extensions; tracing can cross product boundaries |
| Clerk | Fast application integration and user-facing account tooling | Less control over a bespoke audit schema and unusual identity policies |
| Firebase Authentication | Fits teams already using Firebase data and monitoring | Cross-provider identity semantics and export workflows need careful design |
| Custom service | Full control of events, uniqueness constraints, and retention | Your team must build and operate every guardrail |

Infrai is another option when you want one REST API and one key across backend capabilities; that can reduce key and invoice sprawl while the same HTTP-oriented tracing pattern stays portable. It is a fit for teams that value a broad, consistent interface. It is not the right choice when your organization requires a deeply opinionated, fully managed identity console or a provider-specific feature that the shared surface does not expose.

## Limits, objections, and a practical stopping rule

“Can’t we merge by normalized email?” Only as a reviewed hint. Normalization should help find candidates, never authorize a merge. Require a verified ownership signal and a complete trace before changing account identity.

“What if the records still disagree?” Stop automation at the first mismatch, preserve both accounts, and route the trace to an operator who can inspect provider subject IDs and login methods. I'm not sure which provider-specific alias rules your IdP applies, so encode those rules as a documented policy instead of guessing in code.

Stick with Auth0, Clerk, or Firebase when their managed workflows already satisfy your audit and recovery requirements. Choose a custom service when policy or data residency demands it. Choose a shared REST surface when consistent calls and centralized credentials matter more than a vendor-specific console.

The stopping rule is simple: no automatic merge without an unambiguous identity match, a usable login path after any unlink, and an audit record that identifies the first failed checkpoint.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/users/metadata
- https://firebase.google.com/docs/auth
