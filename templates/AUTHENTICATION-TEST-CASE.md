# Authentication Test Case Template

Copy this block once per individual test. This is the per-test record every attack guide's black-box/gray-box/white-box sections assume you're keeping — not the finished finding, just the raw evidence that will later support or rule out one.

```text
Test ID:                 [e.g. BOLA-03, MFA-07]
Attack guide reference:  [e.g. 16-AUTHENTICATION-ADJACENT-AUTHORIZATION.md]
Date / time:

--- Setup ---
Test account(s) used:    [account ID, role, tenant, verification state, MFA state]
Target resource:         [resource ID, owner, tenant — if applicable]
Starting authentication
state:                   [anonymous / password-accepted / MFA-pending / fully authenticated / step-up-pending]

--- Expected policy ---
State the policy BEFORE running the test, not after observing the result:


--- Request ---
Method + path:
Relevant headers:
Relevant cookies (redacted):
Request body (redacted where sensitive):

--- Response ---
Status code:
Relevant response headers:
Response body (redacted where sensitive):
Redirect destination (if any):
Timing (if relevant, multiple samples):

--- Resulting state (verified independently, not assumed from the response) ---
Did the protected resource/account/session actually change?
How was this confirmed? [re-read via an authorized channel / audit log / second account]

--- Interpretation ---
Does the observed result match the expected policy?      [ ] Yes   [ ] No   [ ] Inconclusive
If No: what specific check appears to be missing or incorrect?

--- Confidence ---
[ ] Unconfirmed lead        [ ] Needs gray/white-box corroboration        [ ] Validated finding        [ ] Not applicable (policy doesn't require this)

--- Next action ---
[e.g. repeat with a second controlled account / inspect source for the relevant check / escalate to a full finding write-up using VULNERABILITY-REPORT.md]
```

## Notes on using this template

- One test case per variable changed. If you're running a differential comparison (see `methodologies/DIFFERENTIAL-TESTING.md`), fill out a baseline test case and a separate comparison test case — don't merge them into one record, or you lose the ability to show exactly what changed.
- "Resulting state" is the field most often skipped and most often the difference between a real finding and a false positive. Never leave it blank just because the response looked conclusive.
- Redact session tokens, passwords, and reset/verification tokens in every field — including ones you don't intend to share yet. It's easier to never write the raw secret down than to redact it later.
