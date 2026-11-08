# Differential Testing Methodology

The generalized discipline behind every "G. Differential testing" section across all 17 attack guides — and arguably the single technique that does the most work in this entire course.

## What differential testing actually is

Comparing behavior across two or more conditions while holding every variable constant except one, to determine whether the resulting difference violates the application's intended security policy. It is the method that connects black-box observation, gray-box explanation, and white-box root cause into one defensible chain.

## The repeatable workflow, stated once

Every guide in this course uses some variation of the same sequence:

1. **Define the expected policy explicitly, before testing.** Write down what *should* happen. ("Only the resource owner or an explicitly authorized delegate can read this record.") This step is what separates a defensible finding from a post-hoc rationalization of whatever you happened to observe.
2. **Establish the baseline.** Run the legitimate case and confirm it behaves as expected.
3. **Change exactly one variable.** Identity, role, tenant, resource state, authentication freshness, HTTP method, API version, time — pick one.
4. **Compare actual outcomes, not just status codes.** Data disclosed, state changed, error behavior, audit records, resulting session.
5. **Explain whether the result follows or violates the stated policy.**
6. **Corroborate with gray-box or white-box evidence where available.**
7. **Retest after remediation** — confirming both that the unauthorized case is now denied *and* that the legitimate case still works. A fix that breaks real access isn't a fix.

## Why "one variable at a time" is not optional

Altering multiple fields simultaneously and observing a different result tells you that *something* changed, not *what* caused it. A rejection could come from the field you meant to test or an unrelated field you changed incidentally. Every guide in this course that includes a differential matrix organizes it as single-variable rows for exactly this reason — resist the temptation to combine tests for efficiency; it costs you the ability to draw a conclusion.

## The variable dimensions that recur across this entire course

Identity (same resource, different caller), resource/tenant (same caller, different target), role/privilege (same resource, different permission level), state (before vs. after a transition — suspension, revocation, password change, membership removal), time (fresh vs. stale authentication, valid vs. expired token), interface (web vs. API vs. mobile vs. GraphQL vs. legacy route for the same operation), and request-order (does testing B after A change how A behaves afterward — a signal for cache or context bleed).

## The false-positive this method specifically guards against

A difference existing is not the same as a difference *mattering*. Two responses can differ in wording, formatting, or an incidental header while enforcing the identical underlying policy — and two responses can look identical in their primary message while still leaking the same sensitive distinction through timing, a secondary header, or a follow-up page. Differential testing's job is to find the policy-relevant difference, not just any difference. See `FALSE-POSITIVE-ELIMINATION.md` for the catalog of ways this goes wrong.

## Where this applies

Every attack guide in this course has a dedicated differential-testing section with a matrix scoped to that guide's specific variables — this document is the method underneath all seventeen of them.
