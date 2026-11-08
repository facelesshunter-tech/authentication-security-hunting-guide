# Source Traceability

Every source identified during Phase 0 discovery, with an honest disposition for each — included, consolidated, referenced, or excluded with a reason. Per-domain source mapping for each of the 17 attack guides lives in `mindmaps/AUTHENTICATION-MASTER-MIND-MAP.md` section 3; this document is the complete picture, including the sources that were examined and *not* used.

## Primary sources — included directly

| Source | Location | Disposition |
|---|---|---|
| `Authentication Complete Phases Test.md` | `/home/faceless/Hunt3r/` (878KB, 25,384 lines) | Original course source. Fully read module-by-module to build the 17-phase course (`authentication-security-engineering` repo) in an earlier stage of this project. Referenced throughout this hunting guide via that course's own 17 phase files, not re-read directly for this deliverable. |
| `Authentication Security.md` | `/home/faceless/Hunt3r/` (205KB) | Supplementary challenge/assessment material for the course. Same disposition as above — consumed to build the course, referenced through it here. |
| `authentication-security-engineering` (17-phase course + site) | GitHub, `facelesshunter-tech/authentication-security-engineering` | Used as the conceptual-depth source for all 17 attack guides. Specific lessons read directly (not from memory) when building guides 02 and 05, where fidelity to exact course wording mattered (password-reset poisoning, MFA lifecycle code patterns). |
| `Testing Approach .txt` | `/home/faceless/Hunt3r/` (1,443 lines) | Read in full. Used as the workflow-framing source for every one of the 17 attack guides' threat-model and methodology framing, and directly built `MASTER-TESTING-METHODOLOGY.md`. |
| `Phases checklist.txt` | `/home/faceless/Hunt3r/` (22,674 lines) | Read in full, phase by phase, at the point each corresponding attack guide was built — not skimmed upfront. This is the primary source for the black-box/gray-box/white-box test content in guides 01–17. See the per-guide "Source" line at the top of each file in `attack-guides/` for the exact line ranges read. |

## Secondary sources — referenced, not directly incorporated

| Source | Location | Disposition and reason |
|---|---|---|
| `Authentication-Vulnerabilities` | GitHub, `facelesshunter-tech/Authentication-Vulnerabilities` | Identified during Phase 0 discovery as real black-box evidence (worked solutions to PortSwigger Web Security Academy authentication labs — 6 of 6 completed, verified directly). Not directly quoted or cited in any of the 17 attack guides, because the guides were built primarily from `Testing Approach .txt` and `Phases checklist.txt`, which already provided richer, more senior-level methodology than the PortSwigger lab writeups' scope covers. Worth drawing on for a future "worked examples against real external labs" supplement if that's ever wanted, but not claimed as a source here. |
| `Phantom-Intruder` | GitHub, `facelesshunter-tech/Phantom-Intruder` | A credential-testing toolkit (Sniper/Pitchfork/Cluster Bomb replication). Identified during the bug-bounty-readiness discussion in this session as evidence of real tooling-building ability. Not incorporated into any attack guide's content — it's a tool, not a methodology source, and nothing in the 17 guides currently references specific tooling by name. |

## Sources examined and excluded, with reason

| Source | Location | Reason for exclusion |
|---|---|---|
| `Application-Security-Assessments` | GitHub, `facelesshunter-tech/Application-Security-Assessments` | Verified directly: the `Bug-Bounty` and `OWASP-Top10` folders contain only `.gitkeep` placeholders — no actual content exists to draw from. The `Web-Security-Academy/SQL-Injection` and `/SSRF` subfolders have content but are outside this course's authentication scope. |
| `Cybersec-Writeups` | GitHub, `facelesshunter-tech/Cybersec-Writeups` | Verified directly: contains one walkthrough (a VulnHub box, `kioptrix-level2`), unrelated to authentication security. |
| `my-favorites-api` | GitHub, `facelesshunter-tech/my-favorites-api` | Verified directly: a trivial FastAPI project with five static GET endpoints and no authentication implementation at all. |
| A live, self-built authentication application | N/A | Confirmed absent during Phase 0 discovery — checked the local workspace, this conversation's history, and the user's full GitHub repository list. No such application exists. The user explicitly decided to proceed black-box-first rather than build one (see `APPLICATION-ASSESSMENT-NOT-APPLICABLE.md`). |

## What remains honestly unread

None of the four primary sources have unread portions as of this document's writing — `Testing Approach .txt` and the course material were read in full; `Phases checklist.txt` was read in full across the process of building all 17 guides (confirmed per-guide in each guide's own source citation). If this course is extended with an 18th guide or a revision to an existing one, re-confirm against this table before claiming a source was used, rather than assuming prior coverage still applies.

## The master prompt's own traceability requirement, restated

Every source above has one of five dispositions: included directly, consolidated into another section, referenced as supporting material, excluded with a clear reason, or confirmed unavailable. Nothing in this course claims a source it didn't actually inspect — where a disposition says "verified directly," that means an actual `gh api` call or file read happened during this session, not an assumption carried over from the source's name or description.
