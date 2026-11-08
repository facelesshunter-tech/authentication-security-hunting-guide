# 12 — Legacy Authentication Security

**Source:** `Phases checklist.txt` Phase 4 "Legacy Authentication Security" (read in full, lines 3956–4961) reconciled with the genuinely additional categories from Phase 13 "Legacy Authentication Security Mastery" (BB-08 through BB-13, read in full: legacy API keys, Basic authentication, legacy OAuth/OIDC/SAML integrations, legacy admin routes, deprecated routes/HTTP methods, and fallback/downgrade behavior), plus `Testing Approach .txt` Phase 11 and the course's Phase 13.

**Core principle, stated directly by the source and worth repeating before testing anything here: a legacy endpoint is not automatically vulnerable because it is old.** It becomes a security concern only when it violates the current security policy or exposes a weakness the modern path has already closed. Age is a lead, never a finding by itself.

## A. Security objective

Every authentication pathway reaching an account — current, deprecated, mobile-only, partner-integration, or compatibility-fallback — must enforce the same security policy the modern route enforces, unless a documented, deliberately approved exception exists. The finding is never "the endpoint is old"; it's "this pathway permits a state transition the current policy prohibits."

## B. Attack surface

Legacy login endpoints, deprecated API versions, old mobile authentication routes, HTTP Basic authentication and other older mechanisms, legacy session cookies/token formats, legacy password-reset handlers, old SSO/federated integrations, forgotten administrative or partner login routes, legacy API keys and service credentials, and any fallback/downgrade path the application uses when its modern authentication service is unavailable.

## C. Threat model

The realistic attacker simply targets the weakest reachable pathway to the same account, rather than attacking the hardened modern one directly. No special capability is required beyond discovering that an old route still responds — the value of the attack comes entirely from a policy gap between routes, not from any novel technique against the legacy mechanism itself.

## D. Black-box testing

**Discover actual legacy entry points first** — through authorized documentation, frontend/mobile code, API specs, and source if available. Don't assume a route exists from a guessed pattern (`/api/v1/login`, `/legacy/auth`, `/admin/login`); confirm it actually responds and is officially or unofficially reachable before testing it.

**Modern-vs-legacy login differential.** Establish the current endpoint's behavior for correct/incorrect credentials, unverified accounts, restricted accounts, and MFA-enabled accounts — then repeat every one of those exact cases against the legacy route with the *same* controlled identity. The finding worth looking for specifically: the modern endpoint returns an MFA challenge after password verification, while the legacy endpoint issues a fully authenticated session for the same MFA-enabled account.

**MFA enforcement on legacy paths.** Compare both pathways stage by stage: password submission, password acceptance, MFA challenge creation, MFA completion, session issuance, protected-resource access. A response merely *containing* an MFA flag is not sufficient evidence — validate the resulting session's actual capabilities against a real protected resource.

**Rate limiting and credential-abuse controls.** Compare attempt limits, recovery-request limits, and lockout behavior between modern and legacy paths using small, controlled attempt counts — never a real guessing campaign. A difference in limits alone isn't automatically a finding; the question is whether the legacy path's weaker limit actually enables meaningful abuse.

**Account-state enforcement.** Test unverified, disabled, suspended, and MFA-required accounts against the legacy route. The concrete finding: the modern endpoint correctly refuses a disabled account, but the legacy route still issues a usable session — investigate whether it's reading stale account state, a different identity store, or an incomplete migration.

**Legacy session cookies and token formats.** Compare cookie names/attributes, token formats, lifetimes, and rotation between modern and legacy sessions, then test each against the same protected resource and verify whether logout/revocation actually invalidates both. A different cookie name or token format is not itself a finding.

**Legacy logout and revocation.** Create a session via the legacy route, confirm protected access, log out through the documented endpoint, then send a *fresh* request with the same old session. The realistic finding: the modern system revokes sessions correctly on logout, but a legacy session survives because it's validated through a separate, unconnected session store.

**Legacy password recovery.** Compare token lifecycle (expiration, single-use, purpose binding) between modern and legacy recovery handlers using only a token generated for your own test account. Also check whether a password reset through the modern flow actually invalidates sessions created through the legacy flow.

**Legacy authentication protocols** (HTTP Basic, deprecated API keys, legacy bearer formats, older SSO integrations): are credentials transmitted only over protected transport, are account restrictions actually checked, is MFA required where policy mandates it, can the deprecated mechanism issue a *stronger* session than intended.

**Legacy API keys and service authentication.** Determine which identity a key represents, whether it's scoped to a tenant or service, whether expiration/revocation/rotation actually work, and whether a key can perform operations beyond its documented scope. A long-lived API key isn't automatically a vulnerability — many legitimate service integrations need one; assess scope, storage, rotation, and transport security instead.

**HTTP Basic authentication and similarly old mechanisms.** Basic's credentials are encoded, not encrypted by the scheme itself, so transport security is the whole ballgame here — confirm it's permitted only over HTTPS, credentials aren't leaking into logs or diagnostics, and MFA or other mandatory controls still apply to identities authenticated this way.

**Legacy OAuth/OIDC/SAML integrations.** Compare an older federated integration's token/issuer/audience/signature/redirect-URI validation against the current one — does the old integration accept assertions the modern one would reject, skip a required validation step, or create a session with excessive privileges. This overlaps directly with Guides 07 and 08; the distinct angle here is specifically *drift between the old and new integration*, not the protocol in isolation.

**Legacy administrative and internal routes.** Older admin interfaces, support tools, or impersonation workflows can retain broad privileges after the main application modernizes around them — check whether they use the expected authentication mechanism, require the appropriate role, enforce MFA/step-up where policy demands it, and log actions.

**Deprecated routes and HTTP method differences.** A route deprecated in documentation may still be reachable through the gateway. Check whether equivalent functionality exists through an undocumented method or path that doesn't carry the same authorization policy as the documented one. An unexpected method response alone is not proof of a flaw — the issue is a reachable method performing a protected operation without the required controls.

**Authentication fallback and downgrade behavior** — this deserves its own deliberate test, not just incidental discovery. When the modern authentication service is unavailable or a client can't complete the preferred flow, does the application fall back to password-only auth, an older token format, or a compatibility endpoint? Critically: can a *normal* client simply force this weaker mode on demand, rather than it only triggering during genuine service degradation? A fallback path existing isn't automatically unsafe — the issue is an *unauthorized* downgrade in assurance that a normal client can trigger at will.

## E. Gray-box testing

Build a version/mechanism inventory from API documentation, deprecation notices, and migration docs: status (supported/deprecated/end-of-life), authentication mechanism, required controls, and actual external result — don't assume a route is dead just because its path contains `v1`. Review architecture for duplicated authentication logic specifically — the realistic risk pattern is the modern route delegating to centralized middleware while the legacy route performs its own credential verification and quietly omits a required check. Compare logs between modern and legacy routes for the same event types (successful/failed auth, MFA completion, account restrictions) — a legacy route producing no corresponding events is a lead for bypassing the intended abuse-detection system, though absence of a log entry alone doesn't prove the control itself is missing. Review feature flags and compatibility settings for legacy routes left enabled without a current business justification.

## F. White-box testing

**Trace legacy login to session issuance**: `request → legacy route → input validation → credential verification → account-state checks → MFA evaluation → session/token issuance → authorization`. At each stage, compare directly against the current route's equivalent stage. The high-value investigation lead: the modern route delegates to a shared service while the legacy route directly checks a password hash and issues a session itself — duplicated logic drifts, and that drift is where the actual defect usually lives.

**Review authentication middleware coverage** on legacy routes specifically — authentication required, account active, verification completed, MFA completed, session not revoked, current role/tenant membership, rate limiting. Don't assume a legacy route inherits controls from a neighboring modern route; check its actual middleware chain.

**Review token/session validation** for legacy formats — does the application still accept credentials that *should* have expired or been invalidated during a migration that happened for the modern format but never touched the legacy one.

**Review password-hash migration.** Does the legacy authentication route accidentally preserve a weaker credential-verification path that bypasses the upgrade-on-login logic the modern route uses?

**Review session creation/revocation code** for separate session stores between modern and legacy paths, inconsistent lifetimes, missing revocation checks, or logout that only invalidates one session format while leaving the other usable.

**Review legacy recovery handlers** for the same token-lifecycle properties as Guide 06 (expiration, account binding, single-use), specifically comparing whether the legacy handler enforces them as strictly as the current one.

**Review legacy API-key and service-account logic**, and fallback/downgrade code branches specifically — for each compatibility exception or feature flag found: why does it exist, which users can trigger it, is it still required, is it appropriately restricted, and can it be safely removed or consolidated into the current path.

## G. Differential testing

The primary variable throughout this guide is the authentication pathway itself. For every test pair (modern vs. legacy): correct password, incorrect password, MFA-enabled account, disabled account, unverified account, repeated invalid attempts, password reset, logout, password change, session expiration, protected-resource access — same controlled identity, same inputs, both paths, compare the resulting security property, not just the response.

| Test | Modern path | Legacy path | Security property |
|---|---|---|---|
| MFA-enabled account | Record result | Record result | MFA enforcement |
| Disabled account | Record result | Record result | Account-state enforcement |
| Repeated invalid attempts | Record result | Record result | Abuse protection |
| Logout | Record result | Record result | Session revocation |
| Password change | Record result | Record result | Session invalidation policy |
| Service/API key | Record result | Record result | Credential scope and revocation |
| Fallback trigger | Normal mode | Forced fallback | Assurance-level preservation |

## H. Exploitation and validation

Minimal proof for any finding in this guide: demonstrate the specific policy difference is reproducible with the same controlled identity across both paths, and that the legacy path's weaker behavior actually produces a usable session or credential against a real protected resource — not just a differing status code or response shape.

## I. False-positive analysis

The endpoint being old is not proof of anything — establish the actual weakness. A different token format can be entirely legitimate — check validation, scope, lifetime, and revocation, not the format's age. A different status code may be intentional. An undocumented route isn't automatically unauthorized or vulnerable. Not every operation or integration needs MFA — compare against the applicable policy, not a blanket assumption. Apparent logout failure is often just cached content — always verify with a fresh server-validated request. A deprecated API remaining accessible doesn't automatically require immediate removal — determine whether it's still supported and securely maintained. Different source code between modern and legacy paths is not itself a flaw — identify the specific missing control or exploitable policy violation.

## J. Root cause

The dominant pattern across nearly every finding in this guide: duplicated authentication logic that was correct at the time the legacy route was written, then drifted as the modern route's security policy evolved (new MFA requirements, new rate limits, new revocation behavior) without the legacy path receiving the same updates. The second pattern: a fallback/compatibility path designed for genuine service degradation that a normal client can simply trigger on demand, turning an operational safety net into a standing downgrade option.

## K. Remediation

- Centralize authentication-policy enforcement in shared middleware/services wherever feasible, and remove duplicated logic that can drift.
- Ensure every legacy route enforces current account-state, MFA, and rate-limit requirements — no compatibility exception should silently weaken these.
- Use consistent session validation and revocation across every session/token format the application still accepts.
- Scope legacy API keys and service credentials narrowly, with working expiration and revocation.
- Restrict fallback/downgrade behavior so it only triggers under genuine, server-verified degradation conditions — never on a normal client's demand.
- Set and follow an actual deprecation and retirement plan for routes that no longer have a business justification, rather than leaving them reachable indefinitely.

## L. Regression testing

Re-run the full modern-vs-legacy differential matrix after any change to the modern authentication policy (new MFA requirement, new rate limit, new revocation behavior) — that's exactly the moment legacy paths silently fall behind. Re-test fallback behavior after any change to the modern service's availability/health-check logic.

## M. Finding documentation

Title, summary (how legacy behavior differs from the required policy), affected endpoint, preconditions, modern-vs-legacy comparison table, sanitized evidence (requests, responses, session behavior), expected vs. actual behavior, impact (realistic, not inflated from "it's legacy" alone), root cause (missing middleware, stale state check, duplicated logic drift), remediation, severity per the program's actual criteria, and retest confirmation across *both* pathways.

## Completion criteria

- [ ] Actual legacy entry points discovered and confirmed reachable, not assumed from guessed patterns
- [ ] Modern-vs-legacy differential run across login, MFA, account-state, rate-limiting, session, and recovery behavior
- [ ] Legacy API keys/service credentials tested for scope and revocation
- [ ] HTTP Basic or other older mechanisms tested for transport security and MFA applicability
- [ ] Legacy federated integrations compared against the current OAuth/OIDC/SAML implementation for validation drift
- [ ] Legacy admin/internal routes tested for the same role and MFA requirements as current ones
- [ ] Fallback/downgrade behavior specifically tested for whether a normal client can trigger it on demand
- [ ] Each finding traced to the specific policy difference and root-cause drift, not reported merely because a route is old
