# Master Testing Methodology

The complete, repeatable workflow for a full authentication security assessment, from scope confirmation to final report. This is the *sequence* — for the testing *techniques* used within each step, see `methodologies/`; for the vulnerability-class-specific depth, see `attack-guides/`; for what to have open while actually hunting, see `TESTING-QUICK-REFERENCE.md`.

Built primarily from `Testing Approach .txt`'s 18-phase workflow (read in full), reconciled with this course's own 17-guide structure.

**How to use this**: follow the sequence to establish a baseline and map the real identity system, then *follow evidence*, not the numbered order. If a weakness in step 9 points directly at a question that belongs in step 13, go there immediately rather than finishing every step in between first. The numbering is for completeness-checking afterward, not a rigid script.

## The 21 steps

### 1. Scope confirmation
Read the program's or engagement's scope: permitted domains, subdomains, APIs, mobile apps, environments. Read prohibited-testing rules (automated testing, brute force, account creation, destructive operations). Confirm whether production testing is permitted. Prepare controlled test accounts and data before touching anything else.

### 2. Application reconnaissance
Open the target as an ordinary user would. Identify the technology stack where visible, the general architecture (single app, multi-tenant SaaS, federated), and which of the 17 attack-guide domains are even plausibly in play — not every target has SAML, passkeys, or multi-tenancy.

### 3. Authentication architecture mapping
Use `attack-guides/01-ATTACK-SURFACE-MAPPING.md` in full here. Identify every entry point: registration, login, logout, password change/reset, verification, MFA enrollment/verification/recovery, OAuth/OIDC, SAML, account linking, session management, reauthentication, deletion, admin routes, legacy routes, API/mobile routes.

### 4. Identity and role mapping
Document the actual role and tenancy model: can a user belong to multiple tenants, is a role global or per-tenant, which identities are platform-level vs. organization-level administrators. This directly feeds `attack-guides/11-MULTI-TENANT-IDENTITY.md` and `attack-guides/16-AUTHENTICATION-ADJACENT-AUTHORIZATION.md`.

### 5. Account and session inventory
Build the controlled-identity set every guide assumes: at minimum an ordinary verified account, a second ordinary account (for cross-user testing), an unverified account, a restricted/disabled account if available, and an administrator account if explicitly authorized. Record each one's exact state.

### 6. Endpoint inventory
From step 3's map, build the concrete request list — method, path, required auth state, expected response — that steps 9 through 12 will actually execute against. This is the `AUTHENTICATION-TEST-CASE.md` setup phase at scale.

### 7. Baseline behavior collection
Before attempting anything adversarial, record what *legitimate* use looks like for every entry point in scope: successful login, successful MFA, successful reset, successful federated login. You cannot recognize a violation without first establishing what compliance looks like.

### 8. Threat modeling
For each domain in play, state the realistic attacker capability and required access per the relevant attack guide's own "C. Threat model" section. This determines which of the 17 guides deserve the most time — a target with no federation support doesn't need deep OAuth/SAML time; a multi-tenant SaaS product needs guide 11 early and often.

### 9. Black-box testing
Work through the relevant attack guides' black-box sections. See `methodologies/BLACK-BOX-TESTING.md` for the cross-cutting discipline.

### 10. Gray-box testing
Where any internal visibility is authorized (docs, logs, config, schemas). See `methodologies/GRAY-BOX-TESTING.md`.

### 11. White-box testing
Where source code is available. See `methodologies/WHITE-BOX-TESTING.md`.

### 12. Differential testing
The connective technique across 9–11. See `methodologies/DIFFERENTIAL-TESTING.md`. This is where most leads either firm up into findings or get ruled out.

### 13. Attack-path analysis
Ask whether one weakness compounds another — see the master mind map's relationship diagram (`mindmaps/AUTHENTICATION-MASTER-MIND-MAP.md`, section 2) for the common chains this course has already mapped: enumeration into credential attacks, JWT weaknesses into forged sessions, SAML signature gaps into spoofed SSO identity, stale authorization into cross-tenant access. These are investigative hypotheses, not claims — validate each link independently before treating the chain as real.

### 14. Exploitability validation
For each surviving lead, establish the minimal reproduction using only your own controlled accounts and resources. Every attack guide's "H. Exploitation and validation" section states the specific minimal-proof standard for that vulnerability class.

### 15. False-positive elimination
Before writing anything up, run every surviving lead against `methodologies/FALSE-POSITIVE-ELIMINATION.md`'s nine fallacy patterns. This step is cheap and is the single highest-leverage place to catch an embarrassing report before it goes out.

### 16. Impact assessment
State what the finding actually permits — confidentiality, integrity, availability, privilege, business consequence — grounded in demonstrated evidence, not the vulnerability class's reputation. A BOLA exposing one low-sensitivity field and a BOLA exposing a full financial record are not the same severity.

### 17. Evidence collection
Use `templates/HTTP-EVIDENCE.md` for every request/response pair kept, with redaction applied as you go, not retroactively.

### 18. Root-cause analysis
Every attack guide's "J. Root cause" section names the dominant patterns for that vulnerability class — decode-vs-verify, fetch-then-check instead of scope-in-query, a UI-only control with no server-side twin, duplicated logic that drifted between a modern and legacy path. Name the actual mechanism, not just the symptom.

### 19. Remediation recommendations
Use each guide's "K. Remediation" section as the baseline, adapted to the target's actual architecture. Never recommend "sanitize input" or "add security" as a standalone fix — tie the recommendation to the specific root cause from step 18.

### 20. Regression testing
Use each guide's "L. Regression testing" section to know what to re-check after a fix ships, and confirm both directions: the attack no longer works, *and* legitimate use still does.

### 21. Final reporting
Use `templates/VULNERABILITY-REPORT.md` per finding and `templates/FINAL-ASSESSMENT.md` for the engagement as a whole.

## Decision trees for unexpected results

**A test you expected to fail instead succeeds (a denial you expected didn't happen).**
→ First confirm the test identity actually reached the authorization/validation path at all (an expired session failing earlier in the pipeline looks identical to a missing check, but is a different root cause). → Then confirm the resource/action genuinely was outside the account's permitted scope (check for legitimate sharing, administrative scope, or tenant membership you didn't know about). → Only then treat it as a real finding and move to step 13 (attack-path analysis) to see if it chains into anything.

**A test you expected to succeed instead fails (legitimate access broken).**
→ This usually means your baseline (step 7) was wrong, not that you found a security control. Re-verify the account's actual state before concluding anything. → If the baseline really was correct and it still fails, you may have found an availability/reliability bug, not a security finding — categorize it honestly rather than forcing it into a security narrative.

**Two responses differ, but you're not sure the difference matters.**
→ Go straight to `methodologies/DIFFERENTIAL-TESTING.md`'s single-variable discipline. Re-run the comparison changing *only* the one variable you actually care about. → Check the difference against `methodologies/FALSE-POSITIVE-ELIMINATION.md` pattern 4 (difference-without-violation) before spending more time on it.

**A white-box review finds a missing check, but black-box testing against the same path succeeded normally.**
→ The check is probably enforced somewhere else on the same path that you haven't found yet (middleware, a database constraint, a downstream service). Keep tracing before reporting — see `methodologies/WHITE-BOX-TESTING.md`'s reachability discipline.

**A finding seems to chain into something much bigger than the individual weakness suggests.**
→ Validate every link in the chain independently per step 13 and step 14's minimal-proof standard. Do not report the chain's theoretical maximum impact — report the impact you actually demonstrated, and note the chain as a documented hypothesis if you couldn't complete every link safely within scope.

## Explicit scope boundary carried through this entire course

Deep JWT exploitation beyond algorithm confusion and weak-secret specifics, and dedicated race-condition exploitation, were flagged as out of scope in the original source material (`Testing Approach .txt`) and were never invented beyond what that material actually supports. If either becomes relevant to a real engagement, treat it as a deliberate extension beyond this course's documented coverage, not a gap in its fidelity to its sources.
