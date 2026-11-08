# 09 — Account Lifecycle and Identity Proofing

**Source:** `Phases checklist.txt` Phase 8 "Account Lifecycle and Identity Proofing Security" (read in full, lines 8771–9929 — 6 BB test groups, 5 GB areas, 7 WB tracing areas), `Testing Approach .txt` Phase 4, and the course's Phase 8.

Four concepts get conflated constantly, and keeping them separate is the foundation of this entire guide: **authentication** (does the user control the required authenticator), **identity proofing** (has the application established the required facts about the claimed identity), **account lifecycle management** (is the account in a valid state, and are the attempted actions appropriate for that state), and **authorization** (is the authenticated user permitted to perform this specific action). A user can authenticate successfully while still being unverified, suspended, restricted, or ineligible for the action they're attempting — that gap is where this entire guide lives.

## A. Security objective

For every account-sensitive operation, the resulting account state must satisfy the intended identity, verification, authorization, and lifecycle policy — and a successful HTTP response is never proof that the policy was actually enforced; the resulting state has to be checked independently.

## B. Attack surface

Registration, email/phone verification, identity attribute changes (email, phone, username, legal name, recovery destination, external IdP links), account suspension/restriction/deactivation, reactivation, deletion, and identity reuse after deletion.

## C. Threat model

The realistic attacker here isn't usually breaking cryptography — they're exploiting the gap between what the application *believes* about an account's state and what it actually *enforces*. Concretely: an unverified account reaching functionality meant only for verified ones; a suspended account's existing session continuing to work even though new logins are correctly blocked; an identity change (especially a recovery destination) becoming trusted before its own confirmation step completes; a deleted account's credentials or external-IdP links somehow surviving into a newly created account with the same identity. None of these require special access — they require testing the *states*, not just the endpoints.

## D. Black-box testing

**Registration and account creation.** Baseline a normal registration, then specifically test an unverified account against every feature the documented policy reserves for verified users — a `200 OK` isn't itself a finding, the question is whether the resulting access actually violates policy. Test duplicate-identity handling across every registration channel (web, mobile, API, invitation, social login, admin creation) for inconsistency. Test a deliberately incomplete registration (stopped before verification) for whether the account ends up with unintended privileges, whether a session gets issued anyway, and whether a second registration attempt can hijack the pending record.

**Email/phone verification.** Baseline normal verification, then test a missing/malformed/invalid token (confirm rejection *and* confirm the account actually remains unverified — don't trust the status code alone). Test reuse of an already-consumed token — idempotent acceptance isn't automatically a finding, but authorizing an unintended identity or action on reuse is. The test worth doing deliberately: change an email, then before confirming the new address, check whether the new (unverified) address is already trusted anywhere — especially for account recovery. And test whether an *older* outstanding verification token (from an earlier, now-superseded change request) can still override the *current* pending change.

**Identity changes (email, phone, username, and beyond).** Baseline a normal change: is reauthentication required, must the new destination be verified before becoming trusted, does the old destination get notified, do existing sessions survive per policy. Test authorization specifically — can an account with insufficient privileges, or no privileges at all, perform the change on another account. Test change-and-recovery consistency: after a legitimate change, does recovery actually use the *current* verified destination, or can the old one (or worse, a never-verified new one) still be used. Compare the same change across every supported channel (web, mobile, API, support-assisted) for policy drift.

**Suspension, restriction, deactivation.** These are not interchangeable states — test them as what they actually are, not by assumption. Establish a session, then suspend the account through the authorized workflow, then test: a new login, the *previously issued* session against a protected resource, a protected API call, a recovery operation, and (if applicable) a privileged action. The single most common defect in this entire guide: new authentication correctly blocked, but the existing session keeps working anyway. That's not automatically a finding — some systems intentionally allow limited persistence — but it becomes one the moment it permits an operation the suspension policy explicitly says should stop.

**Deletion and identity reuse.** Baseline deletion: required confirmation, whether the account is disabled immediately or on a schedule, whether sessions are revoked, whether connected external identities are unlinked. Test the old session against protected resources after deletion starts. If a restoration window exists, test whether restoration requires genuine proof and restores the *correct* account rather than just any account matching the claimed identity. The deliberately important test: after deletion, register a new account with the *same* email/username and check whether it inherits anything from the old one — old profile data, old recovery methods, or worst of all, whether an old external-IdP link can authenticate into the *new* account.

**Identity proofing / stronger-assurance changes.** For sensitive operations (recovery-email change, phone replacement, MFA removal, external-IdP linking, legal-identity changes, privilege increases, account restoration, ownership transfer), establish what's actually required — existing session, recent reauthentication, current-destination challenge, new-destination confirmation, admin approval, stronger proofing — and confirm it's enforced consistently across every interface. Keep the distinction sharp: proving control of a mailbox is not the same as verifying legal identity, and an application conflating the two is exactly the kind of gap this section exists to catch.

## E. Gray-box testing

Check whether the account-state model can actually express contradictory states that shouldn't coexist (deleted *and* active, a pending email accidentally treated as authoritative). Compare documented API schemas against observed behavior — specifically whether client-controlled fields in a profile-update schema (an `account_status` or `verified` field exposed in the request body) are actually writable server-side, not just present in documentation. Review administrative and support workflows separately from self-service ones — they often intentionally bypass ordinary steps, so verify they have their *own* approved safeguards rather than none at all. Review audit logs for each tested transition: actor, operation, result, before/after state — and specifically investigate the inverse failure mode, a security-relevant operation that succeeded with no audit trail, or a log claiming success while the actual account state never changed.

## F. White-box testing

**Trace registration**: `request → route → input validation → account-creation service → identity-uniqueness checks → initial state assignment → verification-token creation → session issuance (if any) → persistence`. The central question: does the server establish the correct initial state *regardless of what the client submits*, and is identity uniqueness enforced at the actual authoritative data store (not just application-layer logic that a concurrent request could race past)?

**Trace verification tokens**: generation entropy/lifetime, explicit purpose binding, which account and pending change it's bound to, whether consumption invalidates it, whether tokens leak into logs/URLs/analytics. The critical check: does verification validate against the *authoritative* pending identity record, or does it trust an identity value the request itself supplies alongside the token?

**Trace identity-change transactions**: `authenticated request → authorization/recent-auth checks → proposed-address validation → pending-change record → confirmation challenge → confirmation validation → authoritative update → uniqueness enforcement → session/recovery-policy update → notification/audit`. Look specifically for: a trusted attribute updated *before* required confirmation; validation that exists only in the frontend; a transaction that updates the profile table but not the recovery subsystem, leaving the two inconsistent; a successful change that leaves old sessions valid when policy says it shouldn't.

**Trace account-state enforcement.** Find the authoritative check controlling login, session validation, protected-resource access, sensitive operations, and recovery — then confirm *every one of those paths* actually checks current state, not just login. The realistic architectural gap: suspension enforced at login but never re-checked on subsequent API requests, or enforced in one controller but not in a background job or an alternate route that reaches the same operation.

**Trace suspension and session revocation**: `suspension request → admin authorization → state update → session/token revocation → cache/session-store invalidation → audit → subsequent auth/authz checks`. Specifically check whether session validation actually *consults* the updated state, whether a cache can serve a stale "active" status past the suspension, and whether a signed/stateless token architecture has any mechanism to respect revocation at all before concluding it's broken (it may be a deliberate, documented tradeoff rather than an oversight).

**Trace deletion and restoration** through every component that might retain something: primary record, sessions/refresh credentials, recovery destinations, external identity links, organization memberships, API credentials, search indexes/caches, background jobs. The question for each: does retained data or a retained credential unintentionally preserve access, ownership, or identity association past what the deletion policy actually permits?

**Trace authorization of sensitive identity changes.** Confirm the implementation authorizes the actual *target* account and the actual *action* — not just "a valid session exists." For admin/support-performed changes specifically, confirm the actor's role and scope actually cover that specific operation on that specific account.

## G. Differential testing

The central technique for this entire guide: run the *same* operation across different account states and compare.

| Operation | Verified | Unverified | Suspended | Deleted |
|---|---|---|---|---|
| Login | Baseline | Compare with policy | Must follow suspension policy | Must follow deletion policy |
| Read protected profile | Baseline | Compare with policy | Compare with restriction policy | Must follow deletion policy |
| Change recovery destination | Baseline | Compare assurance requirements | Usually restricted | Must follow deletion policy |
| Start password recovery | Baseline | Compare with policy | Must follow suspension policy | Must follow deletion policy |
| Use existing session | Baseline | Compare with policy | Verify session handling | Verify revocation policy |

Also compare the same operation across channels (web/mobile/API/admin/support) and capture explicit before→action→after triples for every state transition — the evidence that matters is never the status label alone, it's the specific prohibited operation that remained possible afterward. And specifically test tokens against account state independently: a session can remain cryptographically valid while the account it belongs to is no longer permitted to use it — `T0: active, session issued → T1: suspended → T2: session requests protected resource → T3: does it still succeed?`

## H. Exploitation and validation

Minimal proof for state-enforcement gaps: demonstrate the specific prohibited operation succeeding against a controlled account in the restricted state, with a before/after state comparison. Minimal proof for identity-reuse findings: demonstrate a newly registered account (same email/username as a deleted one) actually inherits a privilege, credential, or external-IdP link from the old account. Minimal proof for premature-trust findings: demonstrate an unconfirmed new destination (email, phone) is already usable for something security-sensitive before its own confirmation completes.

## I. False-positive analysis

Don't treat every unverified account as compromised — some applications intentionally permit limited pre-verification access; establish what's actually policy first. Don't treat a `200 OK` as proof of access — verify the actual resulting state and capability. Don't treat token reuse as automatically exploitable — idempotent acceptance is fine if it doesn't authorize an unintended target or action. Don't treat soft deletion (a retention window) as failed deletion. Don't assume every architecture must revoke every session instantly — check the actual documented policy before calling persistence a bug. Don't confuse mailbox control with legal identity — an email verification token proves neither more nor less than that. Don't report merely inconsistent *responses* (different status codes or messages) as a finding without a real security consequence behind the inconsistency. Don't treat a missing audit log as proof of unauthorized access — that's a separate, narrower finding (a monitoring gap) unless you can independently demonstrate the underlying access failure too.

## J. Root cause

The recurring patterns: initial or transitional account state assigned (or trusted) from client input instead of being strictly server-derived; state checked at one enforcement point (typically login) but not re-checked at every other path that should respect it; verification/confirmation tokens validated against a request-supplied identity instead of the authoritative pending record; and multi-step transitions (identity change, suspension, deletion) that update one subsystem (profile) without atomically updating the related ones (recovery, sessions, external links), leaving the two inconsistent and exploitable.

## K. Remediation

- Model account states explicitly (pending-verification, active, restricted, suspended, deactivated, deleted) wherever the business policy actually requires the distinction, and keep every bit of that state strictly server-controlled.
- Bind every verification/confirmation token to a specific purpose and target account; validate against the authoritative pending record, never a client-supplied identity alongside the token.
- Require confirmation before a new destination becomes authoritative for anything security-sensitive, recovery in particular.
- Enforce current account state at every path that touches the account — not just login — including APIs, background jobs, and alternate routes.
- Design session/credential revocation to actually reach the mechanisms that validate sessions (cache invalidation, not just a database flag nobody re-checks).
- Design restoration and identity reuse deliberately: never auto-transfer old credentials, privileges, or external-IdP links onto a newly created or restored account without independent verification.
- Enforce identity uniqueness at the authoritative persistence layer, not just in application-level pre-checks that a race condition could slip past.

## L. Regression testing

Re-test suspension/deletion session enforcement after any session-store or caching-layer change. Re-test identity-change confirmation ordering after any profile-service refactor. Re-test identity reuse after deletion whenever the deletion or restoration workflow changes.

## M. Finding documentation

Title, summary (what the application permits and why it violates policy), affected component/operation, preconditions (required account state), expected vs. actual behavior with its source, sanitized reproduction, evidence (request/response, resulting account state, timestamps), impact (realistic, not exaggerated), root cause, remediation, severity and confidence with any unresolved assumptions noted explicitly.

## Completion criteria

- [ ] Registration tested for correct initial state assignment regardless of client input
- [ ] Verification tested for proof validity, reuse behavior, and correct binding to the current (not a superseded) pending change
- [ ] Identity changes tested for confirmation-before-trust, especially for recovery destinations
- [ ] Suspension tested against both new logins and previously issued sessions, with before/after state captured
- [ ] Deletion tested for session survival and for identity-reuse inheritance on re-registration
- [ ] Sensitive identity-proofing operations tested for the actual required assurance level, not assumed from documentation
- [ ] Every operation compared across at least two account states and, where applicable, across channels
- [ ] Each finding traced to a specific state-enforcement or binding gap, not reported as a generic "lifecycle bug"
