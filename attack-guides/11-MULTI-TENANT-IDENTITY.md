# 11 — Multi-Tenant Identity Isolation

**Source:** `Phases checklist.txt` Phase 11 "Multi-Tenant Identity Isolation Mastery" (read in full, lines 12343–13955 — 14 BB tests, 8 GB areas, 10 WB tracing areas, 9 differential dimensions), `Testing Approach .txt` (folded into its authorization phase), and the course's Phase 11.

**The invariant governing this entire guide:** a tenant ID supplied by the client is not, by itself, proof that the client belongs to that tenant. A request can be perfectly authenticated and still be unauthorized. Authentication establishes who you are; tenant isolation determines which organization's resources you may access — a system can authenticate users correctly and still expose one customer's data to another.

## A. Security objective

For every request touching a tenant-owned resource: the caller's identity must be established correctly, their tenant membership established from trusted server-side information (never a client-supplied value alone), their role checked within the *target* tenant specifically, the requested resource confirmed to belong to that tenant, and the full chain preserved through every side effect the operation triggers.

## B. Attack surface

Look well beyond a field literally named `tenant_id` — boundaries show up as `organization_id`, `workspace_id`, `account_id`, `team_id`, `company_id`, `project_id`, `customer_id`, and similar, sometimes nested (a document belongs to a project, which belongs to a workspace, which belongs to an organization). Cover every operation per resource type, not just detail-view: create, read, update, delete, list, search, export, download, share, invite, transfer ownership, change permissions, generate credentials, trigger actions, view audit history. Tenant leaks concentrate disproportionately in list, search, export, invitation, reporting, and background-job paths — not the detail endpoint everyone tests first.

## C. Threat model

The realistic attacker is simply an authenticated user of Tenant A probing whether anything — a direct object reference, a list query, a signed URL, a background job's output, a cached response — returns or modifies something belonging to Tenant B. No special capability required beyond an ordinary account; this is purely a server-side enforcement question.

## D. Black-box testing

**Baseline first, always.** Prove each controlled identity works correctly *within its own tenant* before interpreting any cross-tenant denial — a denial against Tenant B means nothing if User A1 can't even reliably access their own Tenant A resources.

**Cross-tenant direct object access.** Request a known Tenant B resource using a Tenant A session. Record separately whether the response discloses mere *existence* (a 403 vs. 404 distinction, not inherently a finding either way) versus the actual protected content, metadata, or an editable representation — these are different severities.

**Cross-tenant modification and destructive actions.** Never trust a `200 OK` as proof anything actually changed. Re-read the resource using its authorized owner afterward and confirm whether the intended field, or any unrelated field, actually moved. Also check whether a denied request still triggered a side effect *before* the denial was returned — a `403` response is not proof nothing happened.

**List and search isolation** — this is the single most commonly missed category. A detail endpoint can be perfectly scoped while list, search, autocomplete, pagination, sort, and aggregate-count endpoints leak across tenants. Even a dashboard that correctly shows only Tenant A's projects but reports a *total count* including other tenants' records is a real information-disclosure finding.

**Client-controlled tenant identifiers** (path, query param, JSON body, header, cookie, JWT claim, GraphQL variable, WebSocket message). Their mere presence is not a flaw — the proof requires all three: the authenticated identity's actual tenant, the target resource's actual owning tenant, and the application granting access despite no valid relationship between them.

**Multi-tenant membership and role confusion.** `admin` is not necessarily global — test whether a user who's an administrator in Tenant A is incorrectly treated as one in Tenant B too, whether role evaluation actually happens *within the target tenant* rather than reusing whatever role was valid elsewhere, and whether revoking a membership actually invalidates access to that tenant.

**Tenant switching and stale context.** Switch active organization, then check whether API responses, cached dashboard data, or a previously-elevated role persist past the switch. A visual UI mismatch is a lead, not a finding — confirm actual unauthorized data access or a genuine policy violation before reporting.

**Creation and ownership assignment.** A system can correctly protect reads/updates while mishandling creation — test whether a user can create a resource under a tenant they don't belong to, or assign tenant ownership through a client-controlled field at creation time. Verify the resulting stored ownership with an authorized account afterward, not just the creation response.

**Invitations and membership lifecycle.** Who can issue an invitation, which organization does it target, can an unauthorized caller redirect the target organization, is the invitation bound to the intended org, does revocation actually take effect.

**File storage, downloads, generated artifacts.** A protected API can still leak tenant data through an unprotected file URL or storage object. For signed URLs specifically: who can generate them, which object do they authorize, how long are they valid, can they be shared or reused — the absence of a login requirement on a signed URL is not automatically broken isolation, evaluate the actual scope.

**API/GraphQL/WebSocket/alternate-route consistency.** A protected top-level GraphQL object doesn't guarantee every nested relationship, resolver, or bulk/aggregation operation is equally scoped. For WebSockets specifically: is authorization re-validated when membership or role changes during a long-lived connection, or does a subscription keep an earlier authorization state indefinitely.

**Cache isolation.** Request the same tenant-specific resource as two different tenants and check whether a shared cache key returns one tenant's private response to the other. Different users legitimately receiving the same response (public content) is not a finding — confirm the content is actually private and tenant-specific first.

**Exports, reports, analytics, background jobs** — frequently the highest-impact category because it's outside ordinary request-response flow. Is the tenant checked at job submission *and again* when the worker actually processes it; can the caller request another tenant's job by ID; can a retry or scheduled task run with stale tenant context; do webhook deliveries or notifications ever include another tenant's data.

## E. Gray-box testing

Map the actual identity-to-tenant-to-role-to-resource relationship model: can a user belong to multiple tenants, is role stored globally or per-membership, can resources be explicitly shared across organizations, are service accounts tied to a specific tenant. Trace tenant-context propagation and identify which sources are actually *trusted* — if a token identifies User A1 while a header separately claims Tenant B, does the application blindly honor the header? Review database schemas where authorized: a `tenant_id` column existing doesn't establish isolation by itself — isolation depends on whether queries and authorization logic actually use it, and conversely a *missing* tenant column isn't automatically a vulnerability if ownership is derived correctly through valid parent relationships instead. Review logs for actor, effective tenant, target resource, and authorization outcome — specifically hunting for a mismatch between the tenant *requested* and the tenant actually *authorized*.

## F. White-box testing

**Trace the full path**: `request → routing → auth middleware → identity resolution → tenant-context resolution → membership/role lookup → resource authorization → query/service call → serialization → side effects`. At every stage: where does tenant context actually come from, is that source trusted, does the policy apply to the actual target resource, can a later function bypass the earlier check, can a side effect fire before a denial is returned.

**Trace the data-access layer — the single most important pattern in this guide:**

```python
# Unsafe:
project = db.query(Project).filter(Project.id == project_id).first()

# Safer:
project = db.query(Project).filter(
    Project.id == project_id,
    Project.tenant_id == authorized_tenant_id
).first()
```

The security requirement isn't a specific ORM syntax — it's that the resource lookup itself is scoped to the caller's *validated* tenant, not fetched first and trusted by ID alone. Relationship-based models (shared resources, parent-child ownership) need the equivalent check at whatever level ownership is actually established.

**Trace membership validation**: is it checked against trusted server-side state; are roles evaluated *in the correct tenant context specifically*; are revoked memberships actually treated as inactive, not just flagged; does the application distinguish tenant administrators from platform administrators.

**Inventory middleware coverage** across every route — missing tenant authorization on specific routes, GET vs. mutation methods enforced inconsistently, alternate API versions, bulk endpoints that skip per-resource checks, internal service routes accidentally exposed to untrusted callers. A shared middleware component existing doesn't prove complete coverage — verify the specific route you're testing actually uses it.

**Trace nested-resource relationships.** `Tenant → Project → Document → Attachment`: does the document endpoint verify it actually belongs to the requested project, does the project belong to the authorized tenant, can a valid document ID be paired with an unrelated project ID to skip the check entirely.

**Trace mass-assignment paths** for `tenant_id`, `organization_id`, `owner_id`, `role`, `is_admin`, `visibility`, `billing_account_id` — can a user modify protected ownership/privilege fields through a normal update endpoint that wasn't meant to expose them.

**Trace async jobs and service identities.** Workers often run with broader technical privileges than ordinary users, which makes explicit tenant/resource validation at processing time especially important — not just at submission. Can a job ID retrieve another tenant's output; do retries preserve or lose correct tenant context.

**Trace error handling and serialization** for leakage even when the primary query is correctly scoped — nested object serialization, ORM relationship auto-loading, verbose error messages, GraphQL error extensions can all expose protected data through a denied operation's *response details* even when the main data field is empty.

## G. Differential testing

Nine dimensions, change one variable at a time: identity (same resource, different caller), tenant (same caller, different target), role (same tenant/resource, different privilege level), tenant-switch (same identity, before/after switching active org), interface (web/REST/GraphQL/mobile/export — same intended operation), request-path (detail vs. list vs. search vs. nested vs. export endpoint for the same underlying resource), state (before/after membership revocation, role downgrade, session refresh), request-order (access A, then B, then back to A — checking for context bleed), and side-effect (compare the resource's actual state before/after a request expected to be *denied* — a denial response doesn't matter if the operation already fired).

## H. Exploitation and validation

Minimal proof for any cross-tenant finding: demonstrate the actual protected content or state change, not just a response code, using two controlled accounts with a known, verified membership model between them. The strongest finding correlates all three evidence types — black-box (the request/response), gray-box (membership records confirming no legitimate relationship existed), and white-box (the specific unscoped query or missing check) — but a reproducible black-box finding alone can already be conclusive; don't withhold reporting just because gray-box/white-box access isn't available, state clearly which parts are externally demonstrated versus unconfirmed.

## I. False-positive analysis

The user may legitimately belong to both tenants — check the actual membership and role in each before calling cross-access unauthorized. The resource may be intentionally shared — verify the sharing policy and authorized recipients. A `200 OK` may contain no actual protected data — verify returned content. A `403` may arrive *after* a side effect already occurred — check resulting state and logs, not just the response. Different status codes (`403` vs `404`) may be a deliberate existence-concealment choice, not a bug. A code path that looks unprotected may be covered by a separate authorization layer — trace the complete call path before concluding. A background job running under a broad-privilege service account isn't automatically wrong — verify the worker still enforces tenant/resource checks despite its technical access level.

## J. Root cause

The dominant pattern across nearly every finding class here: a resource fetched by ID first, with tenant/ownership validation applied afterward (or never) instead of being baked directly into the lookup query itself. The second most common: a control implemented correctly on the primary detail/read endpoint but never ported to list, search, export, background-job, or alternate-API-version paths reaching the same underlying data.

## K. Remediation

- Enforce tenant authorization server-side, never relying on a client-selected or client-supplied tenant identifier as proof of membership.
- Scope data-access queries to the caller's validated tenant directly in the query itself, not via a separate check applied to an already-fetched object.
- Apply the same policy to every operation type (read, write, delete, search, list, export, download, invite, admin) and every interface (web, REST, GraphQL, WebSocket, mobile, legacy) reaching the resource.
- Evaluate role within the specific target tenant, never assume a role transfers across organizational boundaries.
- Validate parent-child relationships explicitly for nested resources — a valid child ID paired with an unrelated parent ID must fail.
- Bind background jobs and their artifacts to validated tenant context at both submission and processing time.
- Design cache keys and invalidation to respect the authorization boundary, not just resource identity.

## L. Regression testing

Add automated tests specifically for cross-tenant reads, cross-tenant mutations, membership revocation, tenant switching, search/pagination/export paths, nested resources, background jobs, and cache isolation — for every fixed finding, a regression test that fails against the old behavior and passes against the fix.

## M. Finding documentation

Title, affected endpoint/operation, test identity (role and authorized tenant, no secrets), target resource (type, ID, owning tenant), expected vs. observed behavior, evidence across whichever of black/gray/white-box layers were available, impact (what could actually be read/modified/deleted/triggered), root cause, reproduction conditions, remediation, regression test, severity justified by demonstrated impact — not by the finding's category name alone (a cross-tenant read of public metadata and a cross-tenant export of an entire customer's billing data are both "tenant isolation failures" but not remotely the same severity).

## Completion criteria

- [ ] Baseline established for each controlled identity within its own tenant before testing cross-tenant access
- [ ] Direct object access, modification, and destructive actions tested with resulting state verified independently of response codes
- [ ] List, search, pagination, and aggregate/count endpoints specifically tested, not just detail endpoints
- [ ] Multi-tenant membership tested for role evaluation within the correct target tenant
- [ ] Tenant switching tested for stale cached context or retained privilege
- [ ] File downloads and signed URLs tested for scope and authorization, not just presence of a login requirement
- [ ] Every alternate interface (GraphQL, WebSocket, mobile, legacy API) compared against the primary one for policy consistency
- [ ] Background jobs tested for tenant validation at both submission and processing time
- [ ] Each finding traced to the specific unscoped query or missing check, not reported as a generic "tenant isolation bypass"
