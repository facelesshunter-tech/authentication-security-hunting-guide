# Final Assessment Template

The wrap-up document for a complete engagement — fill this out once you've worked through whichever attack guides applied to the target, not per individual finding (use `VULNERABILITY-REPORT.md` for those).

```text
Target:
Scope and authorization reference:
Assessment dates:
Tester(s):

--- Coverage summary ---
For each attack guide, mark its status. "Not applicable" requires a
reason (e.g. "target has no SAML integration") — never leave a guide
unmarked.

| # | Guide | Status | Notes |
|---|---|---|---|
| 01 | Attack Surface Mapping | [Covered / Partial / N/A] | |
| 02 | Password and Credential Attacks | | |
| 03 | Session Security | | |
| 04 | JWT and Token Security | | |
| 05 | MFA Attacks | | |
| 06 | Password Reset and Recovery | | |
| 07 | OAuth 2.0 and OIDC | | |
| 08 | SAML Security | | |
| 09 | Account Lifecycle | | |
| 10 | Reauthentication and Step-Up | | |
| 11 | Multi-Tenant Identity | | |
| 12 | Legacy Authentication | | |
| 13 | Federated Logout | | |
| 14 | Authentication Logging | | |
| 15 | Cryptographic Failures | | |
| 16 | Authentication-Adjacent Authorization | | |
| 17 | Information Leakage | | |

--- Findings register ---
List every validated finding (not leads — see AUTHENTICATION-TEST-CASE.md
for those). One row per finding, full detail lives in its own
VULNERABILITY-REPORT.md.

| ID | Title | Vulnerability class | Guide | Severity | Status |
|---|---|---|---|---|---|
| | | | | | [Open / Reported / Fixed / Retested-confirmed-fixed] |

--- Severity distribution ---
Critical:
High:
Medium:
Low:
Informational:

--- Unresolved leads ---
Observations that didn't reach confirmed-finding status within this
engagement's scope or time, worth flagging for a follow-up:


--- Overall posture statement ---
[A few honest paragraphs: where the application's authentication and
authorization controls are genuinely solid, where the real risk
concentrates, and what the highest-leverage next step would be if
only one thing gets fixed first. This is the section a reader who
skips everything else should still get real value from.]

--- Explicitly out of scope for this assessment ---
[Anything the master prompt or engagement scope excluded — e.g. deep
JWT exploitation beyond algorithm confusion, dedicated race-condition
exploitation, or an application-specific code review if no live
application was available. State it plainly rather than letting a
reader assume silence means "tested and clean."]
```

## Why "not applicable" needs a reason

A blank row in the coverage table is indistinguishable from a forgotten one. Every guide this course ships with should end up in one of three states for a given engagement: genuinely tested, explicitly not applicable with a stated reason, or explicitly out of time/scope — never silently absent.
