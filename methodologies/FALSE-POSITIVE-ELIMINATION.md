# False-Positive Elimination

Every attack guide has its own "I. False-positive analysis" section with mistakes specific to that domain. This document is the cross-cutting index — the same handful of *meta-patterns* recur independently across all 17 guides, and recognizing the pattern itself is more useful than memorizing 17 separate lists. If a finding you're about to report fits one of these shapes, stop and verify before writing it up.

## 1. The response-code fallacy

A `200 OK` is not proof of unauthorized disclosure — it can wrap an empty object, public metadata, a cached page, or a legitimately shared resource. A `403` is not proof of protection — the side effect may already have fired before the denial was returned. A generic "success" message is not proof the operation actually executed, and a generic error is not proof it didn't. **The fix is always the same: verify the resulting state through an independent, authorized read, never trust the response that describes it.** Shows up in essentially every guide; most explicit in 01, 03, 09, 11, 16.

## 2. The presence-vs-enforcement fallacy

A control *existing* is not the same as a control *being invoked on the path you're testing* or *having its result respected*. Authentication middleware existing doesn't prove authorization happens. A config flag named `reauthentication_required` existing doesn't prove it's enforced. A validation function existing doesn't prove the surrounding code doesn't just log its failure and continue anyway. Central to 10, 14, 16; the decode-vs-verify confusion in 04, 07, and 08 is a specific, especially common instance of this same fallacy.

## 3. The single-sample fallacy

One slow request does not establish a timing side channel. One unusual response does not establish enumeration. One suspicious-looking code pattern does not establish a vulnerability. Every one of these requires repetition, a genuine distribution comparison, or independent corroboration before it becomes evidence rather than a lead. Central to 17 (timing, enumeration) and 16 (code-smell-as-proof).

## 4. The difference-without-violation fallacy

Different cookie names, different token formats, different status codes, different error messages, different environments — none of these are automatically a finding. The question is never "is there a difference," it's "does this specific difference violate the intended policy." A `SameSite=None` cookie, a self-signed certificate, an old signing key still present — all differences, none automatically vulnerabilities. Central to 03, 04, 08, 12, 15.

## 5. The absence-proves-nothing fallacy (in both directions)

Missing CAPTCHA doesn't prove a credential-stuffing vulnerability. A missing log entry doesn't prove an attack went undetected — the event might have been generated and lost downstream, not skipped at the source. Conversely, a control's mere *presence* (rate limiting, a lockout mechanism, MFA) doesn't prove it's actually effective at the threshold that matters. Central to 02, 14; the "missing log ≠ missing control" direction is the core diagnostic move in 14 specifically.

## 6. The client-side display fallacy

The browser's back button showing a page, a cached response rendering, or a UI element being hidden — none of these tell you anything about current server-side state. Only a *fresh request* that requires live server authorization is evidence. Central to 03, 13, 17.

## 7. The legitimate-design fallacy

Administrators seeing more data, IdP-initiated SSO skipping request correlation, a user belonging to two tenants, a short documented propagation window after a permission change, intentional resource sharing — all of these can be entirely correct behavior that merely *looks* like a violation until you establish the actual intended policy first. This is why every differential-testing workflow in this course starts with "define the expected policy" as step one, not an afterthought. Central to 08, 09, 11, 16.

## 8. The terminology-and-format fallacy

A JWT payload being readable doesn't mean it's insecure — encoding is not encryption, and confidentiality is a different property from integrity. SHA-256 appearing somewhere in the code doesn't mean password storage is broken, or secure — it depends entirely on what it's used for. A self-signed certificate isn't automatically exploitable. A `kid` header value is usually meant to be public, not a leaked secret. Central to 04 and 15.

## 9. The narrow-scope fallacy

Testing only the read path when write/delete might enforce differently on the same resource. Testing only the primary route when a legacy, mobile, or alternate-API-version route might not. Testing only one tenant tier, one account state, or one interface. The fix is the differential-testing discipline itself (see `DIFFERENTIAL-TESTING.md`) — systematically vary the dimension you haven't checked yet rather than assuming consistency. Central to 11, 12, 16.

## The one-line version

**Every false positive in this course comes from mistaking a signal for a conclusion.** A response, a difference, a missing field, a cached page, a code pattern — all of these are leads. The conclusion requires the actual resulting state, repeated or corroborated evidence, and an explicitly stated policy the observed behavior actually violates. If you can't state what policy was violated and point to the evidence that it was, you have a lead, not a finding.
