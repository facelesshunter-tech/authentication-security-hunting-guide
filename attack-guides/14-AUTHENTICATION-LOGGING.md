# 14 — Authentication Logging and Monitoring

**Source:** `Phases checklist.txt` Phase 15 "Authentication Logging and Monitoring" (read in full, lines 18724–20093 — 10 BB tests, 7 GB areas, 7 WB tracing areas), `Testing Approach .txt` Phase 13, and the course's Phase 15.

**The distinction that governs this entire guide: logging, detection, alerting, and incident response are separate capabilities.** An event appearing in a log doesn't prove an alert was generated. An alert firing doesn't prove the right person received it. A dashboard showing a failed login doesn't prove a coordinated credential attack would actually be caught. Test each stage independently — the pipeline is `authentication event → generation → delivery → detection → response`, and each link can fail on its own without the others showing it.

## A. Security objective

Security-relevant authentication events must be recorded accurately, protected from unauthorized access or tampering, and made available to the detection and response controls the system's risk profile actually requires. A fifth dimension matters alongside generation/delivery/detection/response: event integrity — can the organization actually trust the recorded evidence, or could events be missing, duplicated, delayed, misattributed, or exposed to the wrong party?

## B. Attack surface

Login success/failure events, account-enumeration telemetry, rate-limit and brute-force detection, MFA security events, password-reset/recovery monitoring, session/logout event logging, federated (OAuth/OIDC/SAML) authentication events, privileged-account monitoring, log integrity and access control, and monitoring-pipeline resilience itself.

## C. Threat model

This guide isn't about an attacker exploiting a technical flaw to gain access — it's about whether an attacker's *activity*, once it happens, is visible to anyone. The realistic question throughout: if a credential-stuffing run, an MFA-fatigue attack, or a session-hijack attempt happened against this application right now, would any human actually find out, and how long would that take? A secondary threat model: an attacker (or a malicious insider) who can read or tamper with authentication logs themselves, either to harvest exposed secrets or to erase evidence of an attack in progress.

## D. Black-box testing

**Login event accuracy.** Sign in successfully, sign out, fail once with a wrong password, fail once against a nonexistent account, then check whatever logs or dashboard you're authorized to see. Specifically verify: is the event missing entirely; is a failed attempt ever recorded as successful; is the wrong user attributed; are credentials visible in the log itself. A generic user-facing error is not evidence either way about what was recorded internally — these serve different purposes and must be checked separately.

**Account-enumeration telemetry.** Compare a failed login against a known account vs. a nonexistent one. Good monitoring returns a generic *external* error in both cases while retaining precise *internal* reason codes (`invalid_credentials`, `unknown_account`, `account_disabled`, `mfa_required`) that let detection distinguish real attack patterns without ever exposing that distinction to the attacker. The finding: internal telemetry that can't make this distinction at all, or a monitoring platform that records the wrong account.

**Rate-limit and brute-force detection.** Within a small, pre-approved attempt count: repeated failures against one account (classic brute force or a confused legitimate user — context matters), failures against many accounts from one source (spraying), failures against one account from many sources (distributed attack or shared network — needs contextual analysis either way). Keep rate limiting (a control that *stops* attempts), monitoring (a control that *records/analyzes* them), and alerting (a control that *notifies* someone) conceptually separate — a system can have any one without the other two, and a missing alert doesn't automatically mean brute force succeeds unblocked.

**MFA security-event monitoring.** Complete a normal challenge, fail one, let one expire if supported. Check specifically whether repeated failures are recorded distinctly from a single ordinary failure, whether MFA *removal* generates a security event (this is a defined monitoring requirement in most mature programs, not optional), and whether a privileged account's MFA downgrade would actually be detectable.

**Password-reset and recovery monitoring.** Request a reset, complete it, attempt one invalid/expired token. The high-value detection scenario worth testing deliberately: a completed reset followed shortly by a new session from an unusual context and a change to the recovery email — that *sequence* deserves monitoring attention even though no single event in it proves compromise on its own.

**Session and logout event logging.** Log in, access a protected resource, log out, replay the old session as authorized. Check whether session creation, revocation, and (critically) *attempted use of a revoked session* are each individually logged — a correctly-enforced revocation with no corresponding log entry is a monitoring gap even though authentication itself is working fine; conversely a logged revocation event alongside a session that's still actually accepted is primarily an enforcement defect with a secondary monitoring angle.

**Federated (OAuth/OIDC/SAML) event monitoring.** Complete a normal federated login, trigger one approved failure, log out, then correlate the flow across application and IdP events using trace/request IDs. Specifically check for coverage of invalid-assertion events, failed state/nonce/issuer/audience/signature validation, and SAML replay-protection triggers — not just success/failure at the top level.

**Privileged authentication monitoring.** Login, failed login, required MFA/step-up, a privileged action, logout — for an authorized privileged test account. Not every admin login needs an immediate high-severity alert (baseline-deviation or periodic-review models can be legitimate) — your job is to verify the *documented* monitoring requirement is actually implemented, not to assume a specific alerting posture is mandatory.

**Log integrity and access control.** Can ordinary users view another user's authentication history without authorization; can application admins alter security logs without appropriate controls; are logs stored somewhere publicly accessible; does retention match documented requirements. Never attempt to actually tamper with production logs — verify through approved audit records or access-control testing instead.

**Monitoring resilience and delivery failures.** Generate a small approved event and verify it reaches central monitoring within the expected time window; compare source and destination timestamps for lag; check whether an ingestion-failure alert exists for the monitoring pipeline itself. A missing event is the *start* of an investigation, not the end — it could mean never generated, lost in transport, rejected by parsing, excluded by a filter, or simply stored somewhere you haven't checked yet.

## E. Gray-box testing

Map the full architecture: which component generates each event, which transports it, how events correlate across services, what happens on delivery failure, how duplicates are handled. Compare source events against centralized events directly — if the application logs a failure but the central platform never received it, the gap is between generation and ingestion (investigate transport); if the central platform has it but no alert fired, the gap is in the detection rule itself (investigate threshold/grouping/conditions) — these require completely different remediation, so pin down *which* stage failed before reporting. Review detection-rule coverage for the full event catalog (repeated failures, credential-stuffing patterns, recovery-sequence anomalies, MFA abuse, privileged anomalies, token reuse, session persistence after logout) and for each rule record its purpose, data source, threshold, time window, grouping key, and known exceptions — a rule *existing* is not proof it works; you need its input events actually available and its logic matching the stated intent. Review correlation and identity attribution specifically for the common defects: an IdP using a different identifier than the application with no mapping, events grouped by username despite usernames not being globally unique, a shared NAT address causing unrelated users to appear as one actor, clock skew making a valid sequence look out of order. Review sensitive-data handling directly in representative log events — search for passwords, session cookies, bearer/refresh tokens, authorization codes, MFA secrets, reset tokens, private keys — a token-like string's mere presence doesn't prove a live leak, confirm it's real, valid, and reachable by an unauthorized party before calling it a finding. Review alert delivery end to end: was the alert actually routed to the right team, did it contain correct identity/application context, was it delivered within the required timeframe — an alert appearing in a dashboard is not proof a human was ever notified. Review exclusions and suppressions for undocumented blind spots, especially whether privileged accounts or service accounts are quietly excluded from rules meant to cover them, and whether new authentication routes (new MFA method, new passkey flow, new legacy-compat endpoint) actually generate events the monitoring platform understands.

## F. White-box testing

**Trace event generation**: `request → authentication logic → outcome → event-generation function → logging interface`. The realistic defect: the success branch correctly emits an event, but the exception handler for invalid credentials returns early *without* logging the failure — trace every branch, not just the happy path, and specifically confirm logging failures can never accidentally be read as an authentication success.

**Review structured event schemas and normalization.** The exact, concrete defect pattern worth hunting for: the application emits `event.outcome = "failure"` while the detection rule expects `event.outcome = "failed"` — the event reaches the platform completely intact and the rule simply never matches it. This is a silent, easy-to-miss failure mode precisely because every individual component looks correct in isolation.

**Trace event delivery and failure handling.** Are delivery failures themselves detectable (not just silently dropped); is queue overflow monitored; does authentication fail open or closed when the logging service itself is unavailable (there's no universal right answer here — it depends on the threat model — but the organization needs to *know* which behavior it has and have a compensating control for the resulting visibility gap).

**Review detection logic**: `normalized event → rule condition → grouping key → time window → threshold → suppression → alert action`. The specific defect worth testing for: a rule that groups failed logins by source IP only — an attacker distributing attempts across many IPs sails straight past per-account-only detection. The fix isn't necessarily "add more grouping keys" reflexively — shared IPs, NAT, privacy constraints, and false-positive rates all factor in; the right design usually combines multiple complementary signals rather than leaning on one identifier.

**Trace alert routing**: `detection result → alert creation → routing rule → notification service → recipient`. The specific defect worth isolating: the detection rule and the underlying event are both completely correct, but the routing configuration simply excludes the affected application — report this as a routing failure, not a detection failure, since the fix is entirely different.

**Review log access/integrity controls** in code and platform config: who can actually read sensitive authentication events, is write access limited to approved ingestion identities, are administrative log modifications themselves audited.

**Review tests for monitoring controls themselves.** A test that only asserts "the logging function was called" is meaningfully weaker than one that verifies a correctly structured event actually reaches the downstream detection component — check which kind of test coverage actually exists.

## G. Differential testing

Nine-step workflow: define the expected event (and whether an alert is actually required for this scenario), generate a controlled action, observe external behavior, inspect source telemetry (did generation happen), inspect centralized telemetry (did delivery/parsing/normalization survive), inspect the detection rule (did the event satisfy its actual conditions), verify alert delivery, trace the implementation to the exact missing/incorrect/misconfigured point, then reproduce against the fix.

| Scenario | Black-box | Gray-box | White-box |
|---|---|---|---|
| Failed login not recorded | Response + missing event | Source vs. central logs | Failure-branch event generation |
| Suspicious failures not detected | Controlled pattern, no alert | Rule config and event stream | Detection logic and grouping keys |
| MFA abuse not monitored | Controlled MFA failures | MFA events and detection results | MFA handler event generation |
| Logs contain secrets | Redacted controlled evidence | Log records and access perms | Logging statements and serializers |
| Alert never delivered | Detection result, no notification | Routing and delivery records | Alert routing integration |
| Incorrect normalization | Observed event differs from schema | Parsed vs. raw event | Parser and schema mapping |

## H. Exploitation and validation

Minimal proof for a generation gap: demonstrate the specific controlled action with no corresponding source event, via your own authorized logs. Minimal proof for a detection gap: demonstrate the event reaching the central platform with the rule's documented condition satisfied, yet no alert created. Minimal proof for a secrets-in-logs finding: locate your own controlled test credential's actual value inside a log record you can access, with a clear chain showing who else could reach it.

## I. False-positive analysis

No alert doesn't mean no monitoring — the event may be logged but simply not configured to alert, or your test may not have met the rule's actual threshold. Not every failed login should alert — individual failures are routine; the finding requires the defined *pattern*. An alert firing proves activity matched a condition, not that an attack actually occurred — correlate further before escalating. An event being present somewhere doesn't mean logging "works" — it may be incomplete, delayed, or missing required fields; compare against the actual schema. A logging function existing in source doesn't prove it's effective — it may not run on every path, its output may get dropped, or no rule may consume it. A missing event doesn't prove an attacker bypassed logging — it may have been generated and lost downstream. High failure volume doesn't automatically prove credential stuffing — legitimate user error, shared networks, and misconfigured clients can all produce it; correlate multiple signals. A monitoring outage isn't harmless just because authentication kept working — the organization may have lost all visibility into attacks during that window regardless.

## J. Root cause

The two dominant patterns: an event-generation path that's incomplete (certain failure branches, especially exception handlers, never call the logging function at all), and a schema mismatch between what gets produced and what the detection rule actually expects (`"failure"` vs. `"failed"`) — both failures are invisible from either side alone, since the producer looks correct and the rule looks correct; only tracing the full pipeline end to end exposes them.

## K. Remediation

- Define one consistent event schema across every producer and consumer; test schema changes against the actual detection rules before deployment, not independently.
- Ensure every authentication code path — success, failure, exception, timeout — calls the event-generation function; logging failures must never be mistaken for or converted into authentication success.
- Monitor the telemetry pipeline itself (delivery health, ingestion lag, parsing failures) as a first-class operational concern, not an afterthought.
- Design detection rules around multiple complementary signals (account, source, pattern, context) rather than a single grouping key that's trivially evaded.
- Route alerts explicitly per application/severity and verify actual delivery, not just alert creation.
- Restrict log read/write access by role, audit administrative modifications, and keep raw secrets out of event payloads entirely — redact or use non-reversible correlation values.

## L. Regression testing

Add tests that verify a correctly *structured* event reaches the actual downstream detection component — not just that a logging function was called. Re-run the full differential workflow after any change to event schema, detection rule logic, or alert routing configuration, since any one of those three can silently break the chain without the other two showing symptoms.

## M. Finding documentation

Title, affected component (application, IdP, event pipeline, detection rule, storage, or alerting integration), expected vs. observed behavior, preconditions (test account, flow, approved test conditions), minimal reproduction, evidence (timestamped action, application response, source event if available, centralized event if available, rule evaluation result, alert-delivery evidence, redacted correlation IDs), security impact (what attack/incident could go undetected, or what's actually exposed), root cause (the specific stage that failed), remediation, retest confirmation (generated → delivered → evaluated → alerted, the full chain, plus a benign comparison to rule out false positives), severity rationale grounded in realistic undetected-attack impact, not the mere existence of a gap.

## Completion criteria

- [ ] Login success/failure events verified for accuracy, not just presence
- [ ] Account-enumeration telemetry tested for internal reason-code precision without external disclosure
- [ ] MFA removal and privileged-account MFA downgrade specifically tested for event generation
- [ ] Session revocation and attempted-use-of-revoked-session each tested as distinct logged events
- [ ] Federated authentication failure modes (bad state/nonce/issuer/audience/signature) tested for event coverage
- [ ] Source-vs-centralized event comparison performed to isolate generation vs. delivery vs. detection failures
- [ ] Logs searched for actual secret values (credentials, tokens) using controlled test data
- [ ] Alert delivery verified end to end, not assumed from dashboard presence
- [ ] Each finding traced to the specific pipeline stage that failed, with the other stages explicitly ruled in or out
