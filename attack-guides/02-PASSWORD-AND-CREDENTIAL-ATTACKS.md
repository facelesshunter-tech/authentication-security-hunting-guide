# 02 — Password and Credential Attacks

**Source:** `Phases checklist.txt` Phase 5 "Password Storage and Credential Defenses" (read in full, lines 4963–6242), `Testing Approach .txt` Phase 5, and the course's Phase 1 (Password Storage and Credential Attacks) for conceptual grounding. Password reset and change specifics are summarized here only where they intersect password storage directly (Tests 3.6 and 5.7 of the source); the full reset/recovery flow is covered in Guide 06.

Password security has two connected sides, and they fail independently: how the password is stored and verified, and how well the system resists an attacker simply trying lots of them. Test and report these separately unless you have demonstrated one real attack chain connecting them — a weak password policy, an insecure hash, and insufficient rate limiting are three different findings, not one.

## A. Security objective

The application should store passwords so that even a full database compromise doesn't hand over usable credentials, verify them correctly on every authentication path, and resist systematic guessing — online or offline — without that resistance itself becoming a denial-of-service lever against legitimate users.

## B. Attack surface

Web login, API authentication, mobile authentication, registration and password creation, password change, forgot-password and reset flows, legacy endpoints and older API versions, and — where authorized — the database layer and authentication logs.

## C. Threat model

Two distinct attackers matter here. The first has no special access and is testing the system from outside: credential stuffing with previously breached pairs, password spraying across many accounts, systematic guessing against one account. The second has, or gains, read access to stored credentials — through a breach, a backup, an insider, or a chained vulnerability — and the question becomes whether that access converts into usable passwords at all. Required attacker capability ranges from none (online guessing) to database or backup access (offline cracking). Victim interaction: none.

## D. Black-box testing

Everything here uses only observable application behavior — no source, no database.

**Password creation and policy enforcement.** Observe the stated password rules at registration. Submit a synthetic account's password that clearly violates policy, and one that satisfies it; repeat through password change and reset where authorized; compare across web, API, and mobile. The finding you're looking for is the frontend rejecting something the backend accepts — not the mere existence of a permissive policy, which on its own isn't a vulnerability.

**Length and input handling.** Compare a normal password, one at the documented maximum, one beyond it, and passwords with permitted Unicode or leading/trailing whitespace, across registration, login, change, and reset. Ask specifically: does registration accept what login later rejects? Does one endpoint truncate while another processes the full value? Inconsistent truncation or normalization can make genuinely different passwords behave as equivalent — that's the real finding, not merely "a long password was accepted."

**Login rate limiting.** Use one designated test account and a small, pre-approved series of failed attempts. Record attempt count, timing, status, retry indicators, and whether the account becomes restricted. Compare the same account across web vs. API login, and the current endpoint vs. any authorized legacy endpoint. A documented rate limit on the primary login with none on an equivalent endpoint is a real finding; a single `200` after one failed attempt proves nothing about whether rate limiting exists at all.

**Credential stuffing defenses.** Distinguish this from ordinary guessing explicitly: stuffing tests previously-exposed username/password pairs, not a search for one account's password. Using only synthetic credentials, check whether attempts are rate-limited, whether repeated failures are detected, whether breached-password screening exists at creation/change. Never obtain or test real leaked credentials. The absence of a CAPTCHA is not itself a finding — CAPTCHA is one possible control among several, not a requirement.

**Account lockout and recovery.** Review documented lockout/delay/challenge behavior. If authorized to test it, use one designated account and stop at the agreed threshold. Compare restriction behavior across endpoints, and check whether the recovery mechanism bypasses a restriction that should remain in force. The critical judgment call here cuts both ways: no protection against guessing is a finding, but a lockout so aggressive that one attacker can trivially lock out another user is *also* a finding — a lockout system is itself an attack surface.

**Password reset and change (storage-relevant slice).** Using controlled accounts: does change require the current password or an approved reauthentication step? Do reset tokens expire and enforce single use? Does changing the password actually invalidate the old one and the token used to set it? Compare an existing session's behavior before and after a password change against the application's stated session policy — not every change must revoke every session, some systems deliberately keep the current one active, so verify the actual policy before calling a retained session a finding.

**What black-box testing cannot prove.** It cannot reliably establish the hashing algorithm, salt generation, work factor, or whether storage is reversible. A password not appearing in any HTTP response doesn't prove secure storage; an odd-looking stored value doesn't prove insecure storage either. Both require gray-box or white-box evidence — don't guess past this boundary.

## E. Gray-box testing

This is where storage questions actually get answered, using API docs, architecture notes, configuration, sanitized logs, or approved synthetic-account database records.

**Hashing configuration.** Identify the configured algorithm — Argon2id, scrypt, bcrypt, and properly-configured PBKDF2 are appropriate; SHA-256 or MD5 used directly on a password is not, no matter how it's dressed up. Record the actual parameters (work factor, memory, parallelism), and confirm the setting applies to the live verification path, not a stale config file a legacy service ignores.

**Salt handling.** Where permitted, create two synthetic accounts with the same test password and compare their stored hash records. Different encoded hashes are consistent with independent salting but don't alone prove the whole design is sound; identical hashes warrant investigation but could have an innocent explanation (test fixtures, a known deterministic migration) — confirm against the actual production path before concluding salts are absent.

**Hashing cost.** A slow hash isn't automatically a secure one — it can still be misconfigured. Record algorithm, parameters, library and version, and compare against the organization's current baseline; check whether legacy account cohorts run a different, weaker path.

**Plaintext or reversible storage.** Inspect the schema and data flow for password-related fields, not just their names — a field called `password_hash` proves nothing about what it actually contains. The strong version of this finding is a synthetic test password recovered directly from its stored representation, or code that demonstrably stores and retrieves the original value.

**Credential-defense configuration vs. observed behavior.** Compare the gateway's documented rate limit against black-box observation — and specifically check for an internal route that reaches the login service while bypassing the gateway entirely. A control configured at the edge is not a control if a path exists around the edge.

**Logs and monitoring.** Confirm successful/failed auth, rate-limit activation, lockouts, password changes and resets are actually recorded with useful timestamps and identifiers — and that credentials, reset tokens, and session secrets are excluded. A log entry proves an event was recorded, not that an attack was prevented; an absent entry doesn't prove a control failed, since logging can be sampled or routed elsewhere. Confirm the expected telemetry path before reporting a gap.

## F. White-box testing

Trace the actual code.

**Creation path:** `HTTP request → validation → password policy → account service → hashing library → database write → response serialization`. At each step: is the password hashed before storage using a dedicated library, with no fallback path that could persist it unsafely on an exception? Is there a second account-creation path — an admin-creation function, an import, a migration script — that writes the password directly without going through the same hashing service? This is a realistic, specific pattern worth hunting for deliberately: one correct path and one forgotten one.

**Verification path:** `Login request → validation → account lookup → account-state checks → password verification → authentication decision → session/token issuance`. Confirm a proper verification function is used (not a plaintext comparison), account-state checks happen before a session is issued, and legacy accounts don't silently use a weaker verification path than current accounts.

**Algorithm selection in code:** confirm no fallback to a fast general-purpose hash exists anywhere reachable, and that nobody invented a custom password-hashing construction instead of using a supported library. Finding an unsafe helper function in an unused file is not the same as proving production passwords are stored insecurely — confirm reachability before reporting.

**Policy enforcement across entry points:** trace registration, change, reset, and admin-reset side by side for server-side validation, the approved policy, the hashing service, and audit events. Not every entry point needs identical user-facing rules, but every entry point needs to satisfy the same underlying security policy.

**Rate-limit implementation:** trace `request → gateway/middleware → rate-limit decision → credential verification → failure accounting → response`. The realistic concern to check for specifically: a counter held only in local per-instance memory, which silently becomes a weaker system-wide limit than intended the moment the application runs across more than one instance.

**Reset/change code (storage-relevant slice):** confirm reset tokens use a secure random generator, are validated and invalidated server-side, and that an expired or already-used token cannot still authorize a change. Compare reset (proof = possession of a valid token) against change (proof = an authenticated session) — the different proof requirements are expected; the test is whether each flow actually enforces its own.

**Credential leakage in code:** search logs, debug output, exception handlers, analytics events, message queues, email generation, API response models, and support tooling for raw passwords, hashes, or reset tokens. Identify the specific source, the value, the destination, and who can reach it — a logging function existing somewhere is not evidence until you know what it actually records and where that goes.

## G. Differential testing

Five-step method: define the expected behavior ("every supported login endpoint applies the same credential-abuse controls"), pick a controlled comparison using the same synthetic account, change one variable (web vs. API login, modern vs. legacy endpoint, valid vs. expired reset token, password before vs. after change, configured value vs. runtime behavior), capture all relevant evidence on both sides, then establish whether the difference actually lets an attacker authenticate without satisfying requirements, guess more efficiently than policy allows, obtain password material, or retain access against policy. If you can't establish a real policy violation and consequence, keep it as an investigation item, not a finding.

| Test area | Black-box | Gray-box | White-box |
|---|---|---|---|
| Password policy | Compare accepted/rejected inputs | Inspect policy and endpoint config | Trace validation functions |
| Password storage | Check for externally exposed credentials | Inspect synthetic-account records and config | Trace hashing and database writes |
| Salt handling | Usually limited evidence | Examine encoded records and library settings | Review implementation and library usage |
| Hashing parameters | Not reliably observable | Verify configured vs. runtime parameters | Trace the actual hashing/verification path |
| Rate limiting | Small authorized tests | Inspect config, counters, logs | Review middleware and enforcement logic |
| Account lockout | Observe restriction/recovery behavior | Inspect policy settings and events | Review counters and state transitions |
| Password reset/change | Test expiration, reuse, lifecycle | Inspect reset settings and records | Trace token generation, validation, invalidation |
| Credential leakage | Inspect responses and interfaces | Review authorized logs/telemetry | Trace secrets into logs, serializers, integrations |

## H. Exploitation and validation

For a storage finding, the minimal proof is a synthetic test password recovered from its own stored representation, or code that demonstrably round-trips it. For a credential-attack-defense finding, the minimal proof is a controlled, rate-respecting demonstration that the documented threshold doesn't hold — not a full-scale guessing run. For a reset/lifecycle finding, demonstrate the specific violated property (reuse of a single-use token, an expired token still accepted, old password still valid after a "successful" change) against your own controlled account only.

## I. False-positive analysis

Challenge every one of these before reporting: an unfamiliar-looking hash encoding is not proof of weak storage — identify the actual algorithm first. Two accounts with different hashes is consistent with salting but doesn't prove the whole design is sound. No CAPTCHA is not automatically a defense gap. A `200` after a failed login is not evidence either way about rate limiting — check the actual resulting authentication state. No lockout is not automatically a vulnerability, and an aggressive lockout can itself be the vulnerability. A secure algorithm named in a config file doesn't prove the production path uses it. An unusual function existing in source doesn't prove it's reachable or used.

## J. Root cause

Weak storage findings usually trace to one of: a fast general-purpose hash used directly, a forgotten second account-creation path that bypasses the hashing service, missing or stale migration off a legacy hash format, or reversible encryption used where one-way hashing was appropriate. Weak credential-defense findings usually trace to: rate limiting enforced only at an edge gateway with an internal bypass route, per-instance-local counters in a multi-instance deployment, or protections applied to the primary login but never ported to an API, mobile, or legacy equivalent.

## K. Remediation

- Use a supported password-hashing algorithm built for password storage (Argon2id, scrypt, bcrypt, or correctly-parameterized PBKDF2), with parameters matched to current guidance, through the library's own salt handling — never a custom construction.
- Never store conventional passwords in plaintext or recoverable encryption; migrate legacy hashes deliberately rather than leaving a permanently weaker cohort.
- Enforce policy and use the same hashing/verification service across every entry point — registration, change, reset, and any administrative or migration path.
- Apply credential-abuse controls server-side, scoped correctly for multi-instance deployments, consistently across every authentication route including legacy and mobile.
- Enforce reset-token expiration and single use, invalidate on password change, and keep credentials and tokens out of logs, analytics, and exception output.

## L. Regression testing

Re-verify the hashing configuration and parameter baseline after any dependency upgrade (library defaults change), re-test rate limiting after any infrastructure change (new instance, new gateway rule), and re-check for the "forgotten second path" pattern after any new account-creation or admin tooling ships.

## M. Finding documentation

Use the standard structure: title, summary, affected component, preconditions, evidence (sanitized request/response, config or DB evidence, source path, observed vs. expected behavior), security impact, root cause, remediation, retest criteria. Never include real passwords, live reset tokens, reusable session cookies, or unrelated users' secrets in a report. Assign severity from reachability, prerequisites, actual impact, and scope — not from the vulnerability's name; direct exposure of recoverable production passwords and a minor rate-limit inconsistency are not the same severity even though both fall under "password security."

## Completion criteria

- [ ] Password policy tested across every entry point (registration, change, reset, admin)
- [ ] Rate limiting and lockout behavior tested with controlled, rate-respecting attempts
- [ ] Hashing algorithm, parameters, and salt handling verified via gray-box or white-box evidence, not assumed from black-box behavior
- [ ] Reset/change token lifecycle (expiration, single-use, invalidation) tested
- [ ] Credential leakage checked across logs, responses, analytics, and exception paths
- [ ] Every entry point compared for consistency, with legacy/API/mobile routes included, not just the primary web flow
- [ ] Each finding traced to a specific root cause and a defensible, demonstrated consequence — not reported on the vulnerability's name alone
