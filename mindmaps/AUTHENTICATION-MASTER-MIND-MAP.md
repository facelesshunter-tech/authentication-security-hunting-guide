# Authentication Security Master Mind Map

This is the root map for the entire hunting guide. It shows every authentication and authentication-adjacent domain in one place, and how they relate to each other. Use it to orient yourself at the start of an assessment, then drop into the relevant grouped sub-map (`mindmaps/`) or attack guide (`attack-guides/`) for depth.

Two diagrams follow. The first is the taxonomy: every domain and its sub-topics, structured as a tree. The second is the relationship map: how a weakness in one domain actually creates or compounds a weakness in another, because real findings almost never stay inside one branch.

## 1. Taxonomy

```mermaid
mindmap
  root((Authentication Security))
    Attack Surface Mapping
      Registration
      Login
      Logout
      Password change
      Password reset
      Email and phone verification
      MFA enrollment and verification
      Recovery codes
      Session creation and renewal
      Remember-me and trusted devices
      OAuth and OIDC login
      SAML SSO
      Reauthentication
      Account deletion and suspension
      Administrative authentication
      API and mobile authentication
      Legacy authentication
    Password and Credential Security
      Storage
        Hashing algorithms
        Salt and pepper
        Work factors
      Verification logic
      Policies
      Credential stuffing
      Password spraying
      Brute-force resistance
      Username enumeration
      Timing discrepancies
      Password reuse
      Reset weaknesses
    Session Security
      Session generation
      Identifiers
      Fixation
      Hijacking
      Expiration
        Idle timeout
        Absolute timeout
      Rotation
      Concurrent sessions
      Invalidation
      Logout behavior
      Cookie attributes
        Secure
        HttpOnly
        SameSite
        Domain and Path
      Device-bound sessions
    Token Security
      JWT fundamentals
        JWS and JWE
        Claims iss sub aud exp nbf iat jti
        alg and kid
      Algorithm confusion
      Signature verification
      Key management and rotation
      Access tokens
      Refresh tokens
      Token replay
      Token leakage
      Token substitution
      Improper claim validation
    Multi-Factor Authentication
      TOTP and HOTP
      SMS and email OTP
      Push authentication
      WebAuthn and FIDO2
      Passkeys
      Recovery codes
      Enrollment
      Removal
      MFA bypass
      MFA fatigue
      Rate limiting
      Step-up authentication
    OAuth2 and OIDC
      Authorization code flow
      PKCE
      Redirect URI validation
      State and nonce
      Scope and audience
      ID tokens
      JWKS and discovery
      Client secrets
      Account linking
      Login CSRF
      Mix-up attacks
      Code interception and injection
    SAML Security
      Assertions
      Signature validation
      Audience Recipient Destination
      NotBefore NotOnOrAfter
      Replay
      XML Signature Wrapping
      IdP and SP trust
      SAML logout
    Account Lifecycle
      Registration and verification
      Account linking
      Recovery
      Suspension and deletion
      Reactivation
      Email and phone changes
      Duplicate identities
      Account takeover paths
    Reauthentication and Step-Up
      Sensitive actions
      Password confirmation
      MFA confirmation
      Session age and freshness
      Privilege changes
      Payment confirmation
    Multi-Tenant Identity
      Tenant membership
      Tenant switching
      Tenant-scoped sessions
      Tenant-specific identity providers
      Cross-tenant authentication
      IdP configuration isolation
    Legacy Authentication
      Basic and digest auth
      Legacy API keys
      Old token formats
      Compatibility endpoints
      Deprecated protocols
      Weak migration mechanisms
    Federated Logout
      Local logout
      Global logout
      Front-channel and back-channel logout
      Session and token persistence
      Revocation synchronization
    Logging and Monitoring
      Login and MFA events
      Password reset events
      Session events
      Token events
      Suspicious activity alerting
      Audit trails
      Sensitive data in logs
    Cryptographic Failures
      Weak randomness
      Predictable tokens
      Weak algorithms
      Incorrect signature verification
      Key management
      Hardcoded secrets
    Authentication-Adjacent Authorization
      IDOR and BOLA
      BFLA
      Horizontal escalation
      Vertical escalation
      Mass assignment
      Property-level authorization
      Workflow authorization
      Stale authorization
    Information Leakage
      Account enumeration
      Timing differences
      Reset and OTP leakage
      Token exposure
      Debug and stack trace output
      Browser storage leakage
```

## 2. Relationship and attack-path map

A weakness in one domain is rarely the whole story. This diagram shows the relationships that matter most in practice — where one finding is the precondition for another, or where two weak controls compound into something reportable.

```mermaid
flowchart TD
    ENUM[Account enumeration] --> CREDATK[Credential stuffing / password spraying]
    ENUM --> RESETLEAK[Targeted password-reset abuse]
    CREDATK --> WEAKSTORE[Weak password storage confirms hit rate]
    RESETLEAK --> TOKENWEAK[Predictable or reusable reset token]
    TOKENWEAK --> ATO1[Account takeover]
    WEAKSTORE --> ATO1

    MFADOWNGRADE[Inconsistent MFA enforcement across routes] --> LEGACY[Legacy or alternate auth route]
    LEGACY --> MFADOWNGRADE
    MFADOWNGRADE --> ATO2[Account takeover without second factor]

    JWTALG[JWT algorithm confusion or weak secret] --> SESSION[Forged session / access token]
    SESSION --> AUTHZ1[Authorization bypass]
    OIDCNONCE[Missing or reused OIDC state/nonce] --> CSRFLOGIN[Login CSRF]
    CSRFLOGIN --> ACCTLINK[Forced account linking]
    ACCTLINK --> ATO3[Account takeover via federation]

    SAMLSIG[SAML signature not fully validated] --> XSW[XML Signature Wrapping]
    XSW --> IDENTITYSPOOF[Spoofed assertion identity]
    IDENTITYSPOOF --> ATO4[Account takeover via SSO]

    STALEAUTHZ[Stale authorization after role/tenant change] --> TENANTCROSS[Cross-tenant data access]
    MASSASSIGN[Mass assignment on registration or profile update] --> PRIVESC[Vertical privilege escalation]
    IDOR[IDOR / BOLA] --> HORIZ[Horizontal privilege escalation]
    HORIZ --> CHAIN1[Chained with weak session revocation]
    CHAIN1 --> SCALE1[Account-wide data exposure]

    REAUTHMISSING[Missing step-up auth on sensitive action] --> SENSOP[Unauthorized sensitive operation]
    SENSOP --> PRIVESC
    SENSOP --> STALEAUTHZ

    LOGOUTFAIL[Logout does not revoke server-side session] --> PERSIST[Persistent unauthorized access]
    FEDLOGOUTFAIL[Federated logout not propagated] --> PERSIST

    INFOLEAK[Information leakage: errors, timing, debug output] --> ENUM
    INFOLEAK --> TOKENWEAK
    INFOLEAK --> JWTALG

    LOGGINGGAP[Missing or incomplete auth logging] -.weakens detection of.-> ATO1
    LOGGINGGAP -.weakens detection of.-> ATO2
    LOGGINGGAP -.weakens detection of.-> ATO3
```

## 3. Domain-to-source traceability

Every branch above is built from a reconciliation of three source types, not invented. This table is the map from taxonomy branch to where its depth actually comes from.

| Domain | Course phase(s) | Testing Approach phase | Checklist phase | Attack guide |
|---|---|---|---|---|
| Attack Surface Mapping | All phases, entry points | Phase 1 | Phase 1 | `01-ATTACK-SURFACE-MAPPING.md` |
| Password and Credential Security | Phase 1 | Phase 5 | Phase 5 | `02-PASSWORD-AND-CREDENTIAL-ATTACKS.md` |
| Session Security | Phase 6 | Phase 8 | Phase 8 (checklist, not yet read) | `03-SESSION-SECURITY.md` |
| Token Security (JWT) | Phase 16 (crypto) | Phase 12 | Phase 12 (checklist, not yet read) | `04-JWT-AND-TOKEN-SECURITY.md` |
| Multi-Factor Authentication | Phases 3-4 | Phase 7 | Phase 7 (checklist, not yet read) | `05-MFA-ATTACKS.md` |
| Password Reset and Recovery | Phase 2 | Phase 6 | Phase 6 (checklist, not yet read) | `06-PASSWORD-RESET-AND-RECOVERY.md` |
| OAuth2 and OIDC | Phase 7 | Phase 10 | Phase 10 (checklist, not yet read) | `07-OAUTH2-AND-OIDC.md` |
| SAML Security | Phase 10 | Phase 10 | Phase 10 (checklist, not yet read) | `08-SAML-SECURITY.md` |
| Account Lifecycle | Phase 8 | Phase 4 | Phase 4 (checklist, not yet read) | `09-ACCOUNT-LIFECYCLE.md` |
| Reauthentication and Step-Up | Phase 9 | Phase 9 | Phase 9 (checklist, not yet read) | `10-REAUTHENTICATION.md` |
| Multi-Tenant Identity | Phase 11 | Phase 14 (as part of authz) | Phase 14 (checklist, not yet read) | `11-MULTI-TENANT-IDENTITY.md` |
| Legacy Authentication | Phase 13 | Phase 11 | Phase 11 (checklist, not yet read) | `12-LEGACY-AUTHENTICATION.md` |
| Federated Logout | Phase 14 | Phase 10 | Phase 10 (checklist, not yet read) | `13-FEDERATED-LOGOUT.md` |
| Authentication Logging | Phase 15 | Phase 13 | Phase 13 (checklist, not yet read) | `14-AUTHENTICATION-LOGGING.md` |
| Cryptographic Failures | Phase 16 | Phase 12 | Phase 12 (checklist, not yet read) | `15-CRYPTOGRAPHIC-FAILURES.md` |
| Authentication-Adjacent Authorization | Phase 17 | Phase 14 | Phase 14 (checklist, not yet read) | `16-AUTHENTICATION-ADJACENT-AUTHORIZATION.md` |
| Information Leakage | Phase 12 | Phase 3 | Phase 3 (checklist, read in full) | `17-INFORMATION-LEAKAGE.md` |

Rows marked "checklist, not yet read" will be read in full at the point each attack guide is built, not before — the same approach used to build the original 17-phase course, and the approach agreed in Phase 0 discovery.

## 4. How to use this map on a real assessment

Start at the top of the taxonomy to orient yourself, but hunt using the relationship map. In practice, findings cluster around a small number of entry points: account enumeration, inconsistent MFA or route enforcement, token validation gaps, and stale authorization after a state change. Each of those sits at the top of a chain in section 2, not at the bottom — which is why attack-surface mapping and baseline establishment come before any single vulnerability class in the hunting workflow (see `MASTER-TESTING-METHODOLOGY.md`, not yet built).
