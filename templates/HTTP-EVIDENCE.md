# HTTP Evidence Template

Copy this block once per request/response pair you want to preserve as evidence. Designed to be pasted directly into a finding write-up (`VULNERABILITY-REPORT.md`) or a test case (`AUTHENTICATION-TEST-CASE.md`) without further editing, which is why the redaction reminders are built into the structure itself rather than left to memory.

```text
Label:                  [what this request demonstrates, in one line]
Captured as:            [account/role used to generate it]
Timestamp:

Request:
METHOD /path HTTP/1.1
Host: [use the target's real authorized host if this is a genuine test
       against an in-scope target; use app.example.test or another
       RFC 2606 reserved domain if this is illustrative only]
[relevant headers]
Cookie: [REDACTED — name only if the name itself matters, e.g. "session=<REDACTED>"]
Authorization: [REDACTED if bearer/basic]

[request body — redact password, token, and secret fields individually,
 do not redact the whole body if structure matters to the finding]

Response:
HTTP/1.1 [status] [reason]
[relevant response headers]
Set-Cookie: [REDACTED if present]

[response body — redact any live token, session ID, or secret value;
 if the finding IS the leaked secret, redact everything except a
 fingerprint (first/last few characters, length, or a hash) sufficient
 to prove its presence without reproducing a usable credential]

--- What this proves ---
[one or two sentences tying this exact evidence to the specific claim
 it supports — not a general description of the vulnerability]
```

## Redaction checklist, every time

- [ ] Passwords — never included, not even redacted-looking placeholders that could be mistaken for real
- [ ] Session cookies and bearer/access/refresh tokens — redacted to a fingerprint, not the full value
- [ ] Password-reset and verification tokens — same treatment as session tokens
- [ ] MFA secrets, recovery codes, and backup codes — never included
- [ ] Real personal data (email, phone, name) belonging to anyone other than your own controlled test accounts — redacted or replaced with synthetic test data
- [ ] Client secrets, API keys, and signing keys — never included

## Labeling discipline

Every piece of HTTP evidence in this course's attack guides is explicitly one of: real evidence against an authorized target, or an illustrative example using a fictional domain. Never let the two blur together in a report — a reader (or a future you) must be able to tell at a glance whether a request actually happened against the target or is a worked example of the technique.
