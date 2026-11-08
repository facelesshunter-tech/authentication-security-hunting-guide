# 16 — Authentication-Adjacent Authorization Flaws

**Source:** `Phases checklist.txt` Phase 17 "Authentication-Adjacent Authorization Flaws" (read in full, lines 21505–22674, the final phase of the file — 9 BB tests, 5 GB areas, 8 WB tracing areas, plus a complete worked case study), `Testing Approach .txt` Phase 14, and the course's Phase 17.

Authentication answers *who are you*. Authorization answers *what are you allowed to access or do*. An application can authenticate a user perfectly and still have a critical flaw because it never actually checked the second question. This is BOLA's defining characteristic: the identifier doesn't need to be predictable, doesn't need to be guessed — the issue is purely that the server failed to enforce the access policy once it had a valid identity in hand.

## A. Security objective

Five boundaries must each hold independently for every sensitive operation: identity (can one user act as another), ownership (can a user reach another user's object), role (can an ordinary user invoke an admin-only action), tenant (can one organization reach another's resources), and privilege (can a user exceed their assigned authority). A single application can fail some of these while correctly enforcing others — test each one deliberately, never assume passing one implies the rest hold.

## B. Attack surface

Profiles/account settings, orders/invoices/payments/refunds, documents/reports/downloads, user and role management, tenant/organization administration, API keys and integration settings, uploads and file retrieval, support tickets, exports and background jobs, administrative/internal APIs, and GraphQL queries/mutations where applicable.

## C. Vulnerability taxonomy

BOLA/IDOR (object ownership not enforced), Broken Function-Level Authorization/BFLA (restricted operation reachable by an unauthorized role), privilege escalation (gaining authority beyond assigned permissions — horizontal: same level, different identity; vertical: a genuinely higher level), tenant isolation failure, mass assignment (client-controlled fields altering protected properties), authorization bypass through alternate routes (one endpoint enforces policy, an equivalent one doesn't), workflow authorization failure (steps reachable out of valid sequence), property-level authorization failure (some fields writable or readable when they shouldn't be), delegated-access failure (sharing/revocation rules wrong), and confused-deputy flaws (a trusted service acts with its own authority on behalf of an insufficiently authorized caller). **Classify by the actual security boundary violated, not by which endpoint you happened to find it on — a single issue can be both BOLA and a tenant-isolation failure at once.**

## D. Threat model

The defining feature of this entire guide: the attacker needs no special capability beyond a *valid, ordinary authenticated session* of their own. No credential theft, no cryptographic attack, no social engineering — every test here is "I am who I claim to be; the question is whether the server checks what I'm allowed to touch."

## E. Black-box testing

**Build the authorization attack surface first** as a structured matrix, not an unstructured endpoint list: `identity → resource → action → expected decision` (e.g. "User A → User B's invoice → Read → Deny"). This discipline matters because it forces you to state the expected policy *before* you observe behavior, which is what separates a defensible finding from a post-hoc rationalization.

**BOLA/IDOR.** Baseline: User A reads their own resource. Differential: identical request, identical session, only the resource identifier changes to one owned by User B. Interpret carefully — `ORD-B` returning B's private data is a potential confirmed finding; both requests returning `404` needs investigation into whether the resource exists at all; the UI blocking B's order while the raw API returns it anyway is itself a distinct, serious finding (server-side enforcement entirely missing, with only client-side cosmetics hiding it). The finding is real only when all four hold: A is authenticated as A, the target genuinely belongs to B or is outside A's scope, the server actually discloses protected information or performs the action, and no legitimate sharing/admin policy explains the access.

**Broken Function-Level Authorization.** Establish how a privileged operation behaves for an authorized administrator, then make the *identical* request as an ordinary user and as an anonymous visitor. A hidden UI button is not a security control, and a client-side route guard is not a substitute for server-side authorization — test the raw endpoint directly regardless of what the UI exposes.

**Horizontal vs. vertical escalation — test both independently.** A system can enforce role checks correctly while completely failing ownership checks, or vice versa; passing one test tells you nothing about the other.

**Tenant isolation.** Using accounts from two distinct tenants, test whether Tenant A's session can reach Tenant B's resource, and specifically check every place tenant context could be supplied: path parameters, query parameters, body fields, headers, GraphQL arguments, export/background-job parameters. A client-supplied tenant ID existing in the request is not itself a vulnerability — the server must validate the caller's actual membership against it. (Cross-reference Guide 11 for the full depth on this topic; this is the authorization-classification angle specifically.)

**Mass assignment and protected properties.** The question is never whether the API accepts extra JSON fields — it's whether a user can change a property they're not authorized to control. Check the *persisted server-side state* after the request, not just the response body, for `role`, account status, tenant ownership, verification status, billing level, approval flags, and ownership identifiers specifically.

**Alternate routes and HTTP methods.** Test the same business operation across every route that reaches it — web UI, REST, mobile API, legacy API version, GraphQL mutation, bulk operation, export/download endpoint — for consistent policy enforcement. An unusual HTTP response alone doesn't prove a bypass; confirm whether the protected operation actually succeeded.

**Workflow and state-dependent authorization.** For any multi-step business process (a refund needing payment existence, refund eligibility, requester permission, amount limits, and approval), test whether a caller can approve their own initiated action when separation of duties is required, skip a required approval stage, modify a resource after approval, repeat a completed operation, or perform a restricted action after their permission was just revoked.

**Authorization after permission changes.** Downgrade a role, remove tenant membership, revoke a sharing permission, disable an account — then check whether *subsequent* requests actually respect the new state, considering cached permissions, long-lived tokens, background jobs, and previously generated download links as distinct places the old permission could persist. Some systems have intentionally short but nonzero propagation windows — compare against the documented requirement before calling persistence a finding.

## F. Gray-box testing

Compare documented policy (API specs, role/permission matrices, tenant membership rules, sharing/delegation requirements) directly against black-box results — a documented restriction the server fails to enforce is much stronger evidence than an undocumented route simply existing. Review authorization configuration where available: is the policy based on the authenticated principal or a client-controlled value, are object-level checks applied *after* the resource is identified (not just role checks before), do privileged routes carry additional controls beyond ordinary authentication. Examine audit logs to corroborate: which principal, which role/tenant, which resource, was a decision recorded, was it allow or deny, did the state actually change. Inspect controlled resource/permission records directly to distinguish a genuine authorization failure from a mistaken assumption about ownership or intended sharing. Investigate cached authorization decisions specifically — compare access immediately before and after a controlled permission change, and determine what's cached (roles, tenant membership, permission lists, object decisions, signed URLs, precomputed exports), how long it persists, and whether the server actually re-checks on retrieval or just trusts the cache.

## G. White-box testing

**Trace the full execution path**: `router → auth middleware → principal construction → resource lookup → authorization policy → business logic → data access → response`. **The single most important discipline in this entire guide: authentication middleware establishing who the caller is proves nothing about whether that caller can access a particular object.** Never treat the presence of auth middleware as proof of authorization.

**Find missing object-level checks — the exact vulnerable pattern:**

```python
@app.get("/api/orders/{order_id}")
def get_order(order_id: str, current_user=Depends(get_current_user)):
    order = repository.get_order(order_id)
    if order is None:
        raise HTTPException(status_code=404)
    return order
```

This authenticates and loads the resource, but never visibly verifies ownership — a high-priority review target, not proof by itself (the check might live in the repository or a downstream policy). The stronger design scopes the lookup directly to the caller's authorized data (`repository.get_order_visible_to(user_id=..., order_id=...)`), which reduces the chance of accidentally returning a resource that was never checked — but the repository function must actually enforce the policy; its name proves nothing on its own.

**Review centralized authorization helpers** (`is_authenticated()`, `is_admin()`, `has_access()`) for what they actually prove — `is_authenticated()` is not a substitute for `can_edit_invoice(user, invoice)`, and conflating the two is one of the most common root causes in this entire vulnerability class. Confirm these helpers deny by default, use trusted identity/tenant information (not client-supplied claims), and avoid granting access when a dependency fails.

**Inspect mass-assignment boundaries** in request-to-model binding — the review signal is an entire request object passed directly into a database update (`database.update_user(user_id, request.json)`) without an explicit allowlist of permitted fields.

**Trace tenant isolation in the data layer** — database query filters, ORM scopes, repository methods, object-storage paths, search indexes, export jobs, background workers, cache keys, service-to-service requests. A query fetching a record solely by `resource_id` with no tenant constraint is the pattern to hunt for, though row-level security or partitioning can legitimately enforce isolation elsewhere — trace the *effective* mechanism, don't assume a literal tenant ID must appear in every query.

**Review every alternate execution path** reaching the same resource: REST controllers, GraphQL resolvers, mobile handlers, admin routes, bulk-processing functions, webhooks, background workers, import/export jobs, scheduled tasks, legacy endpoints. A secure web controller guarantees nothing about a GraphQL resolver or a background job touching the identical data.

**Trace permission-change propagation to enforcement**: are cached permissions actually invalidated, do sessions/tokens carry stale role claims, do downstream services trust those stale claims, do background jobs revalidate access, does a signed URL remain usable past its intended window.

**Review failure behavior explicitly** — when a permission record is missing, tenant membership can't be resolved, a policy service is down, or a role lookup throws: does the application deny access or does it accidentally proceed? A generic server error is not inherently a bypass; the question is whether the failure path results in unauthorized disclosure or action.

## H. Differential testing

Seven-step workflow: define the expected policy ("only the owner or an explicitly authorized delegate can view this document"), establish the baseline with the owner account, change exactly one variable (identity, role, tenant, action, resource state, or execution path) while holding everything else constant, compare actual outcomes (data disclosed, state changed, error behavior, audit records — not just status code), explain whether the result follows or violates the stated policy, corroborate with gray/white-box evidence where available, retest after remediation to confirm both that unauthorized access is now denied *and* legitimate access still works.

| Test | Variable changed | Expected outcome |
|---|---|---|
| BOLA | Resource owner | Unauthorized user denied |
| Role escalation | Caller role | Restricted operation denied to ordinary user |
| Tenant isolation | Tenant/resource context | Cross-tenant access denied |
| Mass assignment | Protected request field | Unauthorized property change rejected or ignored |
| Alternate route | API path or execution path | Equivalent policy enforced |
| Workflow authorization | Resource/workflow state | Invalid or unauthorized transition denied |
| Permission revocation | Current permission state | Access follows documented revocation policy |
| Data export | Requested dataset | Export limited to authorized records |

## I. Worked example: cross-tenant invoice disclosure

Black-box: Alice (Tenant A) requests her own invoice successfully, then requests a controlled Tenant B invoice using the identical session — the full invoice returns, repeated with a second controlled resource to rule out a one-off data mistake. Gray-box: documentation confirms invoices are tenant-private; test records confirm the second invoice genuinely belongs to Tenant B; the audit log shows the request was allowed. White-box: the route loads the invoice by identifier alone (`SELECT * FROM invoices WHERE invoice_id = :invoice_id`) with no tenant-membership check before serialization. Root cause: the application verifies *who* Alice is but never checks whether she's authorized for *that invoice's tenant*. Remediation: enforce tenant membership and resource-level authorization server-side, scope the query itself where practical, apply the identical policy to exports/alternate APIs/background jobs, add regression tests using identities from different tenants, and review existing logs for signs the gap was already exploited.

## J. Exploitation and validation

Minimal proof for BOLA: the four-part check from section E, demonstrated against your own two controlled accounts. Minimal proof for mass assignment: the protected field's *persisted* value actually changed, verified independently of the response body. Minimal proof for a workflow-authorization finding: the out-of-sequence or self-approved action actually completing and changing server-side state, not merely returning a 200.

## K. False-positive analysis

A `200 OK` is not itself a finding — it might contain an empty object, public metadata, or a permitted shared resource; inspect the actual disclosure. Predictable/sequential IDs are not the vulnerability — missing authorization is; don't conflate identifier complexity with the actual security control. Hidden UI controls are not protection — test the raw server endpoint directly. Administrators don't automatically have unrestricted access — some systems deliberately scope admin privileges by tenant, region, or data classification; establish the intended policy first. An expired session causing a request to fail before authorization is even evaluated is an authentication failure, not an authorization one — confirm your test identity actually reached the authorization path. A suspicious code pattern (a missing check in one controller) doesn't prove impact on its own — trace the complete path, since a dependency or data layer might enforce it elsewhere. Stale permissions persisting briefly aren't automatically a vulnerability if the system uses documented short-lived tokens or propagation windows — compare against the actual stated requirement. Testing only read access misses write/delete findings on the same resource — test every sensitive action independently. A single-record endpoint enforcing authorization correctly says nothing about a bulk export of the same data — trace the scope of the resulting dataset specifically. Intentional sharing is not broken isolation — verify ownership, membership, delegation, and sharing policy before reporting a cross-user or cross-tenant access as unauthorized.

## L. Root cause

The dominant pattern, visible in nearly every finding class in this guide: a resource fetched and returned (or a field bound to a model) with authentication confirming identity but with no corresponding authorization decision checking whether *that specific identity* may access *that specific resource or property*. The second major pattern: a centralized authorization helper that actually only proves authentication (`is_authenticated()`) being used in a context that needs to prove authorization (`can_edit_invoice()`) — the names look similar enough in code review that this substitution is easy to miss.

## M. Remediation

- Enforce authorization server-side for every sensitive action on every specific resource — never rely on frontend visibility, route guards, or client-supplied claims.
- Scope resource lookups to the caller's authorized data directly in the query/repository layer, rather than fetching broadly and checking after the fact.
- Allowlist writable fields explicitly; never bind an entire request body into a model update.
- Enforce tenant isolation consistently across every layer — application, service, data, exports, and background jobs — using trusted, validated membership, never a client-supplied tenant identifier alone.
- Centralize policy logic without creating blind spots: confirm every sensitive route and alternate execution path (GraphQL, mobile, bulk, background job) actually invokes the correct, specific policy check, not just a generic authentication gate.
- Fail closed when a required authorization check can't be completed — never silently grant access on a missing permission record or an unavailable policy service.
- Define explicitly how permission changes propagate to caches, tokens, and generated links, and test both directions: unauthorized users denied, legitimate users still working.

## N. Finding documentation

Title (vulnerability class — affected resource/action — accessible to unauthorized identity), affected endpoint/component, vulnerability class (BOLA, BFLA, privilege escalation, tenant isolation, mass assignment, etc. — the accurate category, not a guess), summary, expected vs. observed behavior, prerequisites, sanitized reproduction with controlled accounts, evidence (requests/responses, actual state changes, audit records), impact (confidentiality/integrity/availability/privilege/business consequence demonstrated, not assumed), root cause, remediation, retest criteria (the specific differential tests that must pass), severity rationale grounded in actual scope and sensitivity — a BOLA exposing one low-sensitivity record and one exposing every customer's financial data are not the same severity even though both are technically BOLA.

## Completion criteria

- [ ] Authorization attack surface mapped as an explicit identity→resource→action→expected-decision matrix before testing began
- [ ] BOLA/IDOR tested with the four-part proof standard (authenticated as A, resource belongs to B, server discloses/acts, no legitimate policy explains it)
- [ ] Horizontal and vertical privilege escalation tested independently, not assumed from one passing
- [ ] Mass assignment tested against persisted server-side state, not just response body
- [ ] Every alternate route/execution path (GraphQL, mobile, bulk, export, background job) compared against the primary one for policy consistency
- [ ] Workflow/state-dependent authorization tested for self-approval, skipped stages, and post-approval modification
- [ ] Permission-revocation propagation tested against cached state, tokens, and generated links
- [ ] Each finding classified by the actual security boundary violated and traced to the specific missing or substituted check, not reported as a generic "authorization bypass"
