# Application-Specific Assessment — Not Applicable

The original master prompt for this deliverable asked for an application-specific assessment matrix: every implemented authentication feature mapped against real source code, with black-box/gray-box/white-box test results and a feature-by-feature implementation status.

## Why this section is empty by design, not by oversight

Phase 0 discovery for this project explicitly searched for a live, self-built authentication application to serve as this case study — the local workspace, this conversation's full history, and every repository on the user's GitHub account. None exists. This was confirmed directly, not assumed: see `SOURCE-TRACEABILITY.md` for the specific checks performed and their results.

Presented with that finding, the user made an explicit decision: build this entire course black-box-first, rather than first building a deliberately vulnerable application purely to have something to white-box against. That decision is why every attack guide in this course treats black-box testing as the primary mode, with gray-box and white-box sections written as general implementation-pattern guidance (what the vulnerable code usually looks like, not what a specific named application's code actually contains).

## What this means concretely

- No `FEATURE-TEST-MATRIX.md`, `ENDPOINT-INVENTORY.md`, or `SOURCE-CODE-TRACE.md` exists for a specific application, because there is no specific application.
- Every white-box section across the 17 attack guides uses illustrative code patterns (clearly marked as such) to show what a vulnerable implementation typically looks like, not a trace through a real, named codebase.
- `templates/SOURCE-CODE-REVIEW.md` remains fully usable — it's designed to be filled in against whatever real target a reader eventually points this course at, rather than being pre-filled against one that doesn't exist here.

## If a real application becomes available later

Point `templates/SOURCE-CODE-REVIEW.md` and the white-box sections of the relevant attack guides at it directly. The methodology doesn't change — only the subject does. This document should be replaced with an actual `FEATURE-TEST-MATRIX.md` at that point, not kept alongside one.
