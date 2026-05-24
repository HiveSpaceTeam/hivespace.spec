# Verification Tasks

## Pre-Implementation Checks

### Verify

- [ ] V001 Verify detailed task format and source prerequisites
  - File: `specs/0002-split-identity-service/tasks.md`, `specs/0002-split-identity-service/tasks/*.md`
  - Confirm all detailed tasks have stable IDs, story labels where applicable, exact file paths/file sets, explicit change details, forbidden behavior, and acceptance checks.
  - Acceptance: no task uses vague wording such as "implement feature" without file-level detail.

## Backend Builds And Searches

### Verify

- [ ] V002 [US1] Verify IdentityService build
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj`
  - Run `dotnet build src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj` from `../hivespace.microservice`.
  - Acceptance: build succeeds or only known source-repo baseline warnings remain.

- [ ] V003 [US1] Verify ApiGateway build
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj`
  - Run `dotnet build src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj` from `../hivespace.microservice`.
  - Acceptance: build succeeds with route/cluster updates compiled.

- [ ] V004 [US1] Verify identity route and authority cleanup
  - File: `../hivespace.microservice/`, `../hivespace.web/`, `../hivespace.config/`
  - Search for stale `/identity/**`, `/api/v1/identity/**`, IdentityService `5007`, and UserService `5001` runtime references.
  - Allow only documentation that explicitly describes previous state or migration notes.
  - Acceptance: runtime config points IdentityService to `5001` and UserService to `5007`.

- [ ] V005 [US1] Verify authentication scenarios
  - File: `specs/0002-split-identity-service/quickstart.md`
  - Manually validate buyer, seller, admin, and inactive/suspended sign-in against IdentityService `http://localhost:5001`.
  - Confirm denied accounts do not expose profile/store data.
  - Acceptance: User Story 1 independent test passes.

- [ ] V006 [US2] Verify UserService profile boundary build
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/HiveSpace.UserService.Api.csproj`
  - Run `dotnet build src/HiveSpace.UserService/HiveSpace.UserService.Api/HiveSpace.UserService.Api.csproj` from `../hivespace.microservice`.
  - Manually verify profile, settings, and address flows do not modify credentials, roles, lockout, account status, or email verification.
  - Acceptance: User Story 2 independent test passes.

- [ ] V007 [US3] Verify seller role propagation
  - File: `../hivespace.microservice/src/HiveSpace.UserService/`, `../hivespace.microservice/src/HiveSpace.IdentityService/`
  - Register a store, confirm UserService publishes `StoreCreatedIntegrationEvent`, confirm IdentityService grants seller role/claims idempotently, refresh token, and access seller workflows.
  - Confirm failed propagation remains visible through retry/dead-letter/logs.
  - Acceptance: User Story 3 independent test passes.

- [ ] V008 [US4] Verify split migrations and SeedData
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/`, `../hivespace.microservice/src/HiveSpace.UserService/`
  - Apply development reset/migration path and confirm IdentityService database contains identity-owned records while UserService database contains profile/settings/address/store records.
  - Confirm both seeders use stable shared public user IDs and neither writes the other service's database.
  - Acceptance: User Story 4 independent test passes.

- [ ] V009 [US4] Verify full backend solution build
  - File: `../hivespace.microservice/HiveSpace.sln`
  - Run `dotnet build` from `../hivespace.microservice`.
  - Acceptance: solution build succeeds or only known baseline warnings remain.

## Frontend Builds

### Verify

- [ ] V010 [US1] Verify frontend builds/type-checks
  - File: `../hivespace.web/packages/shared/`, `../hivespace.web/apps/admin/`, `../hivespace.web/apps/seller/`, `../hivespace.web/apps/buyer/`
  - Run configured targeted build/type-check commands for shared package and affected apps.
  - Confirm OIDC authority uses IdentityService `5001`, gateway base URL remains ApiGateway, and no app imports another app directly.
  - Acceptance: affected frontend work compiles and route ownership is reflected.

## Docs And Final Checks

### Verify

- [ ] V011 Verify docs and catalogs are non-empty and consistent
  - File: `services/identity-service/`, `services/user-service/`, `services/api-gateway/`, `services/notification-service/`, `shared/api-catalog.md`, `shared/event-catalog.md`, `architecture/overview.md`, `services/_inventory.md`
  - Confirm changed markdown files are non-empty, links resolve to existing feature artifacts, route/event ownership is consistent, and reused supporting services were not rewritten unnecessarily.
  - Acceptance: docs/catalogs match the implemented service boundary.

- [ ] V012 Verify final source-repo impact/change checks and cleanup
  - File: `../hivespace.microservice/`, `../hivespace.web/`
  - Run required GitNexus detect changes in source repos before committing if implementation changed code.
  - Remove temporary logs, scratch files, build caches, and debug artifacts created during implementation.
  - Acceptance: changed scope matches the split identity feature and no temporary files remain.
