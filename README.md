# Authentication Security Hunting Guide

A senior-level authentication security mind map and bug-hunting playbook: attack-surface mapping, black-box/gray-box/white-box/differential testing methodology, and vulnerability-class deep dives across every major authentication and authentication-adjacent-authorization domain.

This is a companion to, and built from, the [Authentication Security Engineering course](https://github.com/facelesshunter-tech/authentication-security-engineering) — that course teaches the concepts and the "why"; this repo is the practical hunting reference for actually testing a real target.

## Who this is for

Security researchers, bug bounty hunters, and application security engineers who already understand authentication concepts and want a structured, evidence-driven testing methodology rather than another conceptual overview. If you're new to authentication security, start with the course linked above — this repo assumes that foundation.

## Where to start

- **Running a real assessment right now?** Open `TESTING-QUICK-REFERENCE.md` — the condensed, pure-checkbox version meant to sit open in a second tab while you work.
- **Planning a complete engagement?** Start with `MASTER-TESTING-METHODOLOGY.md` — the full 21-step workflow from scope confirmation to final report, with decision trees for when a test produces an unexpected result.
- **Need the orientation map first?** `mindmaps/AUTHENTICATION-MASTER-MIND-MAP.md` shows every domain and, critically, how they chain together into real attack paths — not just a flat taxonomy.
- **Digging into one specific vulnerability class?** Go straight to the matching file in `attack-guides/` — each one is a complete, self-contained A–M methodology (security objective through finding documentation) for that class.

## Repository structure

```text
README.md                          — you are here
MASTER-TESTING-METHODOLOGY.md      — the 21-step engagement workflow, with decision trees
AUTHENTICATION-ATTACK-SURFACE.md   — the consolidated entry-point/state/trust-boundary inventory
SOURCE-TRACEABILITY.md             — every source used, referenced, or excluded, with reasons
TESTING-QUICK-REFERENCE.md         — condensed checkbox checklist for live engagements

mindmaps/
  AUTHENTICATION-MASTER-MIND-MAP.md     — full taxonomy + cross-domain attack-path diagram
  PASSWORD-AND-CREDENTIALS.md
  SESSION-AND-TOKEN-SECURITY.md
  MFA-AND-ACCOUNT-RECOVERY.md
  OAUTH-OIDC-AND-SAML.md
  IDENTITY-AND-ACCOUNT-LIFECYCLE.md
  AUTHENTICATION-CRYPTOGRAPHY.md
  AUTHORIZATION-AND-OPERATIONAL-SECURITY.md

attack-guides/                     — 17 complete vulnerability-class guides, each A–M:
  01-ATTACK-SURFACE-MAPPING.md         09-ACCOUNT-LIFECYCLE.md
  02-PASSWORD-AND-CREDENTIAL-ATTACKS.md 10-REAUTHENTICATION.md
  03-SESSION-SECURITY.md               11-MULTI-TENANT-IDENTITY.md
  04-JWT-AND-TOKEN-SECURITY.md         12-LEGACY-AUTHENTICATION.md
  05-MFA-ATTACKS.md                    13-FEDERATED-LOGOUT.md
  06-PASSWORD-RESET-AND-RECOVERY.md    14-AUTHENTICATION-LOGGING.md
  07-OAUTH2-AND-OIDC.md                15-CRYPTOGRAPHIC-FAILURES.md
  08-SAML-SECURITY.md                  16-AUTHENTICATION-ADJACENT-AUTHORIZATION.md
                                        17-INFORMATION-LEAKAGE.md

methodologies/                     — the testing techniques, generalized across all 17 guides
  BLACK-BOX-TESTING.md
  GRAY-BOX-TESTING.md
  WHITE-BOX-TESTING.md
  DIFFERENTIAL-TESTING.md
  FALSE-POSITIVE-ELIMINATION.md

templates/                         — fillable forms for live use
  AUTHENTICATION-TEST-CASE.md
  HTTP-EVIDENCE.md
  VULNERABILITY-REPORT.md
  SOURCE-CODE-REVIEW.md
  FINAL-ASSESSMENT.md

application-assessment/
  APPLICATION-ASSESSMENT-NOT-APPLICABLE.md  — why there's no app-specific matrix here, and why
```

## What this is built from, and what it isn't

Every attack guide is reconciled from real source material — a 1,443-line end-to-end testing workflow, a 22,674-line granular black-box/gray-box/white-box checklist, and the companion course's own 17 phases — never invented to fill a gap. Where a guide's source material was thinner than the others (MFA, legacy authentication), that's stated plainly in the guide itself rather than papered over. See `SOURCE-TRACEABILITY.md` for the complete, honest accounting of every source used, referenced, or excluded.

This course is deliberately black-box-first: no live, self-built authentication application exists as a white-box case study, by explicit decision rather than oversight (see `application-assessment/APPLICATION-ASSESSMENT-NOT-APPLICABLE.md`). White-box sections throughout describe vulnerable implementation patterns generically rather than tracing a specific named codebase.

Two things are explicitly out of scope throughout, carried forward from the source material rather than silently dropped: deep JWT exploitation beyond algorithm confusion and weak-secret specifics, and dedicated race-condition exploitation.

## A note on how this was built

Every attack guide, mind map, and methodology file in this repository was built the same way the companion course was: read the actual source material in full before writing anything, reconcile rather than invent, and flag honestly wherever a source was thinner or a decision was a judgment call rather than a clean derivation. Nothing here claims coverage it doesn't actually have.
