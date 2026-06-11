# IdentityService Domain Model

## Purpose

IdentityService owns identity state for HiveSpace: credentials, account status, roles, claims, lockout, email verification, IdentityServer configuration, and token/grant ownership.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.IdentityService
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `ApplicationUser` | ASP.NET Identity user | Identity-owned account record with credentials, email verification, status, lockout fields, role, claims, timestamps, and optional store reference |
| `IdentityRole` / role records | ASP.NET Identity role | Authorization roles such as buyer, seller, admin, and system admin |
| External login records | ASP.NET Identity external login | Durable Google provider link owned by IdentityService |
| IdentityServer clients/grants | IdentityServer data | OIDC client configuration, persisted grants, refresh tokens, and protocol state |

## Business Rules

- IdentityService is the source of truth for sign-in, token issuance, authorization roles, claims, account status, temporary lockout, and email verification.
- UserService may keep email or username as non-authoritative profile/display copies, but uniqueness and verification state belong to IdentityService.
- Suspended, inactive, locked-out, or invalid accounts cannot authenticate or receive valid authorization state.
- Store registration does not directly write identity state. IdentityService grants seller access only after consuming the UserService-owned `StoreCreatedIntegrationEvent`.
- Role propagation is idempotent by store owner and store ID so message retries do not duplicate role/claim assignments.
- Google sign-in is allowed only for buyer and seller app contexts. Admin accounts cannot be created or signed in through Google.
- New Google-authenticated users start as normal user accounts. Seller access still requires the existing store onboarding flow.
- A matching existing email/password account may link Google only after Google provides the same verified email, the user explicitly consents, and the existing password is confirmed.

## Lifecycle

| Lifecycle | States / transitions |
|---|---|
| Account status | Active/suspended/inactive-style account availability states owned by IdentityService |
| Lockout | Failed sign-in attempts update ASP.NET Identity lockout fields until lockout expires or is cleared |
| Email verification | Verification request publishes a notification event; successful verification updates identity-owned state |
| Seller transition | `StoreCreatedIntegrationEvent` consumption grants seller role/claims and records the store reference on the identity account |
| Account readiness | When an account becomes usable, IdentityService publishes `IdentityUserReadyIntegrationEvent` for UserService profile creation |
| Google link | Pending external login state becomes a durable external login record only after consent and password confirmation |

## Cross-Service Facts

- IdentityService publishes identity-owned account and email verification events.
- IdentityService consumes store-created facts from UserService for seller role propagation.
- IdentityService does not read or write UserService profile, settings, address, or store tables.
