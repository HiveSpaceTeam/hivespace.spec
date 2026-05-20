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
3. ApiGateway routes versioned identity traffic to IdentityService and profile/store traffic to UserService; `/identity/**` is not present.
4. Message bus handles account-created and seller-role assignment messages with observable failures.

## Validation Flow

1. Start local infrastructure from `../hivespace.config/docker`.
2. Apply the development reset or migration path.
3. Create or seed buyer, seller, admin, and system admin accounts through IdentityService-owned flows.
4. Confirm UserService profile records are created from identity account-created events.
5. Sign in as a buyer and update profile, settings, and addresses through UserService-owned flows.
6. Register a store as the buyer.
7. Confirm UserService creates the store and publishes seller-role assignment request.
8. Confirm IdentityService grants seller role/claims.
9. Refresh authorization state and confirm seller access is available.
10. Confirm admin identity account status changes affect sign-in and authorization, and UserService has no separate profile status controlling access.

## Expected Results

- No service reads another service database.
- Profile creation and seller-role assignment are retryable and idempotent.
- Email verification is owned by IdentityService.
- Store projection events remain UserService-owned unless implementation deliberately changes them.
- Route changes are reflected in gateway config, frontend services, and catalogs.
