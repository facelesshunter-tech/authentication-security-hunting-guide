# White-Box Testing Methodology

The generalized discipline behind every "F. White-box testing" section across all 17 attack guides.

## What white-box testing actually is

Tracing the actual implementation — source code, middleware, configuration-as-code — to establish the *root cause* behind behavior you've already observed or suspect. White-box testing is where a lead becomes an explanation. It is not a substitute for black-box evidence of actual impact; a vulnerable-looking code pattern without a demonstrated consequence is a review target, not a finding.

## The universal trace pattern

Nearly every guide in this course traces some version of the same pipeline:

`request → routing → authentication middleware → identity/principal construction → resource or policy lookup → authorization decision → business logic → data access → response serialization → side effects`

The single most repeated instruction across all 17 guides: **confirm the check exists *on the actual execution path you're testing*, not just somewhere in the codebase.** A validation function, an authorization helper, a policy check — all of these can exist, be entirely correct, and still protect nothing, because the specific route or code path under test never actually calls them.

## The discipline that separates senior review from a keyword search

1. **Trace reachability, not just presence.** A missing check in one controller might be covered by shared middleware you haven't looked at yet. A dangerous-looking function in an unused file protects nothing from an attacker who can't reach it. Confirm the path is live before reporting.
2. **Distinguish "the check exists" from "the check's result is respected."** A recurring pattern across this entire course: code calls a validation function, the function correctly returns an error, and the surrounding code logs it as a warning and continues anyway instead of terminating the operation. The check existing is not the same as the check mattering.
3. **Compare the correct implementation against its siblings.** Nearly every "common defect" pattern in this course is some version of: the primary path does it right, and an alternate path (a legacy route, a background job, an admin override, a different API version) does it differently because the logic was duplicated instead of shared. Always ask what else reaches the same resource or operation.
4. **Recognize decode-vs-verify as a recurring root cause.** This single confusion — treating successful parsing as proof of authenticity — shows up independently in JWT handling, OAuth ID tokens, and SAML assertions throughout this course. It's worth searching for by name.

## Common vulnerable-pattern families to search for directly

- A resource fetched by ID first, with ownership/tenant checks applied after (or never), instead of scoped directly into the query.
- An entire request body bound to a model update with no field allowlist (mass assignment).
- A fallback or exception-handling branch that proceeds on cryptographic or validation failure instead of terminating.
- A centralized "is logged in" check used where an "is allowed to do *this specific thing*" check was actually needed.
- A freshness or state check that reads the wrong timestamp (session creation instead of last authentication, for example).

## What white-box evidence does not give you

Code correctness in isolation is not impact. The strongest findings in this course always close the loop back to black-box or gray-box evidence that the vulnerable path is actually reachable and actually produces the unauthorized outcome — never report a code smell as a confirmed vulnerability without that closing step.

## Where this applies

Every attack guide's white-box section follows this discipline, with the specific vulnerable patterns scoped to that guide's domain.
