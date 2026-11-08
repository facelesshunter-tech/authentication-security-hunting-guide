# Authentication Attack Surface

The consolidated inventory of what exists to test — entry points, authentication states, trust boundaries, and client-controlled values — pulled from across all 17 attack guides into one reference. This is the *catalog*; `attack-guides/01-ATTACK-SURFACE-MAPPING.md` is the methodology for actually mapping it against a real target. Use this to sanity-check your own target-specific inventory for gaps, not as a replacement for building one.

## Entry points

Not every application implements every one of these — mark each as present, absent, or unknown for your target rather than skipping silently.

| Entry point | Primary guide |
|---|---|
| Registration | 01, 09 |
| Login (web, API, mobile, admin, partner) | 01, 02 |
| Logout | 01, 03, 13 |
| Password change | 02, 06 |
| Forgot-password / reset request | 06 |
| Password reset completion | 06 |
| Email verification | 09 |
| Phone verification | 09 |
| MFA enrollment | 05 |
| MFA challenge / verification | 05 |
| MFA removal | 05 |
| MFA replacement | 05 |
| Backup-code generation and use | 05 |
| Passkey / WebAuthn registration | 05 |
| Passkey / WebAuthn authentication | 05 |
| OAuth/OIDC authorization and callback | 07 |
| SAML SSO and ACS endpoint | 08 |
| Account linking / unlinking | 07, 08 |
| Session management (active sessions, device list) | 03 |
| Remember-me / trusted-device functionality | 03, 05 |
| Reauthentication / step-up challenges | 10 |
| Account deactivation | 09 |
| Account deletion | 09 |
| Account restoration | 09 |
| Administrative account management | 01, 16 |
| Tenant/organization administration | 11 |
| Invitation and membership workflows | 11 |
| Legacy login and API routes | 12 |
| Federated logout (RP-Initiated, back-channel, SAML SLO) | 13 |
| API key / service-credential issuance | 04, 12 |

## Authentication states

A target's actual model may use different names, but the underlying state machine is almost always some version of this list. Document which states your target actually has — don't assume all eleven exist.

Anonymous → Registered-but-unverified → Password-accepted → MFA-pending → Fully authenticated → Step-up-required → Restricted → Disabled → Deactivated → Deleted → Privileged/administrative.

For every state, the question that matters is: what is the server-side evidence that distinguishes it from every adjacent state, and which specific check consults that evidence before a protected operation proceeds.

## Trust boundary components

Every one of these is a place a security decision gets made — or should be, and sometimes isn't:

Browser/mobile client, frontend application, backend API, identity provider, authentication service, session store, database, email/SMS verification provider, background workers, file/object storage, external payment or business systems, reverse proxies and API gateways.

## Client-controlled values that must never be trusted as proof of anything

User IDs, tenant/organization IDs, role names, account-status flags, authentication-state flags, resource-ownership identifiers, redirect parameters, workflow-state fields. Every one of these appearing in a request is normal; the question is always whether the server independently verifies it against trusted state rather than accepting the client's claim. See `attack-guides/16-AUTHENTICATION-ADJACENT-AUTHORIZATION.md` for the authorization-specific depth, and `attack-guides/11-MULTI-TENANT-IDENTITY.md` for the tenant-specific depth.

## Where the attack surface is typically under-mapped

The pattern across this entire course: a detail/primary endpoint gets tested carefully, and list, search, export, background-job, and alternate-API-version paths reaching the *same* underlying data get missed entirely. If you've mapped the primary route for every entry point above and stopped there, you've mapped roughly half the real attack surface — see `attack-guides/11-MULTI-TENANT-IDENTITY.md` section D and `attack-guides/16-AUTHENTICATION-ADJACENT-AUTHORIZATION.md` section D for the specific secondary-path categories worth checking deliberately.
