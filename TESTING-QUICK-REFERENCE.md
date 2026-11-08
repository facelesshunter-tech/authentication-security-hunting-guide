# Authentication Security Testing — Quick Reference Checklist

This is the file to have open while you are actually hunting. It is condensed on purpose: every item below is a checkbox, not an explanation. For the reasoning behind a step, the HTTP evidence templates, and the black-box/gray-box/white-box/differential breakdown, go to the matching file in `attack-guides/`.

Source note: this checklist is built from `Testing Approach .txt` (read in full), condensed into checkbox form. The deeper, more granular checklist (`Phases checklist.txt`, 22,674 lines) feeds the full attack guides as each one is built, not this quick reference — this file is meant to stay short enough to actually use mid-engagement.

Work phases roughly in order, but follow evidence, not the list. If a phase does not apply to the target (no SAML, no passkeys), check "not applicable" and move on rather than forcing a test.

---

## Phase 0 — Scope and Test Preparation

- [ ] Confirm authorized domains, subdomains, API hosts, mobile apps, environments
- [ ] Read prohibited-testing rules (automation, brute force, account creation, destructive actions)
- [ ] Confirm whether production testing is permitted
- [ ] Prepare controlled accounts: A (ordinary), B (second ordinary), C (different tenant), D (privileged, if available), E (unverified/disabled, if supported)
- [ ] Set up your evidence template: endpoint, account/role, initial state, request, response, resulting state, timestamp

## Phase 1 — Map the Identity System

- [ ] Inventory entry points: registration, login, logout, password change, forgot/reset password, email/phone verification, MFA enroll/verify/remove, backup codes, passkeys, OAuth/OIDC, SAML, account linking, session management, reauth/step-up, deactivation/deletion, admin auth, legacy routes, API auth, mobile auth, trusted devices
- [ ] Map authentication states: anonymous, unverified, password-accepted, MFA-pending, fully authenticated, step-up-required, restricted, disabled, deactivated, deleted, privileged
- [ ] Record the expected transitions between states
- [ ] Identify trust boundaries: client, frontend, backend, IdP, session store, database, background workers, storage
- [ ] List client-controlled values that must be server-verified: user ID, tenant ID, role, account status, auth flags, ownership IDs, redirect parameters

## Phase 2 — Establish the Baseline

- [ ] Record successful login: endpoint, session/token issued, MFA required, redirect, resulting permissions
- [ ] Record normal failures: wrong password, unknown account, missing/malformed input, unverified/disabled account, missing MFA, wrong code, expired session
- [ ] Establish negative controls: anonymous on a protected resource, Account A on Account B's object, ordinary user on an admin function, expired token on a protected endpoint

## Phase 3 — Authentication Information Leakage

- [ ] Test account enumeration at login, registration, password reset, and any other discovery endpoint — compare existing vs. nonexistent account: status, body, redirect, length, timing
- [ ] Test timing-based leakage with repeated, controlled comparisons — one slow response proves nothing
- [ ] Check error/diagnostic leakage: stack traces, database errors, framework details, debug info, secrets in error responses
- [ ] Check cache/log exposure: shared caches, browser history, URLs, referrer headers, analytics, application logs

## Phase 4 — Registration and Account Lifecycle

- [ ] Test required-field enforcement, duplicate-account handling, email/phone normalization
- [ ] Test verification tokens: expiration, account binding, purpose binding, single-use, replay resistance, attempt limits
- [ ] Test account-state enforcement: can unverified/disabled/restricted accounts authenticate through *any* route?
- [ ] Test email/phone change: recent auth required?, new channel verified?, old channel notified?, sessions revoked per policy?
- [ ] Test deactivation/deletion: sessions, refresh tokens, linked IdP accounts, recovery tokens, background jobs all handled?

## Phase 5 — Password Storage and Credential Defenses

- [ ] If source available: review hashing algorithm, salting, work factor, pepper, hash exposure
- [ ] Review password verification: comparison logic, rehash-on-login, legacy migration, consistency across *all* auth paths
- [ ] Assess credential-attack defenses: login attempt limits, account/source rate limiting, lockout behavior, monitoring

## Phase 6 — Password Change and Account Recovery

- [ ] Map the full flow: request → verification issued → token validated → password set → sessions handled → user notified
- [ ] Test token properties: unpredictable, account-bound, purpose-limited, expiring, single-use, invalidated after use
- [ ] Test recovery authorization: does it trust a client-supplied account ID, an unverified email, stale state, or a token issued for another purpose?
- [ ] Test enumeration/timing on recovery responses
- [ ] Check token leakage: URLs, browser history, referrer headers, logs, analytics, email generation
- [ ] Check host/redirect handling in recovery links (Host header, forwarded-host header)
- [ ] Test password-change security: current password required?, MFA/step-up required?, sessions revoked?, reset tokens invalidated?

## Phase 7 — MFA and Authenticator Lifecycle

- [ ] Map MFA states and confirm the server-side evidence behind each one
- [ ] Test MFA enforcement consistency across *all* auth routes — no downgrade via a legacy or fallback path
- [ ] Review TOTP: secret generation/storage, verification window, attempt limits, replay prevention
- [ ] Test push auth for approval-fatigue or weak number-matching — do not spam real users
- [ ] Test SMS/phone MFA: number verification, number-change handling, recycled-number risk
- [ ] Test enrollment/removal/replacement: correct proof required?, notification triggered?, old authenticators invalidated?
- [ ] Test MFA recovery/backup codes: does recovery defeat the protection MFA is supposed to provide?
- [ ] If passkeys/WebAuthn supported: test challenge freshness, origin validation, RP ID validation, credential-to-account binding

## Phase 8 — Session and Cookie Security

- [ ] Check session ID generation: unpredictable, validated server-side
- [ ] Test session fixation: does the ID rotate on login or privilege change?
- [ ] Check for session ID leakage: URLs, logs, storage, insecure transport
- [ ] Review cookie attributes: `Secure`, `HttpOnly`, `SameSite`, `Domain`, `Path`, `Max-Age`, `__Host-`/`__Secure-` prefixes
- [ ] Test expiration: idle timeout, absolute timeout, remember-me, concurrent sessions
- [ ] Test logout with a *fresh* request afterward — a cached page proves nothing
- [ ] Test revocation after: password reset, password change, MFA removal, account disablement, privilege change, tenant removal
- [ ] Confirm MFA-pending/step-up-pending sessions cannot reach operations requiring stronger auth

## Phase 9 — Reauthentication and Step-Up

- [ ] Identify sensitive actions: password/email/phone change, MFA removal, passkey registration, API key creation, payout changes, exports, privilege changes, account deletion
- [ ] For each: is current-password, fresh MFA, a recent-auth timestamp, or operation-specific confirmation required?
- [ ] Confirm the backend enforces this independently of the frontend and rejects stale auth state
- [ ] Confirm the sensitive operation's resulting state actually changed — not just a success-looking response

## Phase 10 — OAuth2, OIDC, SAML and Federation

*(Mark not applicable and skip whatever the target doesn't implement.)*

- [ ] Map IdP, client/SP, authorization server, token types, account linking, logout behavior
- [ ] OAuth/OIDC: redirect URI validation, `state` binding, PKCE, `nonce`, code handling, token validation, issuer/audience checks, JWKS/signing-key trust, public vs. confidential client handling
- [ ] SAML: issuer, audience, recipient, destination, subject confirmation, `NotBefore`/`NotOnOrAfter`, `InResponseTo` correlation, signature validation, certificate trust, XML Signature Wrapping resistance
- [ ] Account linking: bound via a trustworthy, stable IdP identifier — not unverified email alone?
- [ ] Federated logout: does logout actually revoke the app session, the IdP session, refresh tokens, and local mappings?

## Phase 11 — Legacy, API and Mobile Authentication Paths

- [ ] Discover alternate routes: old API versions, legacy login, mobile-specific routes, admin routes, partner/integration endpoints, deprecated schemes
- [ ] Compare MFA enforcement, account-state enforcement, session handling, rate limiting across main vs. alternate routes
- [ ] If source available: review legacy credential/session formats, deprecated grants, hardcoded secrets, dormant accounts

## Phase 12 — Authentication Cryptography

- [ ] Inventory sensitive values: password hashes, reset/verification tokens, session IDs, MFA secrets, WebAuthn challenges, OAuth `state`/`nonce`, SAML request IDs, signing/encryption keys
- [ ] Review token generation: CSPRNG used?, sufficient entropy (encoded length is not entropy), account/purpose binding, expiration, single-use, revocation
- [ ] If source available: confirm correct use of password hashing vs. general hashing vs. encryption vs. MAC vs. signatures
- [ ] Review key/secret handling: storage, rotation, hardcoded secrets, exposure via logs or source control, TLS configuration
- [ ] Review comparisons: constant-time comparison for sensitive values, signature-verification order, trusted-key selection

## Phase 13 — Authentication Logging and Monitoring

- [ ] Confirm these are logged: login success/failure, MFA outcomes, password changes/resets, authenticator changes, session create/revoke, federation events, account disablement, privilege changes
- [ ] Confirm logs do **not** contain: passwords, OTPs, recovery codes, raw session cookies, tokens, authorization headers, TOTP secrets
- [ ] Assess detection capability: credential stuffing, password spraying, excessive failures, MFA anomalies, repeated recovery attempts
- [ ] Check log protection: access control, integrity, retention, redaction, injection resistance

## Phase 14 — Authentication-Adjacent Authorization

- [ ] For each sensitive resource: establish the intended policy (identity, action, resource, ownership, tenant, role, state)
- [ ] Test horizontal escalation: Account A accessing Account B's resources
- [ ] Test IDOR/BOLA: direct retrieval, modification, deletion, search, bulk operations, exports, file downloads
- [ ] Test vertical escalation: ordinary user invoking privileged/admin functions
- [ ] Test function-level authorization server-side — a hidden frontend button is not a control
- [ ] Test mass assignment / property-level authorization on: `role`, `is_admin`, `is_verified`, `tenant_id`, `owner_id`, `approved`, `balance`, `payment_status`
- [ ] Test multi-tenant isolation: cross-tenant search, object access, exports, admin functions, background jobs
- [ ] Test alternate authorization paths: different HTTP methods, API versions, bulk operations, exports, mobile endpoints
- [ ] Test business-logic authorization: refunds, orders, payouts, ownership transfer, price changes
- [ ] Test workflow-state authorization: permitted transitions, actor authority, self-approval prevention
- [ ] Test stale authorization: access correctly updated after role/tenant/session/ownership changes?
- [ ] Test file/export authorization: ownership, download permissions, link expiration and scope

## Phase 15 — Source-Code Review

*(When source is available.)*

- [ ] Trace: route → auth middleware → auth service → authorization checks → data access → response
- [ ] Review login implementation: what conditions must be true before a fully-privileged session is issued?
- [ ] Review auth-state representation: can the client directly choose the state, or does the server derive it from trusted evidence?
- [ ] Review protected-route enforcement: middleware applied consistently?, resource scoped to the correct identity?
- [ ] Review session creation/revocation: does each revocation event actually invalidate the intended credential?
- [ ] Compare equivalent implementations across controllers/services/API versions for inconsistency

## Phase 16 — Validate Attack Chains and Account-Takeover Impact

- [ ] Identify a plausible chain — these are hypotheses, not claims (enumeration → recovery weakness; MFA gap → alternate route; revocation failure → persistent access; linking error → wrong-identity auth; missing tenant authz → cross-tenant access)
- [ ] Validate each link independently with controlled accounts; stop if a step would exceed authorization
- [ ] Prove actual impact — distinguish unusual response, information disclosure, policy violation, unauthorized access, state change, privilege escalation, and account takeover; these are not interchangeable
- [ ] Use minimal proof against your own controlled accounts only — never real users' data

## Phase 17 — Report, Classify and Recommend Fixes

- [ ] Write: title, affected component, summary, preconditions, reproduction steps, expected vs. actual behavior, evidence, impact, root cause, remediation, limitations
- [ ] Assess severity from: attacker privilege required, victim interaction required, ease of exploitation, confidentiality/integrity/availability impact, accounts affected, existing mitigations
- [ ] Confirm scope, redact credentials/tokens, check for duplicates, follow the program's disclosure rules

## Phase 18 — Final Coverage Review

- [ ] Walk every phase above and mark each: covered, not applicable, or needs revisiting
- [ ] Confirm out-of-scope items (no SAML, no passkeys, etc.) are explicitly marked not applicable, not silently skipped

---

## The three rules

1. **Follow the security state, not just the endpoint.** A login, reset, or MFA endpoint is one part of a larger state machine.
2. **Test authorization after authentication.** A secure login does not guarantee secure access to objects, functions, properties, tenants, or workflows.
3. **Report demonstrated impact, not theoretical possibility.** An unusual response is a lead. A finding needs evidence that a real security property was actually violated.

## Explicit out-of-scope note

Deep JWT exploitation beyond algorithm-confusion basics (weak-secret cracking, `kid`/`jku`/`x5u` header attacks) and dedicated race-condition exploitation are not covered by this checklist's source material. If you're hunting those, they're on your separate general vulnerability-class roadmap, not this authentication-specific one.
