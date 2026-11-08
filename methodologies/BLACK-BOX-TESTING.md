# Black-Box Testing Methodology

The generalized discipline behind every "D. Black-box testing" section across all 17 attack guides. Read this once, apply it everywhere — the guides assume it rather than re-deriving it each time.

## What black-box testing actually is

Testing the application through its normal, authorized interfaces — browser, API, mobile client — using only what any legitimate authenticated (or unauthenticated) caller could observe. No source code, no internal configuration, no database access. The entire discipline is about extracting maximum signal from externally observable behavior: status codes, response bodies, headers, cookies, redirects, timing, and — critically — *actual resulting state*, not just the response that claims to describe it.

## The non-negotiable sequence

1. **Establish a baseline before testing anything else.** Prove that a controlled, legitimate action succeeds in the expected way before you try to break it. A denial against an attack scenario means nothing if you never confirmed the corresponding legitimate scenario actually works — you might just be looking at a broken feature, not an enforced boundary.
2. **Use controlled identities with known properties.** Every guide in this course assumes multiple test accounts with documented state: verified/unverified, MFA-on/off, different tenants, different roles. Build this inventory before testing, not during.
3. **Change one variable at a time.** This is differential testing's core discipline (see `DIFFERENTIAL-TESTING.md`) but it governs black-box work specifically: if you change the account *and* the resource *and* the method in one test, a different result tells you nothing about which variable caused it.
4. **Capture the complete exchange, not just the interesting field.** Method, path, headers, cookies, body, status, response body, redirect destination, and — this is the one people skip — the *resulting state*, verified independently of the response.

## The single most repeated correction across all 17 guides

**A response is not proof of what happened server-side.** A `200 OK` can wrap an empty object, a cached page, or a queued-but-not-executed action. A `403` can arrive after the side effect already fired. A generic success message can mask a silent no-op. Every guide in this course treats the response as a *claim*, not evidence — the actual proof is the resource's state, re-read through an independent, authorized channel.

## What you can and cannot establish black-box

**You can establish**: observable behavioral differences, response structure and content, timing distributions (with enough samples), redirect chains, cookie/header behavior, whether a protected operation actually completed.

**You generally cannot establish**: the hashing algorithm behind a password, whether randomness is cryptographically secure, whether a check exists somewhere else in the code that a single test didn't reach, the actual cause of an observed difference. These require gray-box or white-box evidence — flag them as such rather than guessing.

## Evidence discipline

Redact session tokens, passwords, and reset/verification tokens from anything you keep or share, while preserving enough structure to prove the point (a fingerprint, a length, a redacted-but-present marker). Never test destructive operations against anything but disposable, controlled resources. Never run high-volume attacks (brute force, lockout campaigns) against production without explicit, scoped authorization.

## Where this applies

Every attack guide's black-box section follows this discipline. See `TESTING-QUICK-REFERENCE.md` for the condensed, per-phase checklist version you'd actually have open during a live engagement.
