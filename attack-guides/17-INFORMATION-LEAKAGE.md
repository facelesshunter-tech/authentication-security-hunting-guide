# 17 — Authentication Information Leakage

**Source:** `Phases checklist.txt` Phase 12 "Authentication Information Leakage Mastery" (read in full, lines 13956–15965 — 17 BB tests, 11 GB areas, 12 WB tracing areas, 10 differential dimensions, 3 full worked case studies), reconciled with the earlier, shorter Phase 3 pass (sections 6 and 9 of which fed Guide 06's recovery-specific leakage content), `Testing Approach .txt` Phase 3, and the course's Phase 12.

**The security invariant, stated precisely because "reveal nothing" is not achievable and not the actual standard: the application should disclose only the information permitted for the caller's identity, authorization level, and current workflow state.** A successful login necessarily reveals the credentials were accepted; an authenticated profile page necessarily reveals what that user is authorized to see. Information leakage is specifically disclosure that crosses *beyond* that boundary — and the presence of information in a response is never automatically reportable on its own; sensitivity, intended audience, accessibility, and demonstrated consequence all have to be established first.

## A. Security objective

For every authentication-adjacent response — success, failure, error, redirect, header, cache entry, log line — confirm the information it carries is appropriate for the specific caller who received it, not for some more privileged or more informed caller who might exist.

## B. Attack surface

Account enumeration through login/recovery/registration, account-state and MFA-factor disclosure, timing side channels, error messages and stack traces, reset/verification token exposure, session and auth-token disclosure, OAuth/OIDC and SAML leakage paths, authorization-state and privilege leakage, HTTP headers/redirects/browser behavior, cache-related cross-user leakage, logout/session-expiry disclosure, rate-limit/lockout side channels, and cross-tenant authentication information leakage.

## C. Threat model

The attacker here typically has nothing but the ability to send requests — no credentials, no special access. The entire value of this guide is in what that unprivileged position can still learn: which accounts exist, which are protected by MFA, what internal architecture looks like, and whether any bearer secret (a token, a session ID) leaks somewhere it shouldn't. A second attacker profile: someone with read access to logs, traces, or analytics who's hunting for secrets that leaked into telemetry that was never meant to hold them.

## D. Black-box testing

**Account enumeration through login.** Compare a registered account with a wrong password against an unregistered identifier, same request shape, same everything else. Compare status code, error message, JSON structure, response length, redirects, cookies, and timing. A repeatable, externally observable distinction that identifies registered accounts is the finding — but confirm the difference is actually exposed to the relevant attacker and genuinely discloses something beyond the application's intended policy, since some applications intentionally support public account discovery.

**Enumeration through password recovery** — a particularly high-value category. Even if the *initial* response is identical ("if an account exists, instructions will be sent"), trace the *complete* user journey: a different follow-up page, a visible countdown, a different resend limit, a distinct verification status, or a notification only you (the controlled tester) would see. Response uniformity at the first step is not complete security if a later step in the same workflow leaks the same information.

**Enumeration through registration.** Check "email already registered," "account exists but unverified," "user belongs to another organization" — some disclosure here can be intentional and necessary (preventing duplicate registration), so the test is whether the specific information is appropriate for an *unauthenticated* caller, not whether any distinction exists at all.

**Account-state enumeration.** Active, disabled, locked, unverified, pending approval, password-reset-required, MFA-subject, admin-restricted — compare known test accounts in different states across login errors, recovery responses, redirects, cookies, MFA screens, and API error codes. The same disclosure can be entirely appropriate when shown to the account's own owner attempting to sign in, and inappropriate when the same message is returned to an arbitrary caller probing a list of email addresses — evaluate audience and context, not just content.

**Authentication-factor enumeration.** Does the login flow reveal "enter your six-digit code," "approve on your registered device," "we sent a code to the phone ending in 42," or "this account has no second factor"? Distinguish four meaningfully different severities: disclosure that MFA is used at all; disclosure of the specific factor type; disclosure of partial recovery-contact information; and disclosure of an actually usable authentication secret — these are not the same finding and shouldn't be reported as if they were.

**Timing-based enumeration.** Never conclude from one slow request — collect multiple measurements per condition and compare *distributions*, accounting for ordinary network and runtime variability. The realistic mechanism: an existing account triggers password-hash verification and account-state evaluation that a nonexistent account's shorter code path skips entirely. Stronger evidence is a stable, statistically meaningful, *remotely* observable difference — not a single data point.

**Error messages and stack traces.** Trigger controlled malformed input across login, recovery, verification, session validation, OAuth callbacks, SAML messages, and MFA submissions. Framework versions and internal class names are usually low-severity reconnaissance value; investigate carefully instead for a usable credential, a live token, internal secrets, protected database content, or authentication-decision details that actually enable a bypass — these categories are not interchangeable in severity.

**Reset-token and verification-token leakage.** Trace the authorized recovery/verification workflow end to end: where is the token generated, where delivered, where does it appear in application traffic, do unrelated systems ever receive it, is it scoped to its intended purpose, does it expire and invalidate correctly. A token appearing in its own intended email link is expected, not a finding — the question is whether it reaches somewhere unintended.

**Session and authentication-token disclosure.** Check HTML source, JavaScript variables, URLs, redirect destinations, error responses, debug pages, and client-side storage for session IDs, access/refresh tokens, ID tokens, API keys, recovery codes, and MFA secrets. A token-shaped string is not proof of a usable credential — establish its type, whether it's genuine or a placeholder, which principal it represents, what it authorizes, and whether it's still valid before calling it a finding.

**OAuth/OIDC leakage.** Trace the full flow for exposure of authorization codes, access/refresh/ID tokens, client secrets, and sensitive identity claims appearing somewhere unintended. An authorization code in its intended callback is normal OAuth behavior — it becomes a concern when exposed to an unauthorized party, accepted outside the intended flow, or insufficiently protected. (Cross-reference Guide 07 for the protocol-correctness angle; this is specifically about unintended exposure of otherwise-valid material.)

**SAML leakage.** Check for assertion contents, identity attributes, and internal IdP metadata reaching unintended recipients, plus whether detailed signature-validation errors expose secret configuration. An assertion existing in the expected protocol exchange is not itself a vulnerability — the concern is unintended disclosure, excessive retention, or incorrect access controls around it. (Cross-reference Guide 08.)

**Authorization-state and privilege leakage.** Can an unauthenticated or low-privilege caller discover administrator email addresses, internal user IDs, organization membership, role assignments, or administrative recovery workflows through profile endpoints, search suggestions, invitation flows, or GraphQL schemas? Knowing an administrator's email is not equivalent to gaining admin privileges — but establishing the expected confidentiality policy and demonstrated impact still matters before reporting.

**HTTP headers, redirects, and browser behavior.** Check `Location`, `Set-Cookie`, `Cache-Control`, `Referrer-Policy`, CORS headers, and trace/request identifiers for authentication-relevant leakage outside the response body. If a sensitive token sits in a URL, investigate whether navigation elsewhere could leak it via the referrer — don't assume every URL parameter reaches every destination; check the actual referrer policy and navigation path.

**Cache-related leakage.** Request a user-specific resource as Account A, then as Account B, and compare — repeating in both orders. A response being cached is not automatically a vulnerability (it may be genuinely public, or the cache may correctly isolate authorization contexts) — confirm a *private* response actually crosses to an unauthorized caller before reporting.

**Logout/session-expiry leakage.** Does the logout response differ for an active vs. already-expired session in a way that reveals prior state? A browser displaying a previously rendered page after logout is not proof the server still accepts the session — reload through the normal flow and verify the actual server-side authorization decision.

**Rate-limit and lockout response leakage.** Does a distinct lockout response for registered accounts leak existence? A rate-limit mechanism existing isn't itself a vulnerability — the question is whether its specific *behavior* leaks account information. Use small, approved test sets; never run a real lockout campaign against production.

**Cross-tenant authentication information leakage.** Can a Tenant A user discover Tenant B's user directory, SSO/MFA policy, recovery settings, or invitation status? Compare the response for an authorized tenant member against an identity with zero membership in that tenant — public IdP metadata and private organization authentication settings are not equivalent, don't conflate them.

## E. Gray-box testing

Build a complete inventory of every authentication-adjacent information source (user records, password-verification results, MFA config, recovery records, account-state flags, session records, OAuth/SAML tokens and claims, authentication events, rate-limit state) with its sensitivity, intended recipient, channel, retention, and protecting control — this inventory is the foundation for everything else in this section. Review API contracts and error schemas for sensitive fields present in the schema even if you haven't yet confirmed they're actually serialized to unauthorized callers — a schema showing a field is a lead, not proof. Search logs, traces, gateway logs, and analytics for accidentally stored passwords, session IDs, tokens, reset/verification tokens, MFA recovery codes, and full request bodies — a token appearing in a log is not automatically exploitable, confirm it's still valid, who can actually read that log, and what it authorizes. Compare documented enumeration-prevention policy against observed behavior, specifically checking whether *different routes* implement different error policies for the identical underlying condition (web login generic, mobile API returning a distinct "user not found" code) — this inconsistency is a real, specific pattern worth hunting deliberately. Review recovery/verification token configuration for whether full recovery URLs or raw tokens get accidentally recorded in operational logs. Review reverse-proxy and gateway behavior, since authentication data passes through infrastructure the application layer's own protections never touch. Review frontend/browser data handling for sensitive values entering JavaScript-accessible storage, analytics events, or third-party scripts unnecessarily. Review tenant/role-based information access to confirm administrative visibility is actually scoped — a tenant admin should see their own organization's authentication events, not every organization's. Build the complete data-flow map (identity store → auth service → session/token service → application API → browser, with branches to logs, reverse proxy, error monitoring, analytics, email/SMS provider, background processing) and trace each sensitive item's origin, storage, recipients, and whether any unnecessary copy lingers anywhere in that graph.

## F. White-box testing

**Trace authentication responses**: `request → input validation → account lookup → credential verification → account-state evaluation → MFA/verification decisions → error mapping → response serialization`. At each stage: which internal condition is being detected, does it influence the external response, is that distinction intentional, does the response carry more than required?

**Trace error handling** for exception handlers and serializers that expose internal exception text, stack traces, database details, configuration, or token-validation internals — and specifically check whether production and development error handling actually differ, since the dangerous pattern is a catch-all handler returning detailed exception text regardless of environment.

**Trace account-enumeration branches** for unknown-account, incorrect-password, disabled, locked, unverified, MFA-required, and MFA-failure paths — the implementation can legitimately use distinct *internal* conditions while presenting a generic *external* response; conversely a generic-response helper is insufficient if another layer (a header, a redirect, a timing difference, a separate response field) leaks the original condition anyway.

**Trace timing-related code paths** for genuinely different computational work between account states — password-hash verification skipped entirely for nonexistent users, an expensive lookup performed only for real accounts, different downstream service calls. Not every code-path difference is exploitable; network noise and runtime scheduling can swamp small differences — use controlled measurements to establish whether the gap is actually meaningfully distinguishable remotely.

**Trace token generation and storage** for reset/verification/session/federated tokens: generation method, entropy, scope, expiration, reuse prevention, and whether the implementation ever unnecessarily returns or logs the raw secret. The specific defect worth hunting: the same token reused across different security purposes — an email-verification token that also, unintentionally, authorizes a password reset.

**Trace URL and redirect construction** for password-reset links, verification links, OAuth redirect URIs, and post-login/logout redirects — are sensitive values inserted into URLs unnecessarily, and can an attacker-controlled destination or parameter cause authentication material to be sent somewhere unintended. Don't conflate every open redirect with information leakage automatically — establish the actual connection between the redirect and the exposed data.

**Inspect logging statements directly** for passwords, auth headers, session cookies, tokens, MFA secrets, and full request/response bodies — then confirm those statements are actually *active* in the relevant environment, and check redaction at the source rather than trusting a downstream log-viewer's masking, since the underlying storage may have already received the raw secret regardless of what the viewer displays.

**Inspect serialization and field filtering** for internal account fields leaking into ordinary-user responses, administrative fields appearing where they shouldn't, or provider claims copied through without filtering. Use explicit response models/allowlists, and base the fix on the actual intended disclosure policy rather than just stripping every unfamiliar field reflexively.

**Trace frontend and third-party data flows** into browser storage, DOM, analytics events, error-reporting payloads, and third-party scripts — is a third-party component receiving authentication information it genuinely doesn't need to function.

**Trace caching and response-header logic** for the relationship between auth cookies, authorization headers, cache keys, and CDN/application cache configuration — a response header alone doesn't prove every intermediary's actual behavior, correlate code and configuration with real observed results.

**Trace OAuth/OIDC and SAML processing** specifically for the distinction between an information-leakage finding (an internal diagnostic leaking through a validation-failure error) and a validation-flaw finding (accepting an invalid token or assertion) — these require separate remediation and shouldn't be merged into one report. (Cross-reference Guides 07 and 08 for the validation-correctness side.)

**Trace role and tenant information access** to establish both the disclosure path *and* the specific missing or incorrect access-control decision — a caller retrieving another user's auth configuration, a tenant admin reaching outside their tenant, an ordinary user receiving privileged audit data.

## G. Differential testing

Ten dimensions: account-existence (registered vs. controlled-unregistered, same workflow, same invalid input), account-state (one known property changed at a time — never change multiple properties simultaneously, or you won't know which one caused the observed difference), MFA-state, timing (multiple measurements, documented sample count/median/variability, no invented universal threshold), response-channel (headers, cookies, redirects, size, cache behavior, follow-up endpoints — not just the body text), token-flow (compare a token's intended path against every place it actually appears), identity (anonymous vs. ordinary user vs. owner vs. tenant admin vs. platform admin — the expected response legitimately differs by caller, the question is whether each one gets exactly what policy allows), interface (web/mobile/REST/GraphQL/federated for the same workflow), environment (staging vs. production, never inferring production exposure from a dev-only behavior without separately verifying it), and before/after a controlled state transition (reset, verification, MFA enrollment, session revocation, account disabling — checking whether stale information remains exposed past the transition).

## H. Worked examples

**Password-reset account enumeration**: black-box shows a repeatable distinction between registered and controlled-unregistered identifiers at the reset endpoint; gray-box confirms documentation explicitly states account existence shouldn't be disclosed to unauthenticated callers; white-box shows the recovery handler branching its external response directly on the account-lookup result. Root cause: the external response directly reflects a sensitive internal distinction. This supports an enumeration finding — it does not by itself establish account takeover; don't inflate the severity past what's demonstrated.

**Reset-token exposure in logs**: a controlled token observed in a generated URL is not alone a finding (the token may be required for the flow) — but gray-box log inspection showing the *full URL* stored in a log system accessible to a broader group than intended, combined with white-box confirmation that the handler logs the complete URL rather than a redacted form, together establish a real finding. This is meaningfully stronger evidence than simply finding a token-shaped string in a response.

**Timing-based account enumeration**: black-box repeated measurements suggest a distinguishable timing distribution for existing accounts; gray-box tracing shows the known-account path performs additional processing; white-box confirms password-hash verification is skipped entirely for nonexistent accounts. The remaining question that determines whether this is reportable: is the difference *sufficiently stable and remotely observable* under realistic conditions — a code-path difference existing in source is not sufficient on its own without that practical side-channel evaluation.

## I. Exploitation and validation

Minimal proof for enumeration: the repeatable, multi-channel comparison (body, headers, timing, follow-up behavior) across a controlled registered vs. unregistered pair, demonstrating the distinction survives even when the primary message text is identical. Minimal proof for token leakage: trace the specific token from generation through to the unintended destination, with confirmation of its actual validity and scope — not just its presence somewhere unexpected. Minimal proof for cross-tenant leakage: demonstrate the specific private field reaching a caller with zero legitimate membership in the owning tenant.

## J. False-positive analysis

Different error messages don't automatically mean a vulnerability — the difference may be intentional or already public; establish the actual confidentiality policy. Identical messages don't mean enumeration is impossible — headers, timing, size, redirects, cookies, and follow-up behavior can still differ even when the visible text matches. One slow request proves nothing about timing — use repeated measurements and compare distributions. A token-shaped string isn't automatically a credential leak — it may be a placeholder, expired, or a non-secret identifier. Not every stack trace is high severity — assess what's actually disclosed and what an attacker can realistically do with it. Technology-disclosure headers aren't automatically reportable bugs — apply the program's actual criteria rather than inflating low-impact reconnaissance. A reset token in its own intended link isn't automatically vulnerable — investigate unintended recipients and logging instead. A private response appearing in browser history doesn't prove server-side access — verify the current authorization decision with a fresh request. An administrator seeing extra data isn't automatically a vulnerability — they may be explicitly authorized for exactly that scope. A source-code weakness doesn't prove external exploitability on its own — trace the complete path and confirm the code is actually reachable in the relevant environment.

## K. Root cause

The two dominant patterns across this entire guide: an external response that directly mirrors an internal state distinction the policy intended to keep private (the account-lookup result flowing straight into the error message or timing behavior), and sensitive material logged, cached, or serialized in a broader-than-intended scope because "we might need it for debugging" outweighed "this shouldn't be recorded at all" during implementation.

## L. Remediation

- Return only the information the caller's specific identity and workflow state actually requires; use generic responses wherever account existence or state must remain private, and verify the uniformity holds across the *entire* workflow, not just the first response.
- Normalize the full response path — status, body, redirects, headers, cookies, timing, follow-up behavior — not just the error-message text, since any one of those channels alone can leak the distinction the others successfully hide.
- Keep bearer secrets (tokens, session IDs, recovery codes) out of unnecessary response fields, URLs, and storage locations; apply correct scope, expiry, and single-use rules.
- Redact credentials and tokens at the source before they ever reach logs, traces, or analytics — not just at the display/viewer layer.
- Apply consistent privacy rules across every authentication workflow (registration, login, recovery, verification, MFA, federated login) — a hardened primary login doesn't compensate for a leaky recovery endpoint.
- Scope administrative and cross-tenant visibility explicitly rather than by default.

## M. Finding documentation

Title (the specific information disclosed and affected workflow), affected component, attacker prerequisites (anonymous, ordinary account, tenant membership), precisely what sensitive information is revealed, expected vs. observed behavior, evidence across whichever of black/gray/white-box layers were available, practical impact (what an unauthorized party can actually learn or do — not a hypothetical worst case), root cause, remediation, regression test, severity rationale grounded in realistic exploitability, and explicit limitations (what couldn't be verified, such as a token's actual usability or whether the issue reproduces in production).

## Completion criteria

- [ ] Login, recovery, and registration enumeration tested across the full response (body, status, headers, timing, follow-up pages), not just the primary message text
- [ ] Account-state and MFA-factor disclosure tested with the four-tier severity distinction (existence vs. factor type vs. partial contact info vs. usable secret) applied correctly
- [ ] Timing differences tested with repeated measurements and distribution comparison, never a single request
- [ ] Reset/verification tokens traced from generation through every channel they touch, including logs and analytics
- [ ] Session/auth-token exposure checked across HTML, JS state, URLs, and client-side storage
- [ ] Cross-tenant and cross-role information access tested with a genuinely unauthorized identity, not assumed from documentation
- [ ] Logs and traces searched directly for unredacted secrets at the storage layer, not just the viewer layer
- [ ] Each finding traced to the specific channel and root cause, with practical impact demonstrated rather than assumed from the information's mere presence
