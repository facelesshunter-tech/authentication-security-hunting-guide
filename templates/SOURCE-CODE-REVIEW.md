# Source-Code Review Template

For recording a white-box tracing session (see `methodologies/WHITE-BOX-TESTING.md`). Fill this out *while* you trace, not after — the sequence of what you checked and in what order is itself part of the evidence that the conclusion is sound rather than a keyword search that got lucky.

```text
Operation under review:   [e.g. "password reset token validation"]
Attack guide reference:   [which guide's white-box section this follows]
Files/modules involved:

--- Trace path ---
List each stage of the actual execution path, in order, with the
file/function at each stage. Use the universal pattern from
methodologies/WHITE-BOX-TESTING.md as a starting skeleton and adapt:

1. Entry point (route/controller):
2. Authentication middleware:
3. Identity/principal construction:
4. Resource or policy lookup:
5. Authorization decision:
6. Business logic:
7. Data access:
8. Response serialization:
9. Side effects (if any):

--- At each stage, record ---
Stage:
What this stage is supposed to guarantee:
What the code actually does:
Is the relevant check present?            [ ] Yes   [ ] No   [ ] Present but not invoked here
Is the check's result actually respected   [ ] Yes   [ ] No   [ ] Logged but ignored
  (not just logged and continued)?

--- Reachability confirmation ---
Is this exact code path reachable from an authorized, unauthenticated
or low-privileged request in the live/test environment?
How was reachability confirmed? [not assumed from reading the code alone —
  tie back to a black-box test case via AUTHENTICATION-TEST-CASE.md]

--- Comparison against equivalent paths ---
Does an alternate route, API version, legacy endpoint, background job,
or different HTTP method reach the same resource or operation?
[ ] Yes — compared and recorded separately    [ ] No equivalent path exists   [ ] Not checked yet

--- Conclusion ---
Root cause identified:
Is this a design flaw, implementation flaw, configuration flaw, or
trust-boundary failure?
Confidence this produces a real, demonstrated consequence (not just a
suspicious pattern):
```

## The discipline this template enforces

The most common white-box mistake across this entire course is stopping at "I found a function that doesn't call the check" and reporting that as a finding. This template forces three additional steps before that's allowed: confirming the check isn't enforced somewhere else on the same path, confirming the path is actually reachable, and confirming the result against black-box evidence. A code pattern that fails any of these three is a review note for the codebase owner, not yet a vulnerability report.
