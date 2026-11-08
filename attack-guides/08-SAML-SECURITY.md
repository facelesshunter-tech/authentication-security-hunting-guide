# 08 — SAML Security

**Source:** `Phases checklist.txt` Phase 10 "SAML 2.0 Security" (read in full, lines 11003–12342 — 12 BB tests, 7 GB areas, 8 WB tracing areas, plus a dedicated XML Signature Wrapping investigation), `Testing Approach .txt` Phase 10, and the course's Phase 10.

**Testing principle, stated directly by the source material and worth repeating because it governs everything in this guide:** never assume a SAML response is trustworthy merely because it contains a valid-looking XML document or signature. A correctly signed assertion does not automatically mean the service provider should accept it — the signature can be valid while the issuer, audience, recipient, time conditions, request correlation, or identity mapping is wrong.

## A. Security objective

The service provider must create a session only when the received SAML message satisfies its complete trust and validation policy for the intended authentication transaction, and maps to the correct local identity. Every one of those checks — signature, issuer, audience, recipient, timing, correlation, identity mapping — is independent; passing one proves nothing about the others.

## B. Attack surface

SAML SSO and Assertion Consumer Service (ACS) endpoints, both SP-initiated and (if supported) IdP-initiated flows, the identity-mapping/account-provisioning logic, and — where in scope — Single Logout endpoints.

## C. Threat model

The realistic attacker rarely needs to forge a signature. They need an assertion that's validly signed by *some* trusted key but wrong for *this* context: wrong issuer, wrong audience, outside its validity window, not correlated to the transaction that was actually initiated, replayed from an earlier login, or structured so that the signed element and the element the application actually reads identity from are different (XML Signature Wrapping). Required capability ranges from none (most of these are testable with your own controlled identity) to control of a test IdP for the more advanced structural tests.

## D. Black-box testing

**Capture the legitimate baseline first.** Full SSO login with a controlled account, recording every relevant field: response `ID`/`InResponseTo`/`Destination`/`Issuer`, assertion `ID`, `NameID`, subject confirmation data, `Recipient`, `NotBefore`/`NotOnOrAfter`, `AudienceRestriction`, authentication context, and signature/reference info. This baseline is the reference point for every comparison that follows — not every field appears in every valid message, interpret according to the actual deployed profile.

**Signature enforcement.** Using a controlled IdP or approved test harness, compare a valid signed response against one with an invalid signature, one signed by an untrusted key, and one missing a signature where the deployment requires it. The real question is whether the *required signed security statements* are validated before identity is trusted — SAML deployments may sign the response, the assertion, or both, and an absent response signature is not a finding if the assertion signature is correctly required and validated.

**Issuer validation.** Compare the legitimate issuer against a message asserting a different, untrusted one. An application can legitimately trust multiple IdPs — an unfamiliar issuer isn't automatically untrusted, establish the actual intended trust configuration first.

**Audience restriction.** Compare a legitimate assertion against one whose audience doesn't identify this service provider. Without audience enforcement, material issued for one relying party can be accepted by another that should reject it entirely — this is a distinct, important check independent of signature validity.

**Destination/recipient validation.** Compare legitimate endpoint values against mismatched `Destination`/`Recipient`/ACS values. Applicable rules depend on the profile — don't assume every field must always be present.

**Request-response correlation.** For SP-initiated flows: record the expected correlation identifier, then test how the service provider handles a response that doesn't correspond to the expected outstanding request. IdP-initiated SSO legitimately starts without an SP-originated request to correlate — absence of `InResponseTo` there is not automatically a finding; evaluate the supported flow and its actual compensating protections.

**Time conditions.** Compare an assertion within its validity window against one not-yet-valid and one expired. A small, documented clock-skew tolerance is legitimate — the finding is an assertion rejected outside even that configured tolerance being accepted anyway.

**Replay resistance.** Submit the same approved test message a second time within the permitted testing scope and observe whether it's accepted and what session results. A second response alone doesn't prove a replay vulnerability — some systems handle repeated submissions idempotently; the finding is a replayed assertion producing an unintended *new* authentication or security-relevant effect.

**Identity mapping and NameID handling.** Test how the application maps a valid existing user, a valid-but-unprovisioned user, a user with a different NameID format, a user whose email differs from the NameID, and a user from a different configured IdP. Don't assume email is the authoritative account key in every deployment — establish what actually is, and whether the IdP is even authorized to assert that attribute with meaningful verification behind it.

**Automatic provisioning.** Test whether an unprovisioned identity gets silently provisioned, whether role/tenant assignment comes from trusted IdP claims or server-side rules, and whether changing IdP-side attributes changes local privileges. Automatic provisioning isn't inherently insecure — the question is whether provisioning and attribute trust actually match the intended policy.

**Authentication-context enforcement.** Where the application has an explicit assurance requirement (e.g., a privileged function requiring MFA-backed authentication), test whether it accepts an assertion indicating only a weaker method. Not every SAML application needs to enforce a specific context — establish the actual requirement first.

**XML parser and signature-reference behavior — this is the entry point for XML Signature Wrapping**, covered in depth in the dedicated subsection below. At the black-box level: in a dedicated authorized environment, test whether malformed or structurally ambiguous messages are rejected safely. Never run parser-stress or XXE-adjacent tests against production without explicit authorization and safeguards.

## E. Gray-box testing

**Metadata.** Review entity IDs, ACS/SLO endpoints, binding types, signing certificates, supported NameID formats, and key-use declarations against the actual deployed trust relationship — an obsolete certificate still active past its rollover policy, or an ACS endpoint that differs from the documented configuration, are the realistic drift patterns here.

**Signature-validation settings.** Which signatures are required, which algorithms permitted, which keys trusted, and — critically — whether the *validated* assertion is actually the one used to create the session. Don't infer a bypass from a configuration flag's name alone; confirm how the application and library actually interpret it.

**Request-correlation state.** Is a request identifier stored and checked server-side (not just a client-visible value), does it expire, is it consumed after use, can an unrelated response get associated with the same transaction.

**Replay tracking.** Scope and lifetime of stored assertion/response identifiers, whether storage is shared across application instances (a single-instance in-memory cache can be adequate for some deployments, inadequate for others — evaluate the actual architecture), whether the tracking window aligns with the assertion's own validity window.

**Account-mapping configuration.** Which claims does the application actually treat as authoritative — if email is trusted, is the IdP actually authorized to assert verified email, or is that assumption unfounded?

**Logs.** Compare a controlled successful login against a controlled rejected one across response-validation outcome, issuer/audience decision, signature result, correlation outcome, time-condition result, identity-mapping decision, and provisioning decision — logs should capture enough to distinguish *which* validation failed without retaining full assertions or sensitive attributes.

**Deployment differences.** Compare dev/staging/production for different trusted IdPs, debug-mode signature settings, test certificates left active in the wrong environment, weakened audience/recipient checks, or disabled replay tracking — a relaxed test environment is fine, the finding is that relaxation leaking into where it shouldn't be.

## F. White-box testing

**Trace the full pipeline**: `HTTP request to ACS → request/size validation → base64 decode → XML parse → SAML response processing → signature/trust validation → issuer validation → audience/destination validation → time-condition validation → request correlation/replay checks → identity mapping/provisioning → role assignment → session creation → audit event`. The critical question throughout: can any failure path still create a session?

**XML parser configuration.** DTD processing, external entity resolution, entity expansion, resource limits, ID-attribute registration, duplicate-identifier behavior. A SAML app using XML is not automatically XXE-vulnerable — establish whether untrusted input can actually reach a dangerous parser configuration before claiming it.

**Signature validation — the single most important question in this entire guide:** *does the application extract the identity from the same assertion whose signature and conditions it actually validated?* This is the core XSW question. Also check: is trust anchored to controlled configuration, not to a value the incoming message supplies about itself; are disallowed algorithms actually rejected; do validation errors terminate processing or just log and continue.

**Issuer/audience/recipient validation in code.** The realistic root causes: accepting the message's own self-declared issuer as the trust anchor (circular, trusts nothing); comparing against the wrong audience value; applying a validation rule to the response but not to the assertion actually being consumed; treating a validation error as non-fatal. A check existing in a helper function doesn't protect anything unless the real execution path actually calls it and respects the result.

**Time validation.** Boundary comparisons for `NotBefore`/`NotOnOrAfter`, clock-skew configuration, and whether validation genuinely happens before identity acceptance rather than after.

**Request correlation and replay.** If a request ID is "validated," is it compared against the actual outstanding request, or merely checked for syntactic presence? Replay protection should be evaluated against the actual policy, not the mere existence of a cache table.

**Identity extraction and account linking.** Trace NameID, issuer, email, groups/roles, and tenant identifiers through to the local account decision: does the issuer form part of the identity key where it needs to; is email assumed verified without evidence; can a new federated identity silently auto-link to an existing account; can an IdP assign privileged roles through attributes with no approved trust policy behind that.

**Session creation.** Confirm it happens only after every required check — signature, issuer, audience/endpoint, time, correlation/replay, identity mapping, account status — and investigate whether any exception or alternate branch can create a session before all of those are satisfied.

### XML Signature Wrapping — focused investigation

This gets its own section because it's the SAML-specific vulnerability class with no equivalent anywhere else in this guide set, and it's subtle enough to deserve a dedicated walkthrough.

**The failure mechanism**: the application validates a signature covering XML element A, the document also contains a second identity-bearing element B, and the application extracts the subject from B instead of the validated A — creating a session for an identity the signature never actually protected. The core problem is not that the XML contains multiple elements (that can be entirely normal); it's that the application trusts identity data that isn't covered by the validation it just performed.

Black-box evidence: a structurally ambiguous test message, in an authorized environment, that's accepted rather than rejected. Record the message structure, the signature-validation outcome if observable, which identity the application actually used, and the resulting account and privileges. Gray-box evidence: logs or library output showing which element was validated versus which subject was extracted. White-box evidence: trace parsing → signature-reference resolution → verification → assertion selection → subject extraction → account mapping, and demonstrate the mismatch directly. A strange XML structure alone is not proof — the finding requires showing the validated element and the consumed element are different.

## G. Differential testing

| Dimension | Baseline | Comparison | Question |
|---|---|---|---|
| Signature | Trusted valid | Invalid/untrusted | Is the signature policy enforced? |
| Issuer | Approved IdP | Untrusted issuer | Is trust anchored correctly? |
| Audience | Correct SP | Different audience | Is the assertion meant for this SP? |
| Recipient | Correct ACS | Mismatched | Are endpoint restrictions enforced? |
| Time | Valid | Expired/not-yet-valid | Are validity conditions enforced? |
| Correlation | Expected reference | Missing/mismatched | Is the transaction bound correctly? |
| Replay | First submission | Repeated submission | Is replay handled per policy? |
| Identity | Existing test user | Different subject | Is the correct account selected? |
| XML structure | Normal signed message | Controlled malformed structure | Does the SP consume the signed data? |

Change one property at a time — altering multiple fields together makes it impossible to tell which one actually caused a rejection.

## H. Exploitation and validation

Minimal proof for each finding class is listed in the source material's own "Common Findings" catalog — in every case, the standard is: confirmed policy, a controlled message that violates it, demonstrated acceptance, and a verified resulting identity/session. Never test any of this against real users or production identities — controlled test IdP and accounts only.

## I. False-positive analysis

Treating every unsigned *response* as vulnerable without checking whether the *assertion* signature is correctly required instead. Treating every missing `InResponseTo` as a flaw without confirming whether IdP-initiated SSO is actually the supported flow. Treating a valid signature as proof of a valid assertion — it replaces none of the issuer/audience/time/recipient/correlation/mapping checks. Treating an expired assertion as exploitable without checking the documented clock-skew tolerance first. Treating every repeated response as replay without confirming the resulting authentication state actually changed. Treating email-based mapping as automatically unsafe without checking whether the trust model actually supports it. Treating every XML parser as XXE-vulnerable without confirming untrusted input can reach a dangerous configuration. Treating a suspicious XML structure as proof of signature wrapping without demonstrating the validated-vs-consumed element mismatch directly.

## J. Root cause

The recurring pattern across nearly every finding class here: a check that exists somewhere in the codebase (issuer comparison, audience check, correlation lookup) but is either not actually invoked on the real execution path, or has its result not respected (logged as a warning instead of terminating processing). For XSW specifically: identity extraction operating on raw parsed XML independently of what the signature verification step actually validated.

## K. Remediation

- Use a maintained SAML library's secure-processing APIs; never hand-roll XML signature validation.
- Anchor trust to controlled configuration (issuer, signing keys) — never to values the incoming message asserts about itself.
- Enforce signature, audience, recipient, and time-condition validation on the specific element the application actually consumes for identity, not a sibling element.
- Correlate SP-initiated requests against stored server-side state, not just syntactic presence of a reference value.
- Implement replay tracking that matches the actual deployment architecture (shared across instances where that's required).
- Define identity-mapping rules explicitly — which claims are authoritative, from which IdPs, under what verification assumptions — and never let unauthorized attributes assign privileged roles.
- Create sessions only after every required check passes; no exception path should fall through to success.

## L. Regression testing

Re-run the full differential matrix (signature, issuer, audience, recipient, time, correlation, replay, XML structure) after any SAML library upgrade, since validation defaults and strictness can change silently between versions. Re-test identity mapping specifically after any change to attribute-release configuration on the IdP side. Re-verify XML Signature Wrapping resistance after any change to the XML parsing or signature-reference-resolution code — this is exactly the kind of defect that can be silently reintroduced by a seemingly unrelated parser or library swap.

## M. Finding documentation

Title, summary (what was accepted and why that violates policy), affected component, preconditions, expected vs. actual behavior, sanitized reproduction with controlled identities, evidence (protocol fields, response behavior, resulting account, corroborating config/code), impact, root cause, remediation, severity and confidence with unresolved assumptions noted explicitly. Redact assertions, session cookies, and private identity data from every piece of shared evidence.

## Completion criteria

- [ ] Legitimate SSO baseline captured with every relevant protocol field recorded
- [ ] Signature, issuer, and audience tested independently against otherwise-valid messages
- [ ] Destination/recipient and time-condition validation tested
- [ ] Request correlation tested for the actual supported flow (SP-initiated vs. IdP-initiated)
- [ ] Replay tested with a genuine resulting-state check, not just a second HTTP response
- [ ] Identity mapping tested across multiple NameID formats and at least two controlled subjects
- [ ] XML Signature Wrapping specifically investigated: validated element vs. consumed element traced
- [ ] Every finding traced to the specific missing or unenforced check, not reported as a generic "SAML bypass"
