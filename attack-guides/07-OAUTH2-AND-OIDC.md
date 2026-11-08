# 07 — OAuth 2.0 and OpenID Connect

**Source:** `Phases checklist.txt` Phase 7 "OAuth 2.0 and OpenID Connect Security" (read in full, lines 7582–8770 — 11 BB tests, 4 GB tests, 8 WB tests), `Testing Approach .txt` Phase 10, and the course's Phase 7.

Keep the distinction sharp before testing anything: **OAuth 2.0 is an authorization framework** (a client gets limited access to resources), **OIDC adds an identity layer on top** (a client authenticates a user based on an identity assertion). An OAuth access token is not proof of identity. Confusing access tokens, ID tokens, and the application's own session is the root of most findings in this guide.

## A. Security objective

The client must correctly establish four things before issuing a session: which authorization transaction it initiated, which provider actually issued the response, which client the identity assertion was intended for, and which local account should be authenticated as a result. A successful redirect is not proof of successful authentication; a decoded token is not proof of a valid identity; a valid provider login is not proof the account-linking decision is correct.

## B. Attack surface

Client application, OAuth authorization server, OpenID Provider, callback endpoints, token endpoint, UserInfo endpoint, account-linking endpoints, session/logout mechanisms, mobile deep-link callbacks, and any legacy or alternate federation flow. Testing the client doesn't automatically authorize testing the identity provider itself — confirm the boundary before you start.

## C. Threat model

The realistic attacker here rarely needs to break cryptography. They need to: inject their own authorization response into someone else's transaction (login CSRF via missing/weak `state`), redirect a sensitive response somewhere they control (redirect URI or open-redirect abuse), present an identity assertion that's valid but wrong for this context (wrong issuer, wrong audience, replayed nonce), or exploit a linking flow that trusts an unverified claim (usually email) to attach their identity to someone else's account. Required capability is typically just the ability to initiate or intercept a browser flow — no special access needed.

## D. Black-box testing

**Map the flow first.** Capture the full redirect to the IdP, the authorization request parameters (`response_type`, `client_id`, `redirect_uri`, `scope`, `state`, `nonce`, `code_challenge`/`code_challenge_method`), the callback, and exactly when the application establishes its session. A parameter missing from one captured request isn't automatically a finding — confirm the flow type and whether validation happens elsewhere before concluding anything.

**`state` validation.** Record a legitimate state value, complete the flow once as a baseline, then repeat with it altered, missing, expired, or mismatched. The application must reject all of these. A state value simply *appearing* in a URL proves nothing — the question is whether it's actually validated and bound to the correct transaction.

**Authorization-code handling.** Confirm a code is only exchanged through the real token endpoint, test whether a redeemed code works a second time, test expired/invalid codes, and confirm a failed exchange never results in an authenticated session anyway. Codes legitimately appear in callback URLs in standard flows — that alone isn't the finding; reuse or unauthorized redemption is.

**PKCE enforcement.** Confirm a challenge is present and note its method (`S256` is the one to expect). Where the test environment allows it, compare a correct verifier against an incorrect one at the token endpoint — presence of a PKCE parameter is not the same as the server actually enforcing it.

**Redirect URI validation.** Compare the registered URI against what's actually accepted; probe for loose wildcard matching that would let an authorization response get diverted; check whether the client itself might forward a sensitive response somewhere unintended via an open redirect. Never send a real authorization code to a third-party destination to test this.

**OIDC ID-token validation.** Within an authorized test harness, test rejection of invalid, expired, or wrong-audience identity assertions, and confirm a failed validation never results in a session anyway. A token payload visible in the browser is not evidence either way about what the server actually validated — most validation happens server-side and isn't directly observable.

**Issuer, audience, and nonce**, tested independently against an otherwise-valid signature: does the client accept an assertion from an unexpected issuer; does it accept a token not intended for this client; does it validate the nonce per the flow's actual requirements (not every flow has identical nonce requirements — confirm which one applies before testing).

**Account linking and identity confusion.** Using two accounts you control: does linking require the user to already be authenticated to the target account; does the application verify actual control of the external identity, or does an unverified email claim alone trigger linking; can a linking callback get attached to the wrong local session. Automatic linking by email is not automatically a vulnerability — it depends on the specific provider's email-verification guarantee and the application's trust policy; establish that before calling it a finding.

**Token leakage.** Check callback URLs, redirect chains, browser history, referrer behavior, logs, analytics, error messages, and third-party scripts for access tokens, refresh tokens, ID tokens, or authorization codes. A token appearing in a browser-visible location doesn't automatically mean it's exploitable — establish its validity, audience, scope, and actual exposure path.

**Access/refresh token handling.** Does the resource server enforce the correct audience/scope on access tokens; does it ever mistake an access token for proof of identity (a distinct error from trusting an ID token); for refresh tokens, is client binding enforced, is rotation/reuse-detection implemented where the design calls for it, does revocation actually work.

**Federated logout.** Log out of the application, then send a fresh request to a protected resource. Application logout and identity-provider logout are different operations — the application not re-prompting for IdP credentials on the next login is not itself a bug unless the application's own documented policy promises provider-level single sign-out.

## E. Gray-box testing

Compare the intended client/provider configuration (registered redirect URIs, allowed response types/scopes, PKCE requirement, signing-key config) against the actual deployed values — environment-specific overrides between dev/staging/production are the realistic place this drifts. Review OIDC discovery metadata and JWKS configuration: is the issuer validated against the expected value, are signing keys actually sourced from trusted provider metadata, is the accepted algorithm policy appropriate, can untrusted configuration influence which keys get used for validation. Where authorized, inspect token/session records for how the external identity is actually keyed (issuer+subject vs. email alone) and whether the application session gets created before or after validation actually finalizes.

## F. White-box testing

**Trace the callback end to end**: `callback handler → state validation → authorization-response validation → token exchange → token validation → identity mapping → application-session creation`. The realistic defect: a session gets created even when state validation or token exchange fails somewhere along that chain — confirm the vulnerable path is actually reachable and the resulting session actually grants access before reporting.

**State/nonce implementation.** The common defect: code checks only that a state parameter *exists*, never that it matches the original transaction. Presence is not validation.

**PKCE implementation.** Trace verifier generation, challenge derivation/transmission, storage, and — critically — whether the authorization server actually enforces the verifier at exchange time rather than just accepting whatever challenge method was declared.

**Redirect URI matching.** Check the actual comparison logic for parsing differences that could produce an unintended match (trailing slashes, case sensitivity, subdomain wildcards interpreted more broadly than intended), and confirm the redirect decision is made by the trusted authorization server, not unsafe client-side logic.

**ID-token validation — the core defect pattern:**

```text
decode the JWT, read the claims, trust them
```

versus actually verifying signature, issuer, audience, expiration, and nonce. Decoding is not validating — this is the same confusion as Guide 04, specific to the OIDC context here. Also confirm validation absence in one function doesn't just mean it happens in middleware or a trusted library earlier in the path — trace the complete request flow before concluding it's missing.

**Identity mapping and account linking.** Is the external identity bound to issuer+subject (correct) or email alone (fragile, depends entirely on the provider's verification guarantee)? Can a callback get associated with the wrong local account? Can identity records be overwritten or merged unexpectedly through an unauthorized path?

**Token storage.** Trace token data through server-side storage, browser storage, cookies, logs, analytics, redirects, and error handlers — identify every component that can read each token, and whether refresh tokens get meaningfully stronger protection than short-lived access tokens where the design calls for it.

**Refresh and revocation.** Can a revoked refresh token still obtain a new access token; is reuse detection implemented where the design requires it; are related token families revoked together per policy.

## G. Differential testing

Seven-step method: define the requirement ("the client must reject an assertion whose audience doesn't include this client"), establish a valid baseline, change one variable (swap in a wrong-audience assertion), observe the result, inspect internal config (confirm the expected client ID/issuer/validation profile), trace the code (where should the audience check happen, does it), establish impact (does the invalid assertion actually grant a session).

| Test area | Black-box | Gray-box | White-box |
|---|---|---|---|
| State | Valid vs. mismatched transaction | Flow configuration and logs | Generation and validation code |
| PKCE | Valid vs. invalid verifier | Client/provider settings | Verifier binding and enforcement |
| Redirect URI | Observed callback destinations | Registrations vs. deployed settings | Matching and redirect logic |
| ID-token validation | Valid vs. invalid assertions | Issuer and key configuration | Signature and claim validation |
| Account linking | Controlled linking outcomes | Identity records and policy | Identity mapping and authorization |
| Token leakage | Responses and redirect chains | Logs and telemetry | Token data flow |
| Refresh/revocation | Controlled credential lifecycle | Records and policy | Refresh and revocation logic |

## H. Exploitation and validation

Minimal proof for state/CSRF: demonstrate a response not bound to the originating transaction still establishes a session, using controlled accounts in two separate browser contexts. For ID-token validation: demonstrate a wrong-issuer or wrong-audience assertion (still correctly signed by the real provider for a *different* legitimate purpose) gets accepted anyway. For linking: demonstrate an external identity you control gets attached to an application account without the verification the policy claims to require.

## I. False-positive analysis

A missing state parameter in one captured request isn't a finding without confirming the actual flow and requirement. OAuth alone doesn't establish authentication — confirm whether OIDC or another explicit identity mechanism is actually in use before assuming "login" means "authenticated." A JWT-format access token proves nothing about whether its claims are trusted. A decoded claim containing an email doesn't prove the application trusts or acts on it. An unexpected redirect isn't a finding until you confirm a sensitive credential can actually be diverted through it. A provider session persisting after application logout is expected — local and provider logout are different guarantees, test against what the application actually documents. Two identities sharing an email address doesn't prove account takeover — check issuer, subject, and the provider's verification guarantee first.

## J. Root cause

The recurring patterns: validation parameters that are checked for *presence* rather than *correctness* (state, nonce); decode-without-verify applied to ID tokens exactly as it applies to raw JWTs in Guide 04; identity linking keyed on an unstable or unverified claim (email) instead of the stable issuer+subject pair; and session creation that happens before, or independent of, the validation result actually finalizing.

## K. Remediation

- Generate and validate `state` and `nonce` as actual transaction-binding checks, not just presence checks.
- Enforce PKCE verifier matching server-side at the token endpoint, regardless of what the client claims to have sent.
- Validate redirect URIs against an exact registered value; eliminate loose wildcard matching and any open-redirect path that could divert an authorization response.
- Validate ID tokens completely — signature, issuer, audience, expiration, nonce — using a maintained library's actual verify function, never decode-and-trust.
- Bind external identities using issuer+subject, never email alone; require proof of control before linking, and keep linking/unlinking events audited.
- Minimize token exposure in logs, URLs, and analytics; enforce refresh-token rotation and revocation per the chosen security profile.

## L. Regression testing

Re-test state/PKCE/redirect-URI enforcement after any OAuth library upgrade (defaults and validation strictness can change). Re-test account linking after any change to the identity-mapping logic or provider configuration. Re-test ID-token validation after any signing-key rotation on the provider side.

## M. Finding documentation

Title, affected component (client, provider, callback, token endpoint, or linking flow), expected vs. observed behavior, preconditions, sanitized reproduction, evidence (authorization request, callback behavior, token-validation result, relevant config or source path, resulting session behavior), impact, root cause, remediation, retest criteria. Never include reusable authorization codes, access/refresh tokens, client secrets, or live session cookies in a report. Rate severity by demonstrated impact, not protocol terminology — a protocol deviation with no demonstrated security consequence doesn't automatically earn a high severity.

## Completion criteria

- [ ] Full flow mapped, including when the application session actually gets created
- [ ] `state` tested as a transaction-binding check (mismatched/missing/replayed), not just confirmed present
- [ ] PKCE tested with an incorrect verifier at the token endpoint, where the environment allows it
- [ ] Redirect URI matching tested for loose wildcard or parsing-based bypass
- [ ] ID-token issuer, audience, expiration, and nonce tested independently against a validly-signed token
- [ ] Account linking tested for proof-of-control requirements, not assumed from documentation
- [ ] Token leakage checked across logs, URLs, analytics, and redirect chains
- [ ] Application logout vs. provider logout tested against the application's own documented guarantee, not a generic expectation
