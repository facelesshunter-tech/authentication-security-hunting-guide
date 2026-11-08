# 03 — Session Security

**Source:** `Phases checklist.txt` Phase 6 "Cookie and Session Security" (read in full, lines 6243–7581), `Testing Approach .txt` Phase 8, and the course's Phase 6 for conceptual grounding.

Authentication establishes who a user is; session management preserves that across subsequent requests. If the session lifecycle is insecure, a strong password and working MFA may not protect the account at all — the attacker doesn't need to beat either control if the session that survives them is forgeable, fixable, or simply never actually revoked.

## A. Security objective

A session credential should represent exactly the authentication state it claims to, be generated unpredictably, be rejected the instant it should no longer be valid (logout, password change, revocation, account restriction, expiration), and never let one user's session resolve to another user's identity or privileges.

## B. Attack surface

Browser-based sessions, API bearer tokens, refresh tokens, remember-me functionality, mobile sessions, federated login sessions, administrative sessions, legacy session mechanisms, the session store and authentication middleware itself, and logout/global-revocation features.

## C. Threat model

Six distinct failure modes matter, and they're independent — a cookie can have every recommended attribute and still be insecure, because **a cookie is a transport mechanism; the server-side session or token-validation system determines whether the credential actually grants access.** The six: cookie security (transport-level exposure), session creation (fixation, incorrect state at issuance), session validation (missing checks on a protected request), session termination (logout/revocation that doesn't actually revoke), session isolation (cross-user bleed), and session lifecycle consistency (web vs. API vs. mobile vs. legacy enforcing different rules for the same underlying session).

## D. Black-box testing

**Map session creation and transitions.** From a clean browser profile: request a protected resource anonymously, authenticate, record what cookies change, complete MFA/step-up if applicable, compare state before and after, then log out and check what happens next. The target finding is a session representing a partial authentication state (password-accepted, MFA-pending) that can still reach a resource requiring full authentication.

**Cookie attributes.** Check `Secure`, `HttpOnly`, `SameSite`, `Domain`, `Path`, and expiration on the actual session cookie — not just any cookie present. A missing attribute in one response isn't proof of exploitability in production; confirm the real deployment. `SameSite=None` is not automatically a finding — federated login architectures can legitimately require it. `HttpOnly` doesn't stop the browser from sending the cookie automatically and doesn't eliminate every XSS consequence — don't oversell what it protects against. Domain/Path scoped wider than necessary is a real concern (shared across subdomains that shouldn't trust each other), but Path is not a reliable security boundary against same-origin malicious code either way.

**Session fixation.** This is not simply "does the ID change after login" — the real question is whether an attacker can establish or know a session identifier in advance and have it become privileged once the victim authenticates. Obtain a pre-auth identifier, record it, authenticate, check whether that exact identifier is now privileged and whether the pre-auth session is still accepted anywhere. An unchanged ID after login is a signal to investigate, not a finding on its own.

**Identifier quality.** From black-box alone you can spot an identifier that visibly embeds a username, timestamp, or counter — that's a real concern. A "random-looking" identifier proves nothing about actual entropy or generation quality; that requires gray-box or white-box evidence. Never claim predictability from a short sample of observed values.

**Expiration.** Test both idle and absolute timeout with a controlled account, and always confirm with a **fresh** request — a cached page, a still-loaded tab, or a refresh token legitimately renewing an access token are all false positives for "the session never expires."

**Logout and revocation.** Authenticate, record the session, log out normally, then send a fresh request with the old credential. Clearing a browser cookie is not the same as revoking a server-side session — test the credential itself, not the browser's local state. If the app supports revoking other sessions or logging out everywhere, test that with controlled sessions too.

**Session invalidation after password change.** Authenticate the same test account in two isolated profiles, change the password in one, then test both with fresh requests. Some systems deliberately keep the session that performed the change active while revoking others — that's a legitimate design, not automatically a finding; test against the actual stated policy.

**Session state across account changes.** Suspend, deactivate, change role/tenant, or remove MFA on a controlled account via an authorized administrative action, then reuse the existing session. Not every change requires instant invalidation — some systems re-evaluate on every request, others accept a bounded revocation delay — but the realistic delay should match what the application claims.

**Session isolation between users.** Authenticate two controlled accounts in separate profiles; confirm Account A's session cannot be used to reach Account B's permitted resources, and that server-side identity comes from the validated session, not from any client-controlled user identifier in the request. Two accounts getting cookies with the same *name* is normal and proves nothing about whether the underlying credentials are isolated.

**Cross-subdomain/application scope.** Where multiple subdomains or applications are in scope, check whether a session cookie is shared more broadly than the architecture requires, and whether authentication intended for one application is silently accepted by another. Shared sessions can be a deliberate design — the finding is whether the resulting access matches the intended trust boundary, not the sharing itself.

**Client-side storage.** Check cookies, `localStorage`, `sessionStorage`, IndexedDB, and service-worker storage for long-lived bearer credentials sitting in JavaScript-accessible locations without justification, and confirm logout actually removes or invalidates them rather than just clearing the UI.

## E. Gray-box testing

**Map the architecture**: is the session server-side or a self-contained token, which service issues it, which services accept it, where are expiration/revocation decisions actually made, is the same session honored by both web and API routes. Produce the flow: login → credential issuance → client storage → protected request → validation → authorization → expiration/revocation, including alternate routes.

**Configuration vs. deployment.** A secure-cookie setting in a config file is not proof of production behavior — environment-specific overrides are the single most common gap here (secure cookies enabled in dev, disabled in prod through a deploy-time override). Confirm which service actually consumes the setting and compare it against what you observed in black-box testing.

**Session-store records** (where authorized): check ownership, creation/activity/expiration timestamps, revocation state, and whether a revoked session's record remaining in the database actually still validates — some systems retain records for audit purposes without treating them as live, so the record existing is not the same as the session being valid.

**Revocation propagation across services.** Identify every component that accepts the credential, then check whether logout/restriction/credential-change events actually reach all of them — a web session revoked while its refresh token stays valid, or an API service that doesn't observe the same revocation as the web app, are the realistic failure patterns here.

**Logs and telemetry**: confirm session creation, transitions, expiration, logout, revocation, and failed validation are recorded with correlatable timestamps and account identifiers, with secrets excluded. A missing log entry is not itself proof of a revocation failure — verify the actual session behavior independently before concluding the control failed.

## F. White-box testing

**Trace creation**: `login handler → credential verification → authentication-state checks → session creation → persistence → cookie/token issuance → response`. Confirm the session is created only after all required steps, the identifier is generated securely and rotated at the right transitions, and no exception path can issue a session unintentionally.

**Trace validation**: `request → credential extraction → session lookup/token validation → expiration/revocation checks → account-state checks → authorization`. Confirm validation runs on every relevant request and failure handling can't fall through into an authenticated state.

**Identifier generation**: confirm a CSPRNG is used, with no timestamp/username/counter as a security-critical component, and no custom session-ID scheme reinventing what a supported framework already provides securely.

**Rotation and fixation defenses**: check every transition (login, MFA, step-up, privilege elevation, account switching, reauth) for identifier rotation — and critically, that the *old* identifier actually loses its authority afterward. Issuing a new cookie value while the old one stays fully authenticated doesn't remove the fixation risk, it just hides it.

**Expiration/renewal logic**: confirm idle and absolute timeouts are enforced server-side, a refresh credential can't be used after revocation, and renewal itself requires the appropriate authentication state.

**Logout/revocation code**: the realistic finding pattern here is a logout handler that only clears the browser cookie while the server keeps accepting the underlying credential — confirm the code actually invalidates server-side state, not just client-visible state.

**Identity binding**: confirm the server derives user/tenant/role/auth-strength from validated session data, never from a client-supplied identifier that overrides it, and that a privilege or tenant change actually takes effect rather than being masked by stale cached session data.

**Cookie config in the real deployment path**: trace the actual code/config that sets `Secure`/`HttpOnly`/`SameSite`/`Domain`/`Path`/expiration, including reverse-proxy and framework-level overrides — don't infer production security from a local dev configuration.

**Authorization after validation is a separate control.** A session correctly validating as "this is an authenticated user" doesn't mean every authorization check downstream is correct — that's an authorization flaw (Guide 16), not a session-management flaw, even though the symptom shows up here. Report it in the right category.

## G. Differential testing

Seven-step method: define the requirement ("after logout, the session must no longer authorize protected requests"), establish the baseline (authenticate, confirm access), change one variable (perform logout, nothing else), compare observable behavior (fresh request with the old credential), investigate internally if it still works (revocation config, session store, validation code), confirm the actual consequence, report the root cause and affected paths.

| Test | Black-box evidence | Gray-box evidence | White-box evidence |
|---|---|---|---|
| Cookie attributes | Actual `Set-Cookie` response | Deployment configuration | Cookie-creation code |
| Session fixation | Identifier/state before and after login | Session records and settings | Rotation and transition logic |
| Session expiration | Fresh requests after timeout | Timeout settings and metadata | Expiration/renewal logic |
| Logout | Reuse test after logout | Revocation records and telemetry | Logout and invalidation handlers |
| Password change | Existing sessions before/after | Policy and session records | Session-revocation logic |
| Privilege changes | Access before/after change | Account and role state | Claim-refresh and authorization code |
| Session isolation | Separate-account behavior | Session ownership metadata | Identity binding and validation |
| Identifier quality | Observable identifier patterns | Library/config details | Random-generation implementation |
| Cross-subdomain scope | Cookie behavior across hosts | Domain and app architecture | Cookie configuration and trust relationships |

## H. Exploitation and validation

Minimal proof for fixation: demonstrate the attacker-known pre-auth identifier is accepted as fully authenticated post-login, against your own controlled account. For revocation failures: a fresh request with the old credential succeeding after logout/password-change/restriction, nothing more elaborate needed. For isolation: Account A's session reaching Account B's permitted resource, with both accounts under your control.

## I. False-positive analysis

Challenge each of these before reporting: missing `HttpOnly` alone, without an identified exposure path; `SameSite=None`, without checking whether federated login requires it; an unchanged session ID after login, without checking whether the old one actually retains authority; a cookie still present after logout, without testing the credential against a fresh protected request; a cached page appearing post-logout; a session record still existing in storage, without confirming validation actually accepts it; a retained session after password change, without checking the documented policy (some systems intentionally keep the current session alive); JWTs being used at all — a token format is not proof of secure session management, validate the actual handling.

## J. Root cause

Session findings usually trace to: validation happening somewhere but not everywhere a credential is accepted (web checks revocation, API doesn't); rotation generating a new identifier without actually stripping the old one's authority; a config-level protection (secure cookie, timeout) silently overridden per-environment; or revocation implemented as "clear the client cookie" instead of "invalidate the server-side record."

## K. Remediation

- Serve authenticated traffic over HTTPS only; apply `Secure`, appropriate `HttpOnly`, and a `SameSite` policy matched to the actual cross-site requirements — verify the deployed config, not just the source default.
- Generate identifiers with a CSPRNG, rotate them at every authentication-strength transition, and ensure the old identifier is actually stripped of authority, not just superseded.
- Validate every session server-side on every relevant request, across every service that accepts it — web, API, mobile, legacy — not just the primary path.
- Implement logout and revocation as genuine server-side invalidation, propagated to every service and credential type (access token, refresh token) involved, within a documented and justified delay if any.
- Bind identity, tenant, and privilege to trusted session data only; re-evaluate on change rather than trusting cached claims indefinitely.

## L. Regression testing

Re-test revocation propagation after any new service is added that accepts the session credential, re-verify cookie configuration after any deployment or reverse-proxy change, and re-check fixation defenses after any change to the login/MFA/step-up transition code.

## M. Finding documentation

Title, summary, affected components (endpoint, cookie, session service, token type, or code path), preconditions, reproduction (sanitized), expected vs. actual behavior, evidence (HTTP traffic, cookie attributes, session-state comparison, config, store metadata, source path, sanitized logs), impact, root cause, remediation, retest criteria. Never include a reusable live cookie or session token in a report. The governing question for severity: can a credential that should not grant access still establish or preserve an unauthorized authentication state — and how far does that reach (one account, multiple accounts, a whole tenant, MFA bypassed entirely)?

## Completion criteria

- [ ] Session transitions mapped through login, MFA, step-up, and logout
- [ ] Cookie attributes checked against the actual production deployment, not just one response
- [ ] Fixation tested as attacker-known-identifier-becomes-privileged, not just "did the ID change"
- [ ] Expiration (idle and absolute) tested with fresh requests, not cached pages
- [ ] Logout/revocation tested as server-side credential invalidation, not client-side cookie clearing
- [ ] Session behavior after password change, account restriction, and privilege change tested against documented policy
- [ ] Isolation tested between two controlled accounts
- [ ] Revocation propagation checked across every service that accepts the credential, not just the primary one
