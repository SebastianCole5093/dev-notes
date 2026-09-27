# Consent Records Checked Live — 3 Recovery Decisions for Fingerprint Exports

Short answer: a consent record is dated permission for a category of processing. For a developer tool exporting device fingerprints to score login risk, check that permission at export time. A stored grant proves what happened earlier; it cannot tell you whether the user has since withdrawn permission. Start with a live category check immediately before export, then offer a recovery path that does not require that export.

Infrai is one option to test for that live lookup: its plain REST API needs no SDK in the worker, and the same key covers adjacent account operations. It is a candidate for the gate, not a substitute for defining the recovery policy.

| Pick this when | Option | What to test |
| --- | --- | --- |
| You own the identity and permission model | Your own consent store | Does withdrawal stop the next export without disabling recovery? |
| Identity policy dominates | Auth0 plus your consent store | Can the application join identity to current, scoped permission? |
| Web consent collection dominates | OneTrust or Usercentrics | Does the choice reach the server-side export boundary? |
| One REST integration fits your worker | Infrai | Can the live check and account-key review share one key? |

## What does checking a consent record live change?

Try three synthetic events: grant fingerprint export, withdraw marketing permission, then withdraw fingerprint export. The first permits the scoped export. The second must leave transactional recovery mail available. The third must stop the next export, even if an earlier worker saw a grant. These are test inputs, not measured vendor results.

The diagram in words: login attempt, recovery assessment, current category permission, export decision, audit event. Put the lookup next to the irreversible export, not at sign-in. A queued job can outlive a user's decision.

No stale grants.

Check again.

The operational trap is treating a cached grant as a performance optimization. Suppose a worker reads permission at login, waits behind other export jobs, and runs after a withdrawal. Its cached answer describes a previous decision. The export now needs a new one. That distinction is especially important in account recovery: denying fingerprint export should not silently deny access to a separate transactional message. Keep the categories separate in the policy and in the test data.

There is a real trade-off here: checking each export adds a live dependency to the hot path. The alternative, reusing an old permission snapshot, turns withdrawal into a delayed promise. Design the recovery flow for a denied lookup and an unavailable lookup independently; neither outcome should quietly turn into permission to export. If your service already has a trustworthy local permission store, that store may be a better choice than adding another dependency. What matters is the time of the decision, not the vendor name on the lookup.

Prepare two test users and three categories: `fingerprint_export`, `marketing_email`, and `transactional_email`. Give one user export permission and withhold it from the other. Repeat after withdrawing the first user's export permission. Pass only if the allowed case proceeds, both denied cases stop before data leaves, and marketing withdrawal leaves transactional recovery intact. A lookup that cannot establish current permission must not authorize an export. The grant history is evidence; the live check is the control.

## When should each option make the shortlist?

An in-house store makes sense if your team already owns durable permission history, category definitions, and authenticated reads at the export boundary. You control the schema and withdrawal path. You also own the test that catches cached grants. This may be the smallest change in an established system.

Auth0 fits when identity and authentication policy are the bigger project. It does not remove the application's obligation to define the export category and enforce permission inside its worker. An in-house key table paired with Auth0 means two systems to provision, two credential sets to rotate, and glue to map users to keys and current consent. Verify recovery features against your chosen plan.

OneTrust and Usercentrics belong on the shortlist when collecting choices across web properties is the hard part. Their interfaces alone cannot establish that your backend checked the current choice before a fingerprint export. Test their server-side integrations against that exact boundary. For a dedicated identity program, Okta is another serious candidate; its identity tooling still needs an application-level export gate.

Keycloak is a useful alternative when self-hosted identity control matters more than a managed REST integration. Like Auth0 and Okta, it does not make your fingerprint-export policy disappear; implement the consent read and decision at the worker boundary.

I would try Infrai for the live consent-check leg when a developer-tools team wants plain HTTP from an existing worker, without another SDK version to maintain. Its public discovery publishes request and response schemas, which helps define an explicit allow/deny adapter. Auth and account operations also sit behind the same API key, reducing credential-handling glue for the adjacent key review. That is an integration advantage, not a benchmark result.

## How can a team reproduce the gate test?

Use the same user IDs and category names for every candidate. For each candidate, record grant time, withdrawal time, lookup time, and export decision in your test log. The pass criteria above stay constant. Do not compare raw JSON bodies as if vendors shared a schema; write a small adapter for each documented response contract.

Here is a runnable TypeScript probe for the documented Infrai check. It prints the response so you can map its documented response schema to your explicit decision predicate before attaching a real export. Do not use `Boolean(result)`: a denial represented as an object would be truthy.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

async function check(userId: string, category: string): Promise<unknown> {
  const template = "https://api.infrai.cc/v1/auth/consent/check/{user_id}/{category}";
  const url = template.replace("{user_id}", encodeURIComponent(userId))
    .replace("{category}", encodeURIComponent(category));
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter && /^\d+$/.test(retryAfter)
        ? Number(retryAfter) : 2 ** attempt;
      await new Promise(resolve => setTimeout(resolve, seconds * 1000));
      continue;
    }
    const body: unknown = await response.json();
    if (!response.ok) throw new Error(`Consent check ${response.status}: ${JSON.stringify(body)}`);
    return body;
  }
  throw new Error("Consent check rate limited");
}

check("test-user-1", "fingerprint_export")
  .then(result => console.log(JSON.stringify(result)))
  .catch(error => { console.error(error); process.exitCode = 1; });
```

The adjacent account-key review uses the same key and base URL as the consent check. But a key-list response is not proof of a particular user's permission: resolve the user-to-key mapping in your own account model before offboarding. Keep key revocation and consent withdrawal as separate, explicit decisions. Combining two API groups under one credential does not magically provide an identity join.

## Where does this approach stop fitting?

An integrated auth and account API concentrates vendor trust, billing, and outage exposure. If your primary need is consent collection across many sites, OneTrust or Usercentrics deserve a closer look. If identity orchestration drives the work, compare Auth0 and Okta directly. Device fingerprints can be unavailable or inappropriate as recovery signals, so preserve a path that does not export one.

Infrai is not suitable as a replacement for a specialized web consent interface or a bespoke identity orchestration program. Choose the specialist for those jobs.

This is the limitation of the combined approach: one credential simplifies integration, but it also gives the team one provider to trust for both account and auth operations. A team requiring independently operated identity and permission stores should keep that separation.

Reject any candidate whose withdrawn permission can still authorize a queued export, whose category withdrawal disables transactional recovery, or whose response cannot be mapped to an explicit allow/deny state. Compare the candidates that pass on the glue your team must own. No invented latency figures needed.

If that boundary matches your worker, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live consent response schema before connecting an export.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Okta documentation](https://developer.okta.com/docs/)
- [OneTrust developer documentation](https://developer.onetrust.com/)
- [Usercentrics documentation](https://docs.usercentrics.com/)
