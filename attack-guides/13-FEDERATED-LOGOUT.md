# 13 — Federated Logout and Identity Persistence

**Source:** `Phases checklist.txt` Phase 14 "Federated Logout and Identity Persistence" (read in full, lines 17512–18723 — 10 BB tests, 5 GB areas, 8 WB tracing areas), `Testing Approach .txt` (folded into its federation phase), and the course's Phase 14.

**The central invariant:** after logout, the application must enforce its documented session-termination policy — a user must not regain access through a supposedly terminated session or token unless a new, explicitly permitted authentication flow occurs. **Critical distinction to hold throughout this entire guide: logging out of one application does not automatically mean logging out of the identity provider or every connected application.** Whether those other sessions must also end depends entirely on the documented policy — your job is to establish what that policy actually promises before deciding whether observed behavior is a vulnerability.

## A. Security objective

Four (sometimes five) independent security contexts must each be evaluated on their own terms: the application session, the identity-provider session, the access token, the refresh token, and — where relevant — sessions of other applications sharing the same IdP. None of these are interchangeable, and none of them being terminated proves anything about the others.

## B. Attack surface

Local application logout, IdP logout (RP-Initiated Logout, front/back-channel), cross-application session persistence, refresh/access token lifecycle after logout, silent re-authentication via SSO, SAML Single Logout, logout CSRF, cross-device logout, and browser-cached protected content.

## C. Threat model

The realistic attacker usually already possesses an old session cookie or token (their own, for authorized testing) and is checking whether it still works after the user believed they'd logged out — either because server-side invalidation never happened, propagated too slowly, or only covered one of the several independent contexts listed above. A second, distinct attacker forges a cross-site request to the logout endpoint itself (logout CSRF) — typically low impact on its own, but worth checking whether logout triggers any other state change that raises the stakes.

## D. Black-box testing

**Verify local application logout first.** Log in, open a protected page, record the session identifier, log out, then replay the request to the same protected page *and a second, different protected endpoint* using the old cookie — middleware inconsistency across routes is a real, specific failure mode worth checking for directly. A cleared `Set-Cookie` in the logout response is not evidence of server-side invalidation; only a rejected replay request counts.

**Distinguish local logout from IdP logout.** Log out, then revisit the application and observe whether the IdP silently re-authenticates via an active SSO session, prompts for credentials, or the original application session itself never actually ended. These are three different outcomes requiring three different interpretations: (A) local logout with IdP session intentionally preserved — likely correct if that's the documented design; (B) IdP logout was promised but SSO continues without reauthentication — likely incomplete global logout; (C) the original application session cookie itself remains valid — a different, more basic invalidation failure. Automatic SSO alone is never proof of a vulnerability by itself — only (C), or a documented-but-unfulfilled promise of (B), is.

**Cross-application logout.** Using two applications sharing an IdP: authenticate to both, log out of one, then check both. The specific case worth extra attention: App A logs out but App B shows *automatically re-authenticated* — this could mean B retained its own session independently (fine if that's the design) or the IdP session silently re-issued a new one (needs further investigation to determine which).

**Access-token behavior after logout.** A self-contained access token (e.g. a JWT) remaining valid immediately after logout is not automatically a finding — it may be entirely valid design if the architecture doesn't support immediate revocation and the documented policy only promises expiration-based invalidation. The actual finding: the documented policy promises immediate revocation and the token remains usable anyway, or a token is accepted past its own expiration time. Keep access tokens, refresh tokens, session cookies, and ID tokens analytically separate — they follow different rules and an ID token specifically should never be assumed to work as a general API credential.

**Refresh-token persistence.** Obtain a refresh token, confirm it works, log out, then attempt the documented refresh operation with the same test token. If rotation is enabled, make sure you're not accidentally testing a token that was already superseded by a legitimate rotation — distinguish the current token from any previously-used one explicitly.

**Silent authentication / session resurrection.** Log out, return to a protected route, and observe whether reauthentication happens without a credential prompt. The critical distinction: a *new* session issued through a legitimate SSO flow is not the same thing as the *old* session surviving — confirm which one actually happened by checking the session identifier, not just whether access was regained.

**Logout and MFA freshness.** After logout and a fresh authentication transaction, confirm the system actually re-demands whatever factors its policy requires — particularly for privileged accounts. Don't assume every return visit must re-prompt for MFA; remembered-device and SSO-freshness policies can legitimately suppress a repeat prompt. The actual issue is a mismatch between assurance *actually established* and assurance *required* for the operation being attempted.

**Browser history and cached content.** Back-button and cache-reload are not server-side authorization tests — the real test is whether *fresh* protected data can still be retrieved, or whether an authentication artifact (token, session fragment) is left sitting in browser storage, usable after logout.

**Logout CSRF.** Check whether the logout endpoint requires a CSRF token or other origin control, and whether it performs any *additional* sensitive action beyond simple logout (account switching, IdP-side logout triggering). Forced logout alone is typically a low-severity nuisance — it only becomes more serious if it causes identity confusion or enables a further attack.

**Cross-device logout.** Authenticate the same account on two devices, log out on one, and test the original session on both — plus any dedicated "log out all devices" feature separately. A finding requires a genuine mismatch between the promised scope and actual behavior, not merely the existence of another still-active session where that's the documented design.

## E. Gray-box testing

Compare documented logout architecture (local session, IdP logout, OIDC end-session, SAML SLO, refresh-token revocation, cross-device revocation, propagation delays, failure behavior when the IdP is unreachable) against observed behavior directly. Review session records where authorized: a session record remaining present after logout isn't itself proof it's still usable — some systems retain inactive records for audit purposes; the strong evidence is the session marked active *and* the server accepting it for a protected request. For token revocation specifically, if a refresh token is marked revoked but a refresh request still succeeds, investigate rather than assume — possible innocent explanations include a different token being tested, propagation delay, a different authorization server handling the request, or a separate legitimate re-authentication having issued a new token. Review logs for the full event chain (auth → session creation → logout initiated → local invalidation → IdP logout request/response → refresh-token revocation → any subsequent access attempt) and specifically whether local logout and federated logout are distinguishable in the log data.

## F. White-box testing

**Trace the logout handler**: `request → route → middleware → logout handler → session service → token/revocation service → IdP integration → response`. The critical code-review distinction: a handler that calls `clearCookie()` only clears the *browser's* copy — it says nothing about server-side invalidation, which requires a separate session-store call. Neither alone is sufficient if the documented policy also requires refresh-token revocation or IdP termination; trace every required action independently and verify each one's actual result, not just that the handler function exists.

**Trace session validation at the authorization boundary**, not just the logout controller: `request → session extraction → session lookup → status check → auth decision → authorization → resource access`. The realistic defect pattern: the logout handler correctly marks a session revoked in the database, but a separate component (an API gateway, a cache) trusts a cached session object without re-checking the updated status. The root cause in cases like this isn't "logout doesn't work" — it's that the *authorization path* doesn't actually consult the revocation state.

**Trace refresh-token rotation/revocation**: `refresh request → token extraction → client validation → token-family lookup → revocation/status check → rotation → new access token`. Does logout actually call the token-revocation service, or does it only touch the browser session — this single gap explains a large share of "old refresh token still works after logout" findings.

**Review OIDC logout implementation.** Does the application invalidate its own session *independently* of whether the IdP redirect succeeds (never make local termination depend on successful external round-trip); are post-logout redirect URIs restricted to an approved list; for back-channel logout specifically, is the logout token's issuer/audience/signature/required claims/session-or-subject correlation actually validated per spec, not just its presence checked.

**Review SAML SLO implementation.** Both directions (incoming and outgoing) need tracing: are logout messages signature-validated, is issuer/destination checked, is session-index correlation enforced so an IdP logout message can't terminate an unrelated user's session. The important failure-handling question: if the local session invalidates successfully but the IdP is unreachable, does the application honestly report that global logout couldn't be confirmed, or does it silently claim full success when it only achieved local termination?

**Inspect cookie/session identifier lifecycle** for rotation after authentication, server-side invalidation on logout (not just cookie deletion matching the original path/domain), and whether authentication state can ever be reconstructed from an untrusted client-side flag.

**Inspect identity mapping and account switching** — particularly relevant for apps supporting multiple IdPs or organizations: is the local account correctly bound to the external subject, does switching accounts clear the previous identity context completely, can a new login inherit another user's roles or tenant context through stale mapping.

**Review concurrency and failure paths**: a protected request in flight during logout, two logout requests racing, a refresh request racing token revocation, a delayed IdP callback arriving after local termination. The question isn't whether concurrent requests return different status codes (that's not inherently a bug) — it's whether a race can reconstruct or reactivate an invalidated session, or attach a stale identity to a new one.

## G. Differential testing

Nine-step correlation workflow: define the policy (what should logout actually terminate — local session, IdP session, refresh token, all devices, all related apps), establish the baseline (authenticated, access confirmed), execute one logout operation without changing any other variable, replay the original artifact, observe the resulting identity state, correlate internal evidence (session records, revocation events, logs, code), establish impact (what's actually still accessible), identify root cause, verify the fix with the same replay test.

| Test | Black-box | Gray-box | White-box |
|---|---|---|---|
| Local logout | Old cookie accepted/rejected | Session record and invalidation event | Logout handler and session validator |
| Global logout | Related apps remain authenticated or terminate | IdP config and SLO logs | Federated logout integration |
| Refresh-token revocation | Old token succeeds/fails | Token status and revocation event | Refresh handler and token-family logic |
| Access-token persistence | API accepts/rejects original token | Token policy and API config | Resource-server token validation |
| Silent SSO | New session issued without prompt | IdP session state and auth events | Callback and session-creation flow |
| SAML SLO | Local/federated sessions persist | SLO config and message logs | SAML logout handlers |
| OIDC logout | IdP and local session behavior differ | Client settings and logout events | RP-Initiated/back-channel logic |

## H. Exploitation and validation

Minimal proof for session persistence: replay the exact original session cookie or token against a protected resource after logout and confirm it returns genuine protected data — not a cached page, not a newly-issued session. For refresh-token findings: demonstrate the old token obtains a *new* access token after the documented revocation point. For SAML/OIDC correlation failures: demonstrate a logout message intended for one session terminates (or fails to terminate) a *different*, unrelated session.

## I. False-positive analysis

The IdP still recognizing the user doesn't mean logout is broken — the application may deliberately implement local-only logout. A still-working access token isn't automatically a vulnerability — compare against the actual documented revocation policy and token lifetime, not an assumed universal rule. A disappeared cookie doesn't prove server-side invalidation — always replay the original value. The browser's back button showing protected content is usually cached rendering, not a live bypass — reload and attempt a fresh retrieval. A `GET`-accepting logout endpoint isn't automatically vulnerable — demonstrate the actual unwanted state change. A configured SAML SLO setting doesn't prove it executes correctly — verify the message flow and resulting session state. A session record still existing in storage doesn't mean it's still usable — check its status and attempt a replay. Automatic re-login after logout could be a legitimate fresh SSO transaction, not resurrection of the old session — distinguish the session identifiers.

## J. Root cause

The dominant pattern by far: logout implemented as "clear the browser cookie" without a corresponding server-side session invalidation call, or without calling the separate token-revocation service the documented policy actually requires. The second pattern: an authorization boundary (gateway, cache, middleware) that doesn't re-check revocation status on every request, trusting a cached "authenticated" decision made before the revocation happened.

## K. Remediation

- Invalidate server-side sessions on logout independently of cookie clearing; ensure every protected route — not just the one tested — rejects the revoked session.
- Revoke refresh tokens explicitly when policy requires it; implement rotation and reuse detection.
- Make local session termination independent of successful IdP round-trip; report honestly when global/federated termination couldn't be confirmed rather than claiming full success.
- Restrict post-logout redirect URIs to an approved allowlist.
- Validate SAML/OIDC logout messages per spec (signature, issuer, audience, session correlation) so a logout event can't terminate or spare the wrong session.
- Log logout operations and revocation failures distinctly from ordinary session events, without logging the secrets themselves.

## L. Regression testing

Re-test session/token replay after any change to the session store, cache layer, or API gateway. Re-test SAML/OIDC logout correlation after any IdP configuration change or library upgrade.

## M. Finding documentation

Title, affected component (application, API, IdP, token service, or related app), expected vs. observed behavior, preconditions, sanitized reproduction (before-logout request, logout request, after-logout replay), evidence (session/token status, relevant logs/config), impact (what remains accessible and the realistic consequence), root cause, remediation, retest confirmation (the original artifact no longer works), severity rationale grounded in privilege level and exposure — a local logout correctly preserving an IdP session by design is not a finding; a revoked administrator refresh token that still obtains privileged access is a serious one, and the two should never be scored the same just because both involve "logout."

## Completion criteria

- [ ] Local application logout tested with the original session replayed against at least two different protected endpoints
- [ ] Local vs. IdP logout distinguished explicitly, with the three-way interpretation (local-only, incomplete global, or no termination at all) applied correctly
- [ ] Access token and refresh token tested separately against the documented revocation policy, not treated as equivalent
- [ ] Silent SSO re-authentication distinguished from genuine old-session survival via session identifier comparison
- [ ] MFA freshness tested on the new post-logout authentication, not assumed from the old session
- [ ] SAML SLO or OIDC logout (whichever applies) tested for session correlation, not just message acceptance
- [ ] Cross-device and cross-application logout tested against the documented scope, not a generic expectation
- [ ] Each finding traced to the specific missing invalidation call or unchecked revocation state, not reported as a generic "logout bypass"
