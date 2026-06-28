# Verification Tasks

## Repo Instructions

### Verify

- [ ] V001 Verify `AGENTS.md` prerequisites
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.web/AGENTS.md`
  - Read both target-repo instruction files before implementation and confirm the task execution order follows their backend/frontend architecture rules.
  - Confirm there are no feature tasks that edit `../hivespace.config`.
  - Acceptance: implementation starts with current repo guidance and stays scoped to the backend/frontend repos plus spec-repo docs.

## Backend Verification

### Verify

- [ ] V002 [US1][US2][US3] Verify backend builds
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/`, `../hivespace.microservice/src/HiveSpace.IdentityService/`, `../hivespace.microservice/src/HiveSpace.UserService/`
  - Run the narrowest relevant backend verification after code changes: restore/build the shared messaging library, IdentityService, and UserService projects or the full solution if project-level builds are insufficient.
  - Confirm the new readiness event, register result contract, resend endpoint, verify endpoint, and UserService consumer all compile together.
  - Acceptance: backend build passes with only documented baseline warnings and no stale references to `IdentityUserCreatedIntegrationEvent`.

- [ ] V003 [US1][US2][US3] Verify buyer auth flow
  - File: `../hivespace.web/apps/buyer/`
  - Run the buyer verification command set required by repo guidance, using `pnpm type-check` and a buyer build if needed after the auth-flow changes.
  - Manually validate buyer sign-up -> verify-email -> verify-email-callback -> sign-in recovery routing, including pending-login and anonymous resend behavior.
  - Acceptance: buyer auth pages compile, route correctly, and implement the pending-verification flow without assuming registration creates a session.

- [ ] V004 [US1][US2][US3] Verify seller auth flow
  - File: `../hivespace.web/apps/seller/`
  - Run the seller verification command set required by repo guidance, using `pnpm type-check` and any additional build/lint checks while respecting the documented baseline failures in seller demo/auth areas.
  - Manually validate seller sign-up -> verify-email -> verify-email-callback -> sign-in -> register-seller routing, including pending-login and anonymous resend behavior.
  - Acceptance: seller auth pages compile within the documented baseline, and verification no longer depends on an authenticated session refresh.

## Quickstart And Unchanged-Supporting-Service Verification

### Verify

- [ ] V005 [US1][US2][US3] Verify quickstart scenarios and unchanged supporting services
  - File: `specs/0007-email-verification-signup/quickstart.md`, `services/notification-service/README.md`, `services/api-gateway/README.md`, `shared/api-catalog.md`, `shared/event-catalog.md`
  - Run through the quickstart checklist against the implemented code paths: pending registration, no profile before readiness, verification-to-ready publication, generic resend, blocked pending login, and Google unchanged behavior.
  - Confirm NotificationService delivery still relies on the unchanged verification-requested event and ApiGateway still reuses the existing `/api/v1/accounts/**` prefix without new gateway-owned behavior.
  - Acceptance: manual validation covers all three stories, reused supporting services remain verification-only, and docs/catalog updates match the implemented contracts.
