# Quickstart: Split Identity Service

## Goal

Validate the planned development migration from one UserService identity/profile boundary into separate IdentityService and UserService ownership.

## Prerequisites

- Active feature artifacts:
  - `specs/0002-split-identity-service/spec.md`
  - `specs/0002-split-identity-service/plan.md`
  - `specs/0002-split-identity-service/contracts/`
- Backend and frontend source repo instructions read before implementation:
  - `../hivespace.microservice/AGENTS.md`
  - `../hivespace.microservice/CLAUDE.md`
  - `../hivespace.web/AGENTS.md`
  - `../hivespace.web/CLAUDE.md`

## Development Migration Shape

This feature allows breaking development migration. The implementation path may reset local development databases if that is simpler than preserving existing dev data.

Minimum repeatable outcome:

1. IdentityService database contains identity users, credentials, roles, claims, account status, email verification, OIDC clients, and operational grants.
2. UserService database contains profiles, settings, addresses, and stores keyed by the same public user IDs.
3. IdentityService runs locally on `http://localhost:5001`; UserService runs locally on `http://localhost:5007`.
4. IdentityService SeedData creates identity-owned roles, buyer/seller/admin/system admin accounts, account status, lockout defaults, claims, seller authorization references, and OIDC configuration where applicable.
5. UserService SeedData creates matching profiles, settings, addresses, stores, and store owner references only, using the same public user IDs as IdentityService seed users.
6. Frontend clients use the IdentityService authority URL directly for IdentityServer public endpoints, including `/.well-known/**`, `/connect/**`, and `/Account/**`; ApiGateway does not forward those endpoints, and `/identity/**` and `/api/v1/identity/**` are not present.
7. Message bus handles account-created messages and store-created seller role assignment with observable failures.

## Validation Flow

1. Start local infrastructure from `../hivespace.config/docker`.
2. Apply the development reset or migration path.
3. Confirm IdentityService starts on `http://localhost:5001` and UserService starts on `http://localhost:5007`.
4. Create or seed buyer, seller, admin, and system admin accounts through IdentityService-owned flows.
5. Confirm IdentityService SeedData does not write profile, address, settings, or store records.
6. Confirm UserService profile records are created from identity account-created events or UserService-owned SeedData during reset.
7. Confirm UserService SeedData does not write credentials, roles, claims, account status, lockout, email verification, OIDC clients, or grants.
8. Sign in as a buyer and update profile, settings, and addresses through UserService-owned flows.
9. Register a store as the buyer.
10. Confirm UserService creates the store and publishes `StoreCreatedIntegrationEvent`.
11. Confirm IdentityService consumes `StoreCreatedIntegrationEvent`, grants seller role/claims, and records the store reference on `ApplicationUser`.
12. Refresh authorization state and confirm seller access is available.
13. Confirm admin identity account status changes affect sign-in and authorization, and UserService has no separate profile status controlling access.

## Expected Results

- No service reads another service database.
- Profile creation and seller role assignment from `StoreCreatedIntegrationEvent` are retryable and idempotent.
- Email verification is owned by IdentityService.
- Store projection events remain UserService-owned unless implementation deliberately changes them.
- Route changes are reflected in gateway config, frontend services, and catalogs.
- Runtime config has no stale IdentityService `5007` or UserService `5001` references after implementation, except documentation that explicitly describes the previous state.
