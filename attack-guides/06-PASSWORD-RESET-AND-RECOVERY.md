# 06 — Password Reset and Account Recovery

**Source:** `Phases checklist.txt` TEST 3.6 and TEST 5.7 from Phase 5 (read in full as part of Guide 02's source), plus the recovery-specific sections of Phase 3 (section 6 "Account enumeration through password recovery," section 9 "Password-reset and verification-token leakage," and item 22 "White-box checklist: password-reset and verification flows," read in full), `Testing Approach .txt` Phase 6, and the course's Phase 2, lessons 2.2–2.7 read in full.

Recovery is a high-value target precisely because it's designed to let someone back into an account without their normal credentials — which means a weakness here can bypass protections that are entirely correct everywhere else. A perfectly hashed password and solid MFA don't matter if the reset flow hands out account control through a side door.

## A. Security objective

A recovery token should prove possession of exactly one thing: the ability to receive a message at the account's verified recovery channel, for exactly one account, for exactly one use, within a short window — and the link that delivers it should point to a destination the application controls, not one an attacker can redirect.

## B. Attack surface

Forgot-password request, reset-token validation, password-update submission, email/SMS verification reused for recovery, and the email/link-generation step itself (host selection, template construction, recipient selection).

## C. Threat model

Three distinct attackers: one testing whether the recovery flow discloses account existence or state (enumeration); one who has obtained or can influence a token's destination and is trying to convert that into account control (token theft, reuse, poisoning); and one probing whether the token's binding to a specific account and purpose is actually enforced server-side (account-binding bypass). None of these require the attacker to know the account's actual password.

## D. Black-box testing

**Map the complete flow** end to end: request → verification issued → token delivered → token submitted → token validated → password changed → sessions handled → user notified. Document every stage before testing any single one.

**Account enumeration through recovery.** Submit the same recovery request for an existing verified account, a nonexistent identity, an unverified account, and a restricted account if supported. A generic response like `"If an account exists, recovery instructions will be sent"` reduces direct enumeration through the message body — but check every other observable channel independently: status code, redirect, response timing across repeated controlled requests, and response fields. A recovery email landing in your own controlled mailbox is expected and not itself a finding; the question is whether an *unauthorized* requester can infer account existence from something observable.

**Token properties.** For a controlled recovery token, test: does it expire, is it single-use, is it bound to the account that requested it, is it invalidated after successful use, and — the case most worth deliberately testing — submit the identical token a second time after a successful reset and confirm whether the second submission is rejected or silently succeeds. A generic "success" response proves nothing; confirm the password actually changed (or didn't) on the second attempt.

**Token leakage.** Check response bodies, redirect URLs, query parameters, error pages, and any application-generated link for the raw token value, and — where authorized — logs, analytics, and third-party requests. A token appearing in a URL is not automatically a finding; the real questions are expiration, account binding, validity, and whether it leaked somewhere an unauthorized party could actually observe it.

**Password-reset poisoning via Host-header trust.** This is a distinct, high-value technique the checklist doesn't cover but the course does in depth, and it's worth testing deliberately: send a forgot-password request with a manipulated `Host` (or `X-Forwarded-Host`) header and inspect whether the resulting reset email's link is built from that untrusted header instead of a fixed canonical domain.

```http
POST /forgot-password HTTP/1.1
Host: attacker-controlled.example
Content-Type: application/x-www-form-urlencoded

email=alice%40example.test
```

If the generated email link points at `attacker-controlled.example` rather than the real application, and the victim clicks it, the reset token — the actual security boundary — gets delivered to infrastructure the attacker controls. This is categorically more serious than an ordinary open redirect, because the thing crossing the boundary is a credential capable of taking over the account, not just a navigation destination. Confirm the full chain before claiming account takeover: the token must still be valid, bound to the intended account, unexpired, unused, and actually exchangeable for a password change — a manipulated Host header producing *some* unexpected redirect is not sufficient evidence on its own.

**Reset-message abuse.** Trigger multiple recovery requests for the same (your own controlled) address and check for rate limiting — this matters even without account-takeover potential, since unlimited triggering is an abuse-resource and user-harassment problem in its own right.

**Password-change security (the storage-adjacent slice).** Does change require the current password or an approved reauthentication step; do reset tokens get invalidated the moment the password actually changes; are other active sessions handled per the documented policy. Some systems deliberately retain the session performing the change while revoking others — verify the actual stated policy before calling a retained session a finding.

## E. Gray-box testing

Compare the documented recovery policy (token lifetime, single-use requirement, canonical host configuration) against observed behavior. Where available, inspect delivery logs or email-sending configuration to confirm the recipient is derived from the verified account record, not from any client-supplied value in the request.

## F. White-box testing

**Trace the complete lifecycle**: `recovery request → account lookup → token generation → token storage → email/SMS delivery → token validation → password or account-state update → token invalidation → session handling → audit event`. At each step: is the token generated with a CSPRNG, scoped to the correct account and purpose, server-side-expired, single-use, invalidated on success, excluded from logs and unnecessary responses, and incapable of being substituted into a different workflow than the one it was issued for.

**Token reuse — the exact vulnerable pattern to hunt for:**

```python
async def reset_password(token, new_password):
    reset = await find_reset_token(token)
    if not reset:
        raise Unauthorized()
    await update_password(reset.user_id, new_password)
```

This checks only that the token *exists* — no expiry check, no used-check, no revocation check, and critically, no visible consumption of the token after success. The secure version checks `used_at`, `expires_at`, and `revoked_at` independently, then marks the token used and invalidates any other outstanding tokens for the same account, with the password update and token consumption designed to stay consistent if either operation fails partway.

**Account binding — the exact vulnerable pattern:**

```python
reset = await validate_reset_token(token)
user = await find_user(request.email)
await update_password(user.id, new_password)
```

The token is validated, but the account whose password actually changes comes from a separately client-supplied email, not from the validated token record itself. If the server doesn't independently enforce `user.id == reset.user_id`, an attacker holding a valid token for *their own* account could potentially redirect the password change to a different account entirely by supplying a different email alongside it. The secure version reads the target `user_id` only from the validated token record — the client-supplied identifier should never get a vote.

**Expiration enforcement.** Confirm expiry is checked server-side, not only disabled in the frontend UI after a timer; confirm the timestamp comparison itself is correct (timezone consistency, no seconds/milliseconds confusion); confirm the application checks the *correct* token record when multiple reset requests exist for the same account rather than mixing up expiration status between them; and confirm expiration and revocation are treated as genuinely separate controls — a token within its time window that should have been revoked by an intervening security event (another successful reset, a password change through a different path) must still be rejected.

**Email/link generation.** Trace `user input → account lookup → recovery record creation → recipient selection → email template construction → email provider → delivery`. The recipient must be derived from the verified account record, never overridden by a client-supplied value in the same request. Check whether the canonical host used to build the reset link comes from fixed server-side configuration (`CANONICAL_ORIGIN = "https://app.example.com"`) or from request-derived data (`request.headers.get("Host")`) — the latter is the exact pattern enabling the poisoning attack from section D.

## G. Differential testing

Compare: correct vs. incorrect current password on change (is reauthentication enforced); valid vs. expired token; first vs. second use of the identical token; old vs. new password after a claimed-successful change; existing session before vs. after the change; web reset vs. API reset for consistency; normal `Host` header vs. manipulated `Host` header on the resulting email link.

## H. Exploitation and validation

Minimal proof for reuse: demonstrate the password actually changes (not just a 200 response) on the second submission of an already-used token. Minimal proof for account binding: demonstrate a token validated for Account A results in Account B's password changing, using two accounts you control. Minimal proof for Host-header poisoning: demonstrate the actual generated email content points to an attacker-controlled host, not just that an HTTP redirect responds to a manipulated header — the email content is the real security boundary, a redirect alone is a weaker signal.

## I. False-positive analysis

A recovery email arriving in your own mailbox is expected, not a finding. A generic success-shaped response doesn't prove the server-side operation actually succeeded or failed as claimed — verify the real resulting state. A token appearing in a URL isn't automatically exploitable — establish its actual purpose and capability first. An unrecognized-hostname redirect landing somewhere harmless (a public page) is low or no impact; the same mechanism feeding a password-reset email link is categorically more serious — don't rate every Host-header quirk the same. Different responses between existing and nonexistent accounts aren't automatically a meaningful vulnerability; some applications intentionally treat usernames as public, and severity depends on context.

## J. Root cause

The two dominant patterns, both demonstrated above in code: token validation that checks existence but not lifecycle state (expiry, single-use, revocation), and account selection that trusts a client-supplied identifier instead of deriving the target account solely from the validated token record. A third, less common but higher-severity pattern: link generation trusting request-derived host information instead of fixed server configuration.

## K. Remediation

- Generate tokens with a CSPRNG, bind them to one account and one purpose, enforce server-side expiration and single-use, and consume them atomically with the password update.
- Derive the target account exclusively from the validated token record — never from any client-supplied identifier submitted alongside it.
- Build recovery links from a fixed, server-configured canonical origin — never from `Host`, `X-Forwarded-Host`, or any other request-derived value.
- Rate-limit recovery requests per account and per source to prevent message-flooding abuse independent of any takeover risk.
- Keep raw tokens out of logs, analytics, and unnecessary API responses; invalidate all outstanding tokens for an account whenever one is successfully used.

## L. Regression testing

Re-test token reuse and account binding after any change to the reset handler or its underlying ORM/ accessor layer. Re-test Host-header handling after any change to reverse-proxy or load-balancer configuration, since that's where trusted-host assumptions tend to silently drift.

## M. Finding documentation

Title, affected component, expected vs. observed behavior, preconditions, minimal reproduction with controlled accounts and tokens, evidence (the actual generated email content for poisoning findings, not just an HTTP redirect), security impact, root cause, remediation, retest confirmation. For poisoning findings specifically, demonstrate the full chain through to token delivery at an attacker-controlled destination — a redirect alone is a weaker, different finding (open redirect) worth reporting separately if that's all you can demonstrate.

## Completion criteria

- [ ] Complete recovery flow mapped end to end
- [ ] Enumeration tested across every observable channel (message, status, timing, redirect), not just the response body text
- [ ] Token reuse tested by submitting an already-used token a second time and confirming the actual resulting state
- [ ] Account binding tested by attempting to redirect a validated token's effect to a different account
- [ ] Expiration tested server-side with valid, expired, used, and revoked tokens
- [ ] Host-header-based link generation tested, with the actual generated email content inspected, not just the HTTP response
- [ ] Token leakage checked across logs, URLs, analytics, and third-party requests
- [ ] Password-change session/token invalidation tested against the documented policy, not assumed
