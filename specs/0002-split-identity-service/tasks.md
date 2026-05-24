# Tasks: Split Identity Service

- **Input**: Design documents from `specs/0002-split-identity-service/`
- **Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md), [ADR-0001](../../architecture/decisions/ADR-0001-split-identity-service.md)
- **Detailed tasks**: Implementation tasks live under `specs/0002-split-identity-service/tasks/`
- **Organization**: This file is the compatibility entrypoint. Detailed files are grouped by implementation ownership, then service/app/package/lib, then action.

## Documentation And Catalog Scope

- Editable owning service docs: `services/identity-service/`, `services/user-service/`
- Editable changed supporting service docs: `services/api-gateway/`, `services/notification-service/`
- Editable shared catalogs after task generation: `shared/api-catalog.md`, `shared/event-catalog.md`
- Verification-only supporting services: `services/catalog-service/`, `services/order-service/`
- Reused unchanged contracts: `UserUpdatedIntegrationEvent`, `StoreUpdatedIntegrationEvent`
- Existing contract with changed consumer set: `StoreCreatedIntegrationEvent` adds IdentityService as a consumer for seller role assignment

## Task Index

| Area | File | Status | Task Count | Notes |
| --- | --- | --- | ---: | --- |
| Backend | [tasks/backend.md](tasks/backend.md) | Not started | 27 | Backend services, shared contracts, migrations, SeedData |
| Frontend | [tasks/frontend.md](tasks/frontend.md) | Not started | 9 | Shared package plus buyer, seller, admin app updates |
| Config | [tasks/config.md](tasks/config.md) | Not started | 7 | Runtime ports, appsettings, gateway clusters, Docker/config sync |
| Docs/Catalog | [tasks/docs-catalog.md](tasks/docs-catalog.md) | Not started | 9 | Service docs, catalogs, architecture docs, ADR follow-up |
| Verification | [tasks/verification.md](tasks/verification.md) | Not started | 12 | Builds, searches, quickstart/manual validation |

Total detailed tasks: **64**

## Dependency Order

1. Read backend/frontend repo instructions and inspect existing source layout: `B001-B003`, `F001`, `V001`.
2. Complete foundational backend contract/scaffold work before story implementation: `B004-B009`, `C001-C002`.
3. Complete User Story 1 authentication boundary tasks: `B010-B016`, `F002-F005`, `C003-C004`, `V002-V005`.
4. Complete User Story 2 profile/user boundary tasks: `B017-B021`, `F006-F007`, `V006`.
5. Complete User Story 3 seller role propagation tasks: `B022-B024`, `F008`, `V007`.
6. Complete User Story 4 migration and SeedData tasks: `B025-B027`, `C005-C007`, `V008-V009`.
7. Complete docs/catalog updates after task implementation details settle: `D001-D009`.
8. Run final verification and cleanup: `V010-V012`.

## Story Traceability

| Story | Detailed task IDs | Independent acceptance |
| --- | --- | --- |
| US1 Authenticate through dedicated identity boundary | B004-B016, F002-F005, C001-C004, D001, D003-D006, V002-V005, V010 | Buyer, seller, admin, and inactive/suspended accounts sign in or are denied through IdentityService on `5001` without UserService profile reads. |
| US2 Manage profile data in user boundary | B017-B021, F006-F007, C002-C004, D002-D004, D006, V006, V010 | Profile, settings, address, and store registration APIs work through UserService on `5007` without modifying credentials, roles, lockout, account status, or email verification. |
| US3 Propagate seller access from store registration | B022-B024, F008, D002, D005, D007, V007, V010 | Store registration publishes `StoreCreatedIntegrationEvent`; IdentityService grants seller access idempotently after propagation and token refresh. |
| US4 Migrate development data to split ownership | B025-B027, C005-C007, D008-D009, V008-V012 | Development reset produces identity-owned records in IdentityService and profile/store-owned records in UserService using matching public user IDs. |

## Suggested MVP Scope

Minimum usable MVP is User Story 1 plus the runtime config needed to launch it: `B001-B016`, `F001-F005`, `C001-C004`, `D001`, `D003-D006`, and `V001-V005`.

## Completion Checklist

- [ ] All detailed task files listed in Task Index exist.
- [ ] All detailed tasks have unique IDs across files.
- [ ] Every implementation task includes file path, exact change detail, forbidden behavior, dependencies/callers where relevant, and acceptance.
- [ ] `tasks.md` dependency order matches the detailed task dependencies.
- [ ] Verification tasks cover builds/searches/checks and quickstart/manual validation.
