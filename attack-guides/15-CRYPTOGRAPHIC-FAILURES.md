# 15 — Authentication Cryptographic Failures (TLS, Encryption, Key Management, Randomness)

**Source:** `Phases checklist.txt` Phase 16 "Authentication Cryptographic Failures" (read in full, lines 20095–21503, as part of building Guide 04) — specifically the general-purpose tests not already covered there: 16.7 (TLS/certificate validation), 16.8 (encryption of authentication secrets at rest), 16.9 (cryptographic error handling, general), 16.13 (CSPRNG for all security-sensitive values, not just sessions/tokens), 16.16–16.17 (encryption key management, TLS configuration), 16.18 (crypto integration in OAuth/OIDC/SAML), 16.22–16.24 (cryptographic comparisons, failure paths, secret storage, general angle) — plus `Testing Approach .txt` Phase 12 and the course's Phase 16.

**Guide 04 already covers JWT/token signature verification and signing-key lifecycle in depth — this guide covers everything else cryptographic in the authentication stack: transport security, encryption of secrets at rest, general randomness quality, key management beyond signing keys, and the cryptographic error-handling discipline that applies across all of it.** The central invariant from Guide 04 still governs here: **assess the complete security property, not merely the algorithm's name.** HMAC-SHA-256 is a strong construction with a hardcoded secret; RSA signatures are secure right up until a code path skips verifying them; AES provides strong encryption with incorrect key management or unauthenticated ciphertext handling.

## A. Security objective

Every cryptographic mechanism protecting authentication state — transport encryption, secrets at rest, random value generation, key storage and rotation — must actually deliver the specific security property (confidentiality, integrity, authenticity, unpredictability, freshness, trust validation) the application depends on it for, verified in its actual deployed configuration, not assumed from the algorithm's name or a config file that may not reflect the real runtime path.

## B. Attack surface

TLS/certificate validation on every authentication-relevant connection (login, token exchange, auth callbacks, password recovery — both client-facing and service-to-service), encryption of recoverable authentication secrets at rest, generation of every security-sensitive random value (not just sessions and tokens — MFA recovery codes, OAuth `state`/`nonce`, WebAuthn challenges, API credentials all qualify), key storage and rotation for encryption keys specifically (distinct from the signing keys covered in Guide 04), and cryptographic failure/fallback paths across the whole authentication stack.

## C. Threat model

Two attacker postures matter here. One intercepts or manipulates network traffic, relying on weak or absent transport validation to read or alter authentication material in transit. The other has — or gains — read access to stored data (a breach, a backup, an exposed config) and is testing whether that access converts into something usable: recoverable plaintext secrets, a hardcoded key that unlocks encrypted data at rest, or predictable values that let them forge what should be unforgeable. Required capability ranges from network position (interception) to storage/backup access (data-at-rest attacks).

## D. Black-box testing

**Identify cryptographic artifacts.** Complete a normal login and inspect every resulting cookie, token, and protocol metadata exchange for JWTs, SAML assertions, OAuth/OIDC discovery documents, and certificate chains. A token's format proves nothing about its actual validation — note which properties require internal evidence rather than guessing from appearance.

**TLS and certificate behavior.** Identify every endpoint handling login, token exchange, authentication callbacks, and password recovery; check whether credentials can be sent over unprotected HTTP at all, and review any HTTP-to-HTTPS redirect behavior for a window where credentials could leak. You generally can't black-box test backend-to-backend certificate validation directly — note it as requiring gray-box/white-box evidence rather than guessing.

**Encryption of secrets at rest** (black-box angle): inspect authorized responses, error details, and browser-visible state for recoverable authentication secrets that shouldn't be retrievable at all — if a secret is genuinely supposed to be one-way (passwords, most tokens), its *absence* from any response is expected, not evidence of secure storage; this question mostly requires gray-box/white-box evidence.

**Cryptographic error handling.** Submit malformed or invalid authentication artifacts appropriate to whatever protocol is in play — malformed JWT, invalid signature, expired token, invalid SAML signature, invalid certificate chain. For each: is authentication cleanly rejected, does the error response leak internal cryptographic details, and — the most important check — does any failure cause the application to fall back to a *weaker* validation path instead of simply rejecting?

**Randomness quality — general, beyond sessions/tokens.** Where you can observe multiple instances of a value (MFA recovery codes, OAuth `state`/`nonce` across several login attempts, WebAuthn challenges), check for visible structure: timestamps, counters, usernames embedded, or obvious sequential patterns. Remember the limitation stated directly in the source material: **length is not entropy** — a long value can still be predictable if it's generated from a timestamp or counter, and you cannot reliably establish true unpredictability from a small black-box sample; treat apparent randomness as a lead requiring gray-box/white-box confirmation, never a conclusion on its own.

## E. Gray-box testing

**Encryption-at-rest review**, where authorized: identify which authentication-related values the design requires to be *recoverable* (distinct from passwords, which should never be) versus one-way hashed, and confirm recoverable secrets are actually encrypted rather than falling back to plaintext storage. Check whether encryption keys are stored *separately* from the ciphertext they protect — keys sitting alongside their own ciphertext with no access separation defeats the purpose regardless of the algorithm's strength.

**Randomness implementation review**, general: for every security-sensitive value category — session identifiers, reset/verification tokens, MFA recovery codes, API credentials, nonces, challenge values, OAuth `state`/`nonce` — confirm a platform-provided CSPRNG is actually used, with no timestamp/counter/username as the sole or partial entropy source, and no reuse of a value where the design specifically requires freshness or uniqueness.

**Encryption key management**, distinct from Guide 04's signing-key focus: generation and entropy, storage and access permissions, separation between environments (dev/staging/prod should never share production encryption keys), rotation and revocation, backup/recovery handling, the actual encryption mode and algorithm, whether integrity/authentication is applied alongside confidentiality where the design requires it (unauthenticated encryption lets an attacker manipulate ciphertext undetected), and how decryption failures are handled — do they ever produce an unsafe fallback rather than a clean rejection.

**TLS/certificate configuration review**, gray-box angle: hostname validation, certificate-chain validation, trust-store configuration, mutual-TLS requirements where applicable, and specifically whether any client library in the stack disables certificate validation — a pattern that's invisible from outside but catastrophic if present on a service-to-service connection carrying authentication material.

**Cryptographic integration in OAuth/OIDC and SAML**, cross-referencing Guides 07 and 08 but from the crypto-correctness angle specifically: is ID-token signature verification actually using trusted key material from the provider's published key set (not an attacker-reachable source); for SAML, are signatures validated using trusted keys with issuer/destination/audience checked alongside the signature, not instead of it. The critical reminder that applies to both: **a valid signature is necessary in federated flows but it is never the entire trust decision** — a correctly-signed assertion from an untrusted issuer, or a signed token for the wrong audience, must still be rejected regardless of signature validity.

## F. White-box testing

**Trace TLS/certificate configuration in code** for connections handling authentication material specifically — search for configuration or code that disables certificate validation, accepts any certificate, or trusts an arbitrary certificate authority. A finding here must identify the specific affected connection, what data or identity is actually exposed, and confirm that connection genuinely carries authentication traffic — a local test certificate in an isolated dev environment is not a production finding.

**Trace encryption of authentication secrets.** Where the design requires recoverable secrets (distinct from passwords, which never should be), confirm the implementation actually encrypts rather than silently falling back to plaintext on some code path, and that decryption keys are sourced separately from the data they protect rather than bundled together in a way that defeats the separation.

**Trace secure-randomness usage directly in code**, across the full catalog from section E: session identifiers, reset/verification tokens, MFA recovery codes, API credentials, nonces, OAuth `state`/`nonce`. Common concrete mistakes to search for: a general-purpose non-cryptographic PRNG used for any of these; reset tokens or recovery codes derived from a timestamp; a static token reused across multiple accounts; a nonce reused where the design specifically requires freshness.

**Trace cryptographic comparisons.** Confirm password-hash verification, HMAC verification, token verification, and signature verification all use the appropriate library's actual verification function — never an ad hoc string-equality comparison or a hand-rolled check. Constant-time comparison specifically matters where an attacker can make *repeated* comparisons against a secret and observe timing — it's not a universal requirement for every equality check in the codebase, so don't flag every `==` reflexively.

**Trace cryptographic failure paths** across every mechanism: do invalid signatures actually cause rejection; does failed decryption ever produce unsafe plaintext or a default-authenticated state instead of an error; do missing keys ever let authentication proceed *without* verification rather than failing closed; are unsupported algorithms rejected outright. The specific defect pattern worth hunting deliberately: normal validation correctly rejects an invalid signature, but a fallback exception handler then decodes the artifact *without* verifying it so processing can continue anyway — this defeats the entire cryptographic boundary the primary path establishes, and it's exactly the kind of thing that looks fine in a code review that only reads the happy path.

**Trace secret and key storage** across the codebase and configuration for hardcoded signing/encryption/client secrets, private keys, API credentials, development keys reused in production, insecure default secrets, and secrets inadvertently printed to logs. For each candidate: is it actually sensitive, is it live or obsolete, is it used in a real cryptographic operation, who can reach it, and can it actually be rotated or revoked if compromised — a string merely *resembling* a key is not sufficient evidence on its own.

## G. Differential testing

Nine-step workflow, same as Guide 04: define the required property, establish a baseline with valid controlled material, change one cryptographic property at a time, observe external behavior, inspect internal evidence, trace the code, establish root cause, assess what identity/boundary can actually be bypassed, verify the fix.

| Test | Black-box evidence | Gray-box evidence | White-box target |
|---|---|---|---|
| TLS validation failure | Certificate/connection behavior | Trust-store configuration | Client TLS settings |
| Weak encryption-at-rest | Observable artifacts and behavior | Key separation and algorithm config | Encryption implementation |
| Predictable recovery code/nonce | Token behavior and lifecycle | Token records and metadata | Generator implementation |
| Hardcoded encryption key | Often inconclusive externally | Secret-management evidence | Source and deployment config |
| SAML/OIDC signature-only trust | Assertion accepted or rejected | Trusted metadata and validation logs | Issuer/audience validation alongside signature |
| Cryptographic exception fallback | Error behavior | Error logs and feature config | Exception and fallback branches |

## H. Exploitation and validation

Minimal proof for a TLS-validation finding: demonstrate the specific connection accepting a certificate that shouldn't be trusted, with confirmation that connection genuinely carries authentication material. Minimal proof for a weak-randomness finding: implementation evidence (not a black-box sample) showing the generation mechanism is predictable, combined with a controlled demonstration of the consequence. Minimal proof for an at-rest encryption finding: a synthetic test secret recovered from its stored representation, or code that demonstrably round-trips it without a legitimate recovery need.

## I. False-positive analysis

A self-signed certificate is not automatically a vulnerability — it can be used safely in an appropriately configured, controlled trust environment; the question is whether the intended peer identity is actually authenticated and the trust configuration is appropriate for that context. "The application uses AES" proves nothing about key storage, mode, nonce handling, or integrity protection — review the complete design. A key identifier (`kid`) appearing in a token or config is usually meant to be public — confirm whether the actual secret or private key material is exposed before treating it as a leak. An old key still being present doesn't mean rotation failed — establish whether it's still *trusted*, which may be a legitimate compatibility overlap. Not every cryptographic weakness needs an actual successful crack to be reportable — implementation evidence plus a controlled proof of the violated property is sufficient; don't expose real credentials or keys chasing unnecessary "proof."

## J. Root cause

The recurring patterns across this guide: transport security assumed rather than verified on service-to-service connections specifically (client-facing HTTPS gets scrutiny; backend connections often don't); encryption keys stored alongside the data they protect, eliminating the access separation that gives encryption its value; random values for *secondary* security-sensitive purposes (recovery codes, nonces) generated with less rigor than the primary session/token generator, because they weren't treated as equally sensitive; and cryptographic fallback-on-failure paths that exist for operational resilience but silently degrade the security guarantee instead of failing closed.

## K. Remediation

- Enforce TLS with certificate and hostname validation on every connection carrying authentication material, including service-to-service — audit for any disabled-validation configuration specifically, since it's invisible from outside.
- Use authenticated encryption (not confidentiality alone) for any recoverable authentication secret, with keys stored and access-controlled separately from the ciphertext.
- Generate every security-sensitive value — not just sessions and tokens — with a platform CSPRNG; treat recovery codes, nonces, and challenge values with the same rigor as primary credentials.
- Never let a cryptographic validation failure fall through to a weaker fallback path that proceeds anyway; fail closed.
- Keep encryption and signing keys out of source code and shared configuration; separate by environment; define rotation and emergency-compromise procedures and verify retired keys actually stop being trusted.

## L. Regression testing

Re-verify TLS configuration after any infrastructure or reverse-proxy change. Re-test randomness-dependent values after any change to the generation library or its configuration. Re-run the fallback-path tests (malformed/invalid artifacts of every type) after any error-handling refactor, since that's exactly where a "helpful" fallback tends to get reintroduced.

## M. Finding documentation

Title, affected component (TLS config, encryption-at-rest, key management, specific generator), expected cryptographic property, observed failure, preconditions, minimal reproduction with controlled test data, evidence (baseline vs. modified artifact, resulting decision, relevant config or source path), security impact, root cause, remediation, retest confirmation, false-positive assessment (what alternative explanations were ruled out), severity grounded in the actual trust boundary crossed and attacker prerequisites — not the cryptographic term's name alone.

## Completion criteria

- [ ] TLS/certificate validation tested on authentication-relevant connections, including service-to-service where visibility allows
- [ ] Encryption-at-rest reviewed for any recoverable authentication secret, confirming key/ciphertext separation
- [ ] Randomness quality checked across the full catalog of sensitive values, not just sessions and tokens
- [ ] Cryptographic failure paths tested for unsafe fallback-on-error behavior across every mechanism (JWT, SAML, TLS, encryption)
- [ ] Federated signature validation confirmed to be paired with issuer/audience checks, not trusted on signature validity alone
- [ ] Hardcoded or environment-shared keys searched for in source and configuration
- [ ] Each finding traced to the specific violated cryptographic property, not reported as a generic "weak crypto" label
