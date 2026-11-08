# 10 — Reauthentication and Step-Up Authentication

**Source:** `Phases checklist.txt` Phase 9 "Reauthentication and Step-Up Authentication Security" (read in full, lines 9930–11002 — 8 BB tests, 6 GB areas, 7 WB tracing areas), `Testing Approach .txt` Phase 9, and the course's Phase 9.

The core principle this whole guide is built on: **being logged in does not automatically mean a user is sufficiently authenticated for every operation.** A session can be entirely valid while the user's last *authentication* is too old, too weak, or the wrong method for a specific sensitive action.

## A. Security objective

For every protected operation, the application must enforce the required authentication assurance, freshness, and authorization before the action takes effect — enforced server-side, across every route that can reach the operation (web, API, mobile, admin, alternate HTTP methods, background workflows), not merely displayed as a UI prompt.

## B. Attack surface

Password change, recovery-email/phone change, MFA removal, API-credential generation, payout/ownership changes, account deletion, access to particularly sensitive records, and any other operation the application's own policy marks as requiring fresh or stronger proof.

## C. Three concepts to keep separate

**Authentication** (establishing control of an authenticator), **reauthentication** (re-establishing confidence in identity for a specific operation — e.g. entering the password again before changing MFA), and **step-up authentication** (meeting a *stronger* assurance requirement for a higher-risk action — e.g. requiring a passkey before a high-value transaction). These overlap but aren't interchangeable: a fresh password check can satisfy a reauthentication policy without satisfying a phishing-resistant-MFA requirement, and completing MFA earlier in a session doesn't automatically prove the user meets a *fresh*-authentication requirement right now.

## D. Threat model

The realistic attacker already has a valid, currently-logged-in session (their own, if testing authorized) and is trying to perform an operation that policy says should require additional proof — either by simply not providing it, by providing stale proof, by hitting an alternate route that forgot to enforce the same policy as the main one, or by exploiting a challenge that got bound to the wrong context (wrong operation, wrong session). No special capability is required beyond an existing session — this guide is entirely about whether that session alone is being trusted for more than it should be.

## E. Black-box testing

**Baseline first.** Establish what a legitimate, fully-compliant operation looks like — authenticate, complete whatever reauthentication the UI asks for, perform the change, verify it actually took effect.

**Missing reauthentication.** Authenticate normally, deliberately skip the additional step the target operation is supposed to require, then attempt the operation through the ordinary documented interface. The absence of a password field in one captured request isn't sufficient evidence by itself — the server might have a separately-maintained recent-authentication state. The actual proof requirement: establish that the session genuinely has *not* satisfied the freshness/assurance policy, and the prohibited operation *still* takes effect.

**Authentication freshness.** Compare a fresh-authentication scenario (authenticate, immediately perform the operation) against a stale one (authenticate, let the policy window lapse via an approved test configuration, then attempt the same operation). Keep every other variable identical. Critically, don't conflate session expiration, access-token expiration, refresh-token expiration, authentication freshness, and MFA freshness — these are five different controls, and a session can be fully valid while authentication itself is stale.

**Step-up enforcement.** Pick an operation with an explicit stronger-assurance requirement (MFA removal, payout change in a test environment, privileged API-credential creation, ownership change) and compare a session that hasn't completed the required step-up against one that has. The realistic defect: the web UI correctly shows an MFA challenge, but the underlying server-side operation never actually checks whether it was completed.

**Direct access to the underlying operation.** Identify the sensitive operation's real request, establish a session that hasn't met the reauthentication/step-up condition, then hit that request directly (bypassing the UI prompt entirely) through an authorized testing tool. None of the following alone prove anything: the page being reachable without a prompt, a `200 OK` response, the endpoint simply existing, or client-side code containing a prompt. The proof is the direct request bypassing a required server-side check *and* producing the prohibited state change.

**Alternate routes and HTTP methods.** Compare the same sensitive action across every supported interface (browser, mobile, public API, account-management API, admin interface, legacy endpoint). A different status code alone doesn't establish a bypass — verify the resulting authentication state and whether the operation actually completed.

**Reauthentication failure handling.** Test incorrect password, incorrect MFA code, expired/cancelled/missing challenge response, invalid passkey ceremony. Confirm failure never authorizes the action — and specifically check whether a challenge that *begins but never completes* can leave behind a state the application later mistakes for success.

**Authentication-context confusion.** The scenario worth testing deliberately: begin reauthentication for operation A (e.g. password change), then attempt a *different* sensitive operation B (e.g. disabling MFA) and check whether the earlier authentication event gets accepted for B too. This isn't automatically a finding — some systems intentionally let one recent authentication authorize multiple operations within a window — but if the design is supposed to require purpose-specific authentication, verify the event is actually bound to the correct account, session, intended operation, required method, and freshness window, not just a generic "recently authenticated" flag.

## F. Gray-box testing

Compare documented policy (freshness windows, which methods satisfy which operations, step-up requirements) against deployed configuration and observed behavior — a setting literally named `reauthentication_required` existing doesn't prove it's enforced, and its absence doesn't prove a gap if the policy lives elsewhere. Inspect session/authentication metadata where authorized: does the application distinguish session *creation* from the *last successful authentication*, does it record which method was used, can it tell password-only from MFA-completed. Inspect the step-up challenge's own lifecycle — is it bound to a specific account, session, required method, and intended operation/scope, or could a challenge issued for one context authorize a different one. Compare environments for debug-only shortcuts, disabled MFA requirements, or administrative exceptions broader than intended in production.

## G. White-box testing

**Trace the sensitive-operation path**: `request → route/controller → session/token validation → account-state check → authorization check → reauthentication/step-up policy check → business operation → persistence → audit`. The critical question: is the authentication check performed *before* the sensitive side effect, using authoritative session state — and can a background job or alternate entry point perform the same operation without preserving that requirement? A check existing somewhere in the repo that isn't actually invoked on the relevant path protects nothing.

**Trace freshness checks** — the realistic vulnerable pattern:

```python
def recent_authentication(last_authenticated_at, now, max_age):
    if last_authenticated_at is None:
        return False
    age = now - last_authenticated_at
    return 0 <= age <= max_age
```

Review for: correct timestamp source (does this reflect the *authentication* event, not just session creation?), correct time-window comparison, handling of missing timestamps, clock/timezone assumptions, and whether a client-supplied timestamp or claim is ever trusted instead of a server-recorded one.

**Trace assurance-level enforcement.** A boolean `mfa_verified` flag isn't automatically unsafe, but check whether the policy actually needs to distinguish more states than that: password-only vs. password+MFA vs. passkey vs. phishing-resistant specifically, and a *stale* MFA event vs. a fresh one. The realistic defect: a stronger requirement collapsed into a generic authenticated flag that doesn't actually encode which method or how recently.

**Trace challenge generation and validation.** Unpredictable challenge/transaction ID, bound to the correct account and context, expiration and attempt limits enforced, the challenge marked successful only *after* real validation (not on initiation, not on a client claiming success), and the successful event can't be applied outside its permitted context. The specific mechanism matters — WebAuthn needs origin/challenge/RP-ID/signature validation (see Guide 05), TOTP needs shared-secret validation within the correct window, password reauthentication needs real verification plus throttling.

**Trace session mutation.** Confirm the sequence is genuinely `challenge initiated → proof submitted → proof validated → context updated → operation authorized`, and specifically hunt for implementations that mark a session as freshly authenticated the moment a challenge is merely *initiated*, or when a request simply reaches the challenge endpoint, rather than only after successful validation.

**Trace shared middleware.** Confirm the actual routes handling sensitive operations use the shared policy component (not just that the component exists), check whether any exclusions are documented and justified, and whether the protected business operation is reachable through a route the middleware doesn't cover.

**Trace business-operation boundaries.** Some applications validate reauthentication in a controller but perform the actual sensitive change in a separate service — confirm the security decision stays effective through to that service, not just at the controller's entry point, and check whether the operation is triggerable through another supported entry point that skips the controller entirely.

## H. Differential testing

| Dimension | Baseline | Comparison | Question |
|---|---|---|---|
| Authentication age | Fresh | Stale | Is freshness enforced? |
| Authentication method | Password | Password + MFA | Is the required assurance level enforced? |
| Challenge state | Completed | Pending/failed | Can incomplete proof authorize the action? |
| Entry point | Main web route | Supported API route | Is enforcement consistent? |
| Session state | Active | Revoked/expired | Is session state respected? |
| Operation | Ordinary profile change | Sensitive security change | Is the assurance proportionate and enforced? |
| User privilege | Ordinary | Administrator | Are role-specific requirements applied? |

Change one variable at a time; don't compare unrelated operations and infer a vulnerability without first confirming the same security requirement should actually apply to both.

## I. Exploitation and validation

Minimal proof for every finding class here follows the same shape: confirmed policy, a session that demonstrably hasn't met it, and the protected operation's resulting state actually changing — not a status code, the actual state. For context-confusion findings specifically: demonstrate a challenge or authentication event issued for operation A gets accepted for operation B, against your own controlled account.

## J. False-positive analysis

Don't assume every sensitive operation needs fresh authentication — establish the actual policy first. Don't confuse a valid session with fresh authentication — they're different things even when both hold. Don't confuse MFA *enrollment* with MFA *completion this session* — having MFA configured proves nothing about whether the current session satisfied it, and conversely, a request missing an MFA field doesn't prove MFA wasn't completed earlier via a server-tracked flag. Don't assume every operation needs its own separate challenge — some systems legitimately let one recent authentication cover multiple operations within a window; check the documented scope. A UI prompt is never proof of server-side enforcement. An error response doesn't prove protection — the action may have already succeeded before the error. A successful response doesn't prove a bypass either — it might mean "reauthentication required" or "queued, not yet executed." Don't confuse authorization with authentication assurance — a user can be freshly authenticated but lack permission, or have the right role but insufficient assurance; these are independent checks that both need testing.

## K. Root cause

The recurring patterns: a UI-only prompt with no corresponding server-side check on the actual operation; a freshness check that uses session-creation time instead of last-authentication time; an assurance level collapsed into a single generic flag that can't distinguish method or recency; a challenge marked successful on initiation rather than on validated completion; and shared enforcement middleware that exists but doesn't actually cover every route reaching the protected operation.

## L. Remediation

- Enforce every reauthentication/step-up policy server-side; never rely on the UI prompt as the control.
- Track authentication freshness separately from session validity — use the actual last-successful-authentication timestamp, not session creation time.
- Represent assurance level with enough fidelity to distinguish method and recency, not a single boolean.
- Bind every challenge to its account, session, intended operation, and required method; mark it successful only after genuine validation completes, never on initiation.
- Apply the policy consistently across every route, client, and background workflow that can reach the protected operation.
- Keep authorization checks independent of and in addition to authentication-assurance checks — fresh authentication never substitutes for a permission check.

## M. Finding documentation

Title, summary (operation, required assurance, observed violation), affected component, preconditions (account/role/session/authentication state), expected vs. actual behavior with its policy source, sanitized reproduction, evidence (requests/responses, timestamps, authentication context, proof the action actually took effect), impact (realistic — don't inflate from the phrase "reauthentication bypass" alone), root cause, remediation, severity and confidence with limitations noted.

## Completion criteria

- [ ] Baseline captured for at least one fully-compliant sensitive operation
- [ ] Missing reauthentication tested with the resulting state independently verified, not just the response
- [ ] Fresh vs. stale authentication compared with every other variable held constant
- [ ] Step-up tested by comparing a session that completed it against one that didn't
- [ ] Direct/API access to the operation tested, bypassing any UI-only prompt
- [ ] Every supported interface (web, API, mobile, admin) compared for the same operation
- [ ] Challenge-context binding tested: does a challenge for one operation authorize a different one
- [ ] Each finding traced to a specific enforcement gap, not reported as a generic "reauthentication bypass"
