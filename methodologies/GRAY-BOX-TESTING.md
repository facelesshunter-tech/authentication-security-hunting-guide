# Gray-Box Testing Methodology

The generalized discipline behind every "E. Gray-box testing" section across all 17 attack guides.

## What gray-box testing actually is

Black-box testing plus limited, authorized internal visibility: API documentation, architecture diagrams, configuration, logs, database schemas, read-only admin views, or partial source code. Gray-box testing doesn't replace black-box testing — it *explains* it. The question gray-box evidence answers is not "what does the application do" (you already know that from black-box testing) but "why does it do that, and is the explanation consistent with the intended policy."

## The core gray-box move

Compare **documented or configured intent** against **observed black-box behavior**, and treat any discrepancy as a lead requiring further investigation — not an automatic finding. A documented restriction the server fails to enforce is strong evidence. An undocumented route simply existing is not, by itself, evidence of anything.

## The most valuable gray-box sources, in practice

- **Logs and audit trails**: correlate your controlled test request with what actually got recorded — the actor, the decision, the resulting state. This is often the fastest way to distinguish "the control is missing" from "the control exists but didn't fire on this path."
- **Configuration vs. deployed reality**: a setting in a config file is not proof of production behavior. Environment-specific overrides (dev vs. staging vs. production) are the single most common place intent and reality diverge.
- **API schemas and documentation**: a sensitive field appearing in a schema is a lead, not proof it's actually serialized to an unauthorized caller — confirm the real behavior.
- **Database records** (where authorized): the actual stored state is the strongest tie-breaker when a black-box response is ambiguous about whether something really changed.

## The recurring gray-box diagnostic pattern

Across every guide in this course, the same two-branch diagnosis shows up: if a source event or record exists but a downstream effect (an alert, a central log entry, an enforcement decision) doesn't, the gap is in the *pipeline* between generation and consumption. If the source itself never generated the event or record, the gap is at the *origin*. Gray-box evidence is what lets you tell these apart — black-box testing alone usually can't.

## What gray-box evidence does not give you

It explains *why* observed behavior happens, but it still doesn't prove *reachability* or *exploitability* on its own. A missing check visible in a config review might be covered by middleware you haven't inspected yet; a documented policy you haven't seen enforced black-box might still be enforced somewhere you haven't tested. Gray-box findings corroborate black-box findings and point toward white-box confirmation — they rarely stand alone as a complete finding.

## Where this applies

Every attack guide's gray-box section follows this discipline, scoped to that guide's specific evidence sources (SAML metadata, OAuth client configuration, tenant membership records, detection-rule configuration, and so on).
