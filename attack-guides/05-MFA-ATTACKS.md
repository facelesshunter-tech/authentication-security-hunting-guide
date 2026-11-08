# 05 — MFA Attacks (TOTP, Push, SMS, Backup Codes, Passkeys/WebAuthn)

**Source:** `Testing Approach .txt` Phase 7 (read in full), and the course's Phase 3 (MFA Attacks, lessons 3.2–3.7 read in full), Phase 4 (MFA Enrollment and Recovery, lessons 4.2–4.6 read in full), and Phase 5 (Passkeys/WebAuthn/FIDO2, lessons 5.8–5.15 read in full).

**Honest source gap:** unlike guides 02–04, `Phases checklist.txt` has no dedicated MFA deep-dive phase — only a brief discovery step inside Phase 1 ("Step 1.8 — Discover passkey and WebAuthn functionality"). This guide is therefore built primarily from the course's conceptual depth and worked examples plus `Testing Approach .txt`'s workflow framing, not from a checklist-style granular test bank like the other guides. The depth is real, but it comes from a different mix of sources — flagged here rather than silently presented as equivalent.

## A. Security objective

MFA should mean what it claims: an account requiring a second factor should never reach a fully authenticated state by any route — primary, fallback, legacy, or recovery — without that factor actually being verified. The approval decision itself (push, OTP entry) should be bound to the specific login attempt it's meant to authorize, not reusable or guessable or subject to habituation.

## B. Attack surface

MFA enrollment, verification, removal, replacement, backup-code generation and use, trusted-device/remember-me functionality, every authentication route that could theoretically skip or weaken the MFA requirement (legacy endpoints, API, mobile, password-only fallback), and — where passkeys are supported — registration and authentication ceremonies specifically.

## C. Threat model

Several distinct attackers matter here, each requiring different capability: one who already has the password and is trying to get past the second factor (push bombing, SMS interception, TOTP replay, downgrade to a weaker route); one who is relaying a live authentication session in real time (adversary-in-the-middle phishing); and one who exploits the lifecycle around the factor itself rather than the factor's cryptography (enrollment takeover, weak removal/replacement authorization, backup-code brute force). Required victim interaction ranges from none (downgrade, replay) to active (approving a push prompt, completing a phished login).

## D. Black-box testing

**MFA state mapping (foundation for everything else).** Submit valid primary credentials and stop at the MFA challenge — record exactly what the response contains and which protected resources are reachable before the challenge is completed. Then complete it and compare. The target finding: any resource reachable in the MFA-pending state that should require full authentication.

**Push bombing / MFA fatigue.** Trigger repeated authentication attempts against a controlled account and observe: is there any throttling on push notifications, can the user reject a suspicious prompt, does excessive prompt activity generate any alert, and — critically — is the approval actually bound to the specific login attempt that triggered it, or can a stale approval authorize a different request? Number matching raises the bar but isn't a complete fix — if a victim can be socially engineered into supplying the matching number to a live phishing flow, the attack can still succeed depending on the implementation. Do not report "the app uses push notifications" as a finding on its own; the finding is a demonstrated absence of abuse controls or a reusable approval.

**Real-time (AiTM) phishing susceptibility.** This is conceptually different from every other test here: a correctly-implemented OTP system can still be defeated because a manually entered code is a transferable value, not a cryptographic binding to the legitimate origin. HTTPS on the phishing page doesn't help — it only proves a secure connection to the wrong domain. When assessing this, keep the stages analytically separate: stealing a password, relaying an OTP, completing authentication through an intermediary, obtaining a resulting session credential, and successfully reusing that credential are five different things, and evidence for one does not prove the others. Passkeys (WebAuthn) are the actual mitigation here, since origin binding is enforced by the protocol itself — see the passkey section below for what "correctly implemented" means in practice.

**OTP interception and relay.** Distinguish SMS OTP (vulnerable to SIM swap, telecom interception, and relay), email OTP (vulnerable if the same mailbox also gates password recovery — a dependency-concentration problem, not an OTP problem per se), and TOTP (avoids SMS delivery risk but a manually-entered code is still a transferable value, still phishable in real time). None of these bind the response to a specific browser transaction the way WebAuthn does.

**TOTP replay.** Complete an MFA verification with a valid controlled code, then submit the exact same code again within its validity window. If the second submission is accepted, the server is checking only "does this match a valid TOTP value" rather than tracking whether that specific time-step has already been consumed for the account. Use a fresh server-side consequence as evidence (a new authenticated state, not just an irrelevant 200), and don't confuse a code submitted after its window expired (that tests expiration, not replay) with a genuine replay.

**SMS/SIM-swap-adjacent testing.** You can't test the telecom layer directly, but you can test the application's contribution to the attack chain: does changing the phone number require verification of the old number or recent strong authentication, can a newly-added number be used immediately for sensitive recovery, are old recovery factors invalidated when a new one is added, is the user notified of the change. A missing notification alone isn't an account-takeover finding — it's a detection gap, categorize it accordingly.

**MFA downgrade.** Build an authentication-path inventory across every route: entry point, primary factor, required second factor, fallback behavior, resulting session, protected actions available, and whether the factor can be removed/replaced from that path. The concrete downgrade patterns to test for, each independently: a weak fallback that grants a full session without sufficient verification; MFA enforced on `/login` but not on an older or parallel route reaching the same account; recovery used to strip or replace a strong factor with a weak one; a remembered-device marker that's forgeable or too easily set; a legacy protocol/endpoint that simply never got the MFA requirement. A weaker method existing isn't automatically a finding — an *unjustified* reduction in required assurance is the actual target.

**Passkey/WebAuthn-specific black-box checks.** Initiate a real authentication ceremony, record the expected challenge through your own test harness, complete it normally to confirm the baseline, then test whether a response tied to an incorrect or expired challenge is actually rejected — don't assume rejection just because the endpoint returns a generic response, confirm the real authentication outcome. For origin validation: determine whether the server validates against an exact expected origin or a defined allowlist, and if a wildcard-style trust boundary exists, establish specifically what it matches and whether an attacker can control anything inside it — a wildcard is a lead, not an automatic finding.

## E. Gray-box testing

Compare the documented MFA policy against observed behavior — if documentation says the billing dashboard requires MFA but you can reach it from an MFA-pending session, that's a strong discrepancy, but confirm the test account was genuinely in the pending state and the dashboard is genuinely supposed to require full auth before reporting it. Review available configuration for push rate limits, OTP validity windows, and replay-protection settings, and compare against what you actually observed. Check relying-party configuration for passkeys: is the expected RP ID derived from trusted server-side config, or could client input influence it? A legitimately configured parent-domain RP ID is not itself a finding — check whether the configured scope is intentional, not whether it differs from the exact hostname.

## F. White-box testing

**MFA enforcement in code**, the realistic defect pattern from course Phase 3: MFA is checked only *if* a code is supplied, rather than being required whenever the account's policy demands it —

```python
if mfa_code is not None:
    if not verify_mfa_code(user, mfa_code):
        raise Unauthorized()
return create_authenticated_session(user.id)
```

— which silently grants a full session when the attacker simply omits the MFA parameter. The fix checks the account's actual requirement, not merely whether a code happened to be present.

**Enrollment code.** The dangerous pattern: an enrollment endpoint that immediately activates a new authenticator on save, with no distinction between "configured," "verified," and "active and trusted." A secure design separates these into a pending-enrollment record, requires recent authentication and authorization to start a factor change, and only activates the authenticator after the pending enrollment is confirmed with a valid code *and* confirmed to belong to the same user who started it. Check specifically whether a pending enrollment can be confirmed by any account other than the one that created it.

**Secret handling.** Search for the shared TOTP secret appearing anywhere it shouldn't: application logs (`logger.info("...secret=%s", secret)` is the exact anti-pattern to grep for), error reports, analytics, API responses, backups. The secret is more sensitive than any single OTP — it doesn't expire, so if it leaks, rotating the displayed code solves nothing; the authenticator itself has to be replaced.

**Removal and replacement code.** The realistic vulnerable pattern is a disable-MFA or replace-authenticator function gated only on a valid session, with no check for recent authentication, no verification of the existing factor, and no authorization check distinguishing "this session is allowed to change security settings" from "this session can read a profile." Removal should require either the existing factor or a properly-designed recovery process — never just "a session exists." Replacement is the more dangerous of the two, since it can transfer account control directly: check whether the old factor is revoked only *after* the new one is successfully verified (avoiding both unauthorized takeover and accidental lockout), not before.

**Backup codes.** Check the actual generator — `random.randint()` or any non-cryptographic PRNG generating a small numeric space is a real, specific defect to hunt for (should be `secrets` or equivalent CSPRNG). Also check single-use enforcement (can the same backup code authenticate twice), rate limiting on submission (a six-digit code has a small enough space that brute force matters), and whether backup codes are invalidated when the primary authenticator is replaced.

**WebAuthn verification code — the critical pattern to hunt for:**

```python
async def authenticate_with_passkey(credential_id, signature):
    credential = await find_credential(credential_id)
    if verify_signature(credential.public_key, signature):
        return create_session(credential.user_id)
```

This is incomplete as a WebAuthn implementation regardless of whether the signature check is cryptographically correct — it's missing challenge validation, origin validation, RP ID hash validation, and authenticator flag checks (user presence/verification). A real implementation must validate the complete assertion, not an isolated signature; confirm whether the omitted checks live in a library layer before concluding the vulnerability is real, but also confirm the library is actually being invoked correctly — using a correct library doesn't help if the application ignores its validation errors or supplies the wrong expected origin or challenge.

**Credential-to-account binding.** Confirm a credential registered to one account cannot authenticate a different account — this matters especially for discoverable credentials, where the authenticator identifies itself without the user first supplying an account identifier; the server must still independently verify the account association, not trust the credential identifier as identity proof on its own.

## G. Differential testing

Compare: MFA-enabled account vs. MFA-disabled account on the same operation; primary route vs. every alternate/legacy route for the same login, specifically for MFA enforcement consistency; valid OTP vs. the same OTP replayed; fresh challenge vs. stale/expired challenge for WebAuthn; existing-factor-verified removal vs. session-only removal. For each, the question is the same: does the weaker path actually grant a resulting session with the same privileges as the properly-authenticated path.

## H. Exploitation and validation

Minimal proof for downgrade: demonstrate the alternate route actually issues a fully authenticated session for an account whose policy requires MFA, using your own controlled account. For enrollment/removal/replacement: demonstrate the specific missing check (no recent-auth requirement, no existing-factor verification, cross-account enrollment confirmation) against your own account, not a real user's. For replay: demonstrate the second submission of the identical code produces a genuine new authenticated state, not just a 200 with no consequence.

## I. False-positive analysis

An application using push notifications is not itself a finding — the finding is a demonstrated absence of throttling or a reusable approval. A missing SIM-swap notification is a detection gap, not automatically account takeover. A fallback method existing is not automatically a vulnerability — an unjustified assurance reduction is. A wildcard RP ID or origin allowlist is a lead, not a finding, until you establish what it actually matches and whether an attacker can control anything inside that scope. A code submitted after its time window expired tests expiration, not replay — don't conflate the two. Finding a challenge record still present in storage doesn't prove a vulnerability; the question is whether the server will still accept an assertion tied to it when it shouldn't.

## J. Root cause

The recurring pattern across this entire guide: an authentication-strength requirement that's checked conditionally ("if a code is present") instead of unconditionally ("if the account requires it"), and lifecycle operations (enrollment, removal, replacement) gated on session validity alone instead of on recent authentication plus explicit authorization for that specific change. For passkeys specifically: partial WebAuthn implementations that verify the signature but skip challenge, origin, or RP ID validation — treating "the cryptography is correct" as equivalent to "the complete protocol was followed."

## K. Remediation

- Make MFA enforcement unconditional on the account's actual policy, never conditional on whether the client happened to supply a code.
- Separate enrollment into pending/verified/active states; require recent authentication and explicit authorization before starting a factor change, and confirm enrollment only for the account that initiated it.
- Require either the existing factor or a properly-scoped recovery process before removal or replacement — never session validity alone — and sequence replacement to activate the new factor before revoking the old one.
- Generate backup codes and any other security-sensitive random value with a CSPRNG, enforce single-use and rate limiting.
- Track accepted TOTP time-steps per authenticator to reject replay within the validity window.
- For WebAuthn, use a maintained library and verify the complete assertion — challenge, origin, RP ID hash, signature, and required flags — never signature alone; derive expected origin and RP ID from trusted server-side configuration, never from client input.

## L. Regression testing

Re-test every alternate authentication route after any new route ships (mobile, partner integration, API version) to confirm MFA enforcement didn't get left behind. Re-test enrollment/removal/replacement after any session or authorization-middleware change. Re-test WebAuthn challenge/origin/RP-ID validation after any library upgrade.

## M. Finding documentation

Title, affected component (specific route or lifecycle operation), expected vs. observed behavior, preconditions, minimal reproduction with controlled accounts, evidence, security impact (what does the bypassed/downgraded/replayed factor actually grant), root cause, remediation, retest confirmation. For AiTM/phishing-related findings specifically, be precise about which of the five stages (credential theft, OTP relay, completed authentication, session credential obtained, session credential reused) you actually demonstrated — don't claim account takeover from evidence of relay alone.

## Completion criteria

- [ ] MFA state transitions mapped (pending vs. fully authenticated) with resource access tested in each
- [ ] Push/OTP abuse controls tested: throttling, rejection, binding to the specific login attempt
- [ ] TOTP replay tested with an identical code resubmitted within its validity window
- [ ] Downgrade tested across every discovered authentication route, not just the primary one
- [ ] Enrollment, removal, and replacement each tested for authorization independent of session validity alone
- [ ] Backup-code generation, single-use, and rate limiting tested or reviewed in source
- [ ] If passkeys/WebAuthn are supported: challenge, origin, RP ID, and signature validation each tested independently
- [ ] Every finding traced to the specific missing check, not reported as a generic "MFA bypass"
