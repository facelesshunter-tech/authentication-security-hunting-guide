# 04 — JWT and Token Security

**Source:** `Phases checklist.txt` Phase 16 "Authentication Cryptographic Failures" (read in full, lines 20095–21503) — the JWT- and token-specific tests only (16.1–16.4, 16.14, 16.15, 16.20–16.24, and the worked example). The password-hashing, TLS, encryption-at-rest, and general key-management tests from the same source phase are reserved for Guide 15 (Cryptographic Failures) to avoid duplicating this guide. Also drawing on the course's Phase 16 for JWT structural fundamentals and `Testing Approach .txt` Phase 12.

**Explicit scope note, carried forward honestly from the source material:** deep JWT exploitation beyond algorithm confusion and the attacks below — weak-secret brute-forcing specifics, `jku`/`x5u` header manipulation chains — was flagged as outside this course's original scope in `Testing Approach .txt`. This guide covers what the source material actually supports in depth; treat anything beyond it as a research extension, not a gap in this guide's fidelity to its sources.

## A. Security objective

A token should be accepted only when its signature, algorithm, issuer, audience, expiry, and purpose all satisfy the application's trust policy — not when it merely parses correctly. **Decoding a JWT is not verification.** A three-segment base64 string and a cryptographically authenticated claim set are different things, and the entire attack surface in this guide lives in applications that conflate the two.

## B. Attack surface

Any endpoint that accepts a bearer JWT: API authentication, OAuth/OIDC ID and access tokens, internal service-to-service tokens, and any custom session-replacement scheme built on signed tokens. Also in scope: the signing-key management surface (generation, storage, rotation, retirement) and the verification middleware itself.

## C. Threat model

The attacker controls the token's content entirely — header, payload, and whatever they put where a signature should be. The only thing standing between that and an accepted forged identity is the verifier's logic. Required capability: ability to send an arbitrary token to the target endpoint (trivial, no special access needed). Required victim interaction: none. The entire question this guide answers is whether the verifier actually checks what it needs to before trusting what the attacker handed it.

## D. Black-box testing

**Identify the cryptographic artifacts first.** Complete a normal login or token exchange, inspect the resulting cookies/tokens/responses for three-dot-separated strings, and record header, payload, algorithm, issuer, audience, and key identifier for anything that looks like a JWT. A token's format doesn't prove its security — a three-segment string being present tells you nothing about whether the server actually verifies it.

**Signature verification.** Obtain a legitimate controlled test token, confirm it works normally, then modify a non-sensitive test claim without recomputing a valid signature, and replay it. The server must reject it. If it doesn't, you have a serious lead — but confirm the response represents actual authenticated access, not a public endpoint, a cache artifact, or a gateway rejecting the token before your modified version ever reached the application.

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ0ZXN0LXVzZXItMTciLCJhdWQiOiJhcGkuZXhhbXBsZS50ZXN0IiwiaXNzIjoiaHR0cHM6Ly9pZC5leGFtcGxlLnRlc3QvIiwiZXhwIjoxOTAwMDAwMDAwfQ.<signature>
```

(Illustrative only — fictional host and claims.)

**Algorithm and header handling.** This is the classic algorithm-confusion surface. Identify the accepted algorithm(s) from documentation or config, then — in an authorized test environment — check whether the server trusts an algorithm the *token itself* declares rather than enforcing its own configured policy, and whether a token-controlled key reference (`kid`, `jku`, `x5u`) can influence which key material gets used for verification. An unusual algorithm name in a header isn't itself a finding; the finding is untrusted input bypassing the server's intended verification policy.

**Issuer, audience, expiry, purpose.** Test each property independently against a cryptographically valid signature: correct issuer vs. wrong issuer, correct audience vs. wrong audience, valid vs. expired, correct token type vs. wrong type (e.g. a refresh token presented where an access token is expected). A valid signature only proves the token was signed by a key the verifier accepts — it says nothing about whether *this* token should be trusted for *this* operation. A token issued for an internal reporting API accepted by an administrative API is exactly the shape of finding this test is designed to catch.

**Cryptographic error handling.** Submit malformed structure, invalid signature, expired token, unsupported algorithm. Check specifically whether a failure ever results in the request proceeding anyway — an exception handler that falls back to decoding without verifying is a real, recurring pattern, not a hypothetical.

## E. Gray-box testing

Review token-validation configuration and available logs: is signature verification actually enabled, which algorithms are allowlisted, is the configured key source trustworthy, does a validation failure actually appear in logs, and — critically — is the token accepted at a gateway layer but reinterpreted differently (or not re-verified at all) once it reaches the application. A token passing through two layers with two different trust decisions is a common place for this class of bug to hide.

## F. White-box testing

**Trace JWT verification end to end**: `Authorization header → token parsing → trusted key selection → algorithm enforcement → signature verification → claim validation → identity mapping → authorization`. Verify a maintained library is used (never hand-rolled signature math), algorithms are explicitly restricted rather than trusted from the token's own header, keys come from an approved source that untrusted header values can't redirect, and — the single most important check — that no code path treats successful *parsing* as proof of *authenticity*.

Red flags to search for specifically: signature verification disabled or stubbed out anywhere reachable; a decode-only function used where a verify function should be; an exception handler that accepts the token after verification throws; algorithm trust taken from the token's own `alg` header; issuer/audience values accepted without comparison against an expected value.

**Trace signing-key management**: generation, storage, access permissions, key-identifier selection rules, rotation procedure, retirement of old keys, dev/prod separation, and whether an untrusted token header can influence key retrieval unsafely. A `kid` value itself is not a secret and can legitimately appear in a token — the risk is specifically when untrusted key-selection data bypasses the trusted-key policy, not the presence of a `kid` field.

**Trace key rotation and revocation**: can the system rotate without an outage, do verifiers learn about new keys only through an approved trust mechanism, and — the realistic finding here — can a key that was supposedly revoked still successfully verify a freshly-presented token because cached key material or a retirement step was incomplete. A temporarily-retained old key for compatibility can be legitimate; the question is whether that overlap is deliberate policy or an oversight.

**Inspect cryptographic comparisons** used in token/HMAC verification specifically: library verification functions, not ad hoc string equality. Constant-time comparison matters most where an attacker can make repeated comparisons against a secret and observe timing — not every comparison needs it, so don't flag every `==` reflexively.

**Inspect secret storage** for hardcoded signing secrets, private keys, or client secrets reachable from the codebase or configuration, and for development keys reused in production. For each: is it actually sensitive, is it live, who can reach it, what does it let an attacker forge.

## G. Differential testing

Nine-step method: define the required property ("a token with an invalid signature must never authenticate a user"), establish a baseline with a valid controlled token, change one cryptographic property (alter a claim without recomputing the signature), observe external behavior, inspect internal evidence (logs, config), trace the code, establish root cause (crypto misuse vs. trust config vs. claim validation vs. key management), assess the actual identity/boundary that can be bypassed, then verify the fix by repeating the test and confirming rejection.

| Test | Black-box evidence | Gray-box evidence | White-box target |
|---|---|---|---|
| Invalid JWT signature | Modified token accepted or rejected | Validation config and logs | JWT verification function |
| Wrong JWT audience | Access decision | Expected audience config | Claim-validation logic |
| Expired token | Acceptance or rejection | Token policy and timestamps | Expiry validation |
| Hardcoded signing key | External behavior often inconclusive | Secret-management evidence | Source and deployment config |
| Weak key rotation | Old-key acceptance behavior | Key inventory and rotation records | Key-selection and retirement logic |

### Worked example: modified JWT accepted

Black-box: a legitimate controlled token grants access; a harmless test claim is modified without a valid signature; the modified token still returns protected data. Confirm this is genuine authenticated access, not a public response or cache artifact. Gray-box: check whether signature verification is enabled, which algorithms are allowed, whether the key is trusted, and whether the log shows a validation failure that the application ignored. White-box: trace the middleware and find it calling a *decode* function and using the resulting claims without ever calling *verify*. Root cause: the application treats successful parsing as proof of authenticity. Remediation: use the maintained library's actual verification function, explicitly restrict algorithms, validate issuer and audience, reject on any failed check. Verification: repeat the test — legitimate token succeeds, modified token fails, no protected data returns under the forged identity, the failure is now logged.

## H. Exploitation and validation

Minimal proof: the modified-claim-without-valid-signature test above, demonstrating protected data or an elevated identity returned under the forged claim — against your own controlled account and endpoint, nothing more elaborate required. For algorithm confusion specifically: demonstrate the server actually honors an algorithm or key source the token itself dictated, not just that the header *could* be manipulated.

## I. False-positive analysis

The source material's own named mistakes, all worth checking before reporting: "the payload is readable, so it's insecure" — JWTs are encoded, not encrypted, confidentiality is a separate property from integrity; "the token is long, so it's unpredictable" — length is not entropy; "it has a signature, so it's trusted" — the signature can be invalid, the key untrusted, or the issuer/audience wrong, independently of each other; "the token is accepted, so verification is disabled" — the endpoint might be public, or another component already authenticated the request; "a `kid` in the token is a leaked secret" — a key identifier is usually meant to be public, verify whether the actual key material is exposed before claiming so; "the old key is still present, so rotation failed" — confirm whether it's still *trusted*, not just present, retained-for-compatibility is a legitimate design.

## J. Root cause

The two dominant patterns: (1) decode used where verify should be — successful parsing treated as proof of authenticity, usually via an exception handler that falls back to unsafe behavior on verification failure, and (2) trust taken from the token itself — algorithm or key source read from attacker-controlled header fields instead of enforced from server-side configuration.

## K. Remediation

- Require signature verification for every signed token on every path that accepts one — no decode-only shortcuts, no fallback-on-failure branches that proceed anyway.
- Explicitly restrict accepted algorithms server-side; never trust an algorithm declared by the token itself.
- Validate issuer, audience, expiry, and token type/purpose independently — a cryptographically valid signature is necessary but never sufficient on its own.
- Source verification keys only from an approved, server-controlled location; never let untrusted header values (`kid`, `jku`, `x5u`) redirect key retrieval.
- Keep signing secrets and private keys out of source code and shared appropriately narrowly; rotate on a defined schedule and confirm retired keys actually stop being trusted, not just stop being issued.

## L. Regression testing

Re-run the modified-claim and wrong-audience/issuer tests after any JWT library upgrade (verification defaults can change silently), and after any key-rotation event, confirm the newly-retired key is actually rejected, not just no longer actively issued.

## M. Finding documentation

Title, affected component (JWT verification, specific endpoint or service), expected vs. observed behavior, preconditions, minimal reproduction with a controlled modified token, evidence (baseline request/response, modified artifact, resulting decision, relevant config or source path), security impact, root cause, remediation, retest confirmation, and an explicit false-positive assessment showing you ruled out public endpoints, caches, and gateway-layer rejection before concluding the application itself accepted the forged token. Severity depends on what the forged or cross-audience token actually grants — note the specific identity, privileges, or service boundary crossed, not just "JWT bypass" as a label.

## Completion criteria

- [ ] Signature verification tested with a modified claim and no valid signature
- [ ] Algorithm handling tested for trust taken from the token's own header vs. server-enforced policy
- [ ] Issuer, audience, expiry, and token-type/purpose each tested independently against a validly-signed token
- [ ] Signing-key management reviewed: storage, rotation, retirement, and whether retired keys are actually rejected
- [ ] Verification code traced to confirm decode and verify are not being confused anywhere reachable
- [ ] Cryptographic failure paths checked for unsafe fallback-on-error behavior
- [ ] Each finding traced to the specific violated property (signature, algorithm trust, issuer, audience, expiry, purpose) rather than reported as a generic "JWT vulnerability"
