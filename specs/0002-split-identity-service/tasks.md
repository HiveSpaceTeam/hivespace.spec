# Tasks: Split Identity Service

- **Input**: Design documents from `/specs/0002-split-identity-service/`
- **Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/api-routes.md, contracts/integration-messages.md, quickstart.md, architecture/decisions/ADR-0001-split-identity-service.md
- **Tests**: Automated test tasks are not generated because TDD/automated tests were not explicitly requested. Verification tasks use targeted builds and the manual quickstart flow.
- **Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Include exact file paths in descriptions

## Documentation And Catalog Scope

- Editable owning service docs: `services/identity-service/`, `services/user-service/`
- Editable changed supporting service docs: `services/api-gateway/`, `services/notification-service/`
- Editable shared catalogs after task generation: `shared/api-catalog.md`, `shared/event-catalog.md`
- Verification-only supporting services: `services/catalog-service/`, `services/order-service/`
- Reused unchanged contracts: `StoreCreatedIntegrationEvent`, `StoreUpdatedIntegrationEvent`, `UserUpdatedIntegrationEvent`

## Target IdentityService LiteService Folder Structure

```text
../hivespace.microservice/src/HiveSpace.IdentityService/
|-- HiveSpace.IdentityService.Api/
|   |-- Endpoints/
|   |   |-- IdentityEndpoints.cs
|   |   `-- AdminIdentityEndpoints.cs
|   |-- Consumers/
|   |   `-- Sync/
|   |       `-- SellerRoleAssignmentRequestedConsumer.cs
|   |-- Extensions/
|   |   |-- HostingExtensions.cs
|   |   `-- ServiceCollectionExtensions.cs
|   |-- Pages/Account/              # IdentityServer interactive UI only
|   |-- wwwroot/localization/
|   |-- Program.cs
|   `-- appsettings.json
`-- HiveSpace.IdentityService.Core/
    |-- Features/
    |   |-- Accounts/
    |   |-- EmailVerification/
    |   |-- Roles/
    |   `-- Tokens/
    |-- Identity/
    |   `-- ApplicationIdentityUser.cs
    |-- Exceptions/
    |   `-- IdentityDomainErrorCode.cs
    |-- Interfaces/
    |-- Persistence/
    |   |-- IdentityDbContext.cs
    |   |-- IdentityDbContextFactory.cs
    |   |-- EntityConfigurations/
    |   |-- Migrations/
    |   `-- SeedData/
    `-- Services/
```

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Confirm implementation scope, target repo rules, and source layout before editing source repos.

- [ ] T001 Read backend source repo instructions in `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md`
- [ ] T002 Read frontend source repo instructions in `../hivespace.web/AGENTS.md` and `../hivespace.web/CLAUDE.md`
- [ ] T003 [P] Read backend service architecture guidance in `../hivespace.microservice/docs/agent/service-architecture.md`
- [ ] T004 [P] Read backend startup conventions in `../hivespace.microservice/docs/agent/startup-conventions.md`
- [ ] T005 [P] Read backend coding rules in `../hivespace.microservice/docs/agent/coding-rules.md`
- [ ] T006 [P] Inspect current UserService identity files under `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Pages/Account/`
- [ ] T007 [P] Inspect current UserService profile/store files under `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/`
- [ ] T008 [P] Inspect current shared frontend auth/profile modules under `../hivespace.web/packages/shared/src/features/`
- [ ] T009 [P] Inspect current app auth/profile route usage in `../hivespace.web/apps/admin/src/router/index.ts`, `../hivespace.web/apps/seller/src/router/index.ts`, and `../hivespace.web/apps/buyer/src/router/index.ts`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Establish shared contracts, service scaffolding, and migration boundaries that all user stories depend on.

**CRITICAL**: No user story implementation should begin until this phase is complete.

- [ ] T010 Run GitNexus impact analysis for UserService identity split symbols before source edits, covering `ApplicationUser`, `UserDbContext`, `AccountService`, `AdminService`, `StoreService`, and `UserService` in `../hivespace.microservice/src/HiveSpace.UserService/`
- [ ] T011 Generate a new IdentityService LiteService scaffold at `../hivespace.microservice/src/HiveSpace.IdentityService/` using `.\scripts\new-service.ps1 -ServiceName HiveSpace.IdentityService -TemplateName ms-lite -AddToSolution` from `../hivespace.microservice`
- [ ] T012 Create/verify the generated IdentityService API project file in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj`
- [ ] T013 Create/verify the generated IdentityService Core project file in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/HiveSpace.IdentityService.Core.csproj`
- [ ] T014 Verify the LiteService scaffold has no separate IdentityService Application or Infrastructure projects under `../hivespace.microservice/src/HiveSpace.IdentityService/`
- [ ] T015 Register the generated IdentityService LiteService projects in the solution file `../hivespace.microservice/HiveSpace.sln`
- [ ] T016 [P] Add `IdentityUserCreatedIntegrationEvent` in `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/IdentityUserCreatedIntegrationEvent.cs`
- [ ] T017 [P] Add `SellerRoleAssignmentRequestedIntegrationEvent` in `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/SellerRoleAssignmentRequestedIntegrationEvent.cs`
- [ ] T018 [P] Review email verification event ownership references in `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/UserEmailVerificationRequestedIntegrationEvent.cs`
- [ ] T019 [P] Review email verified event ownership references in `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/UserEmailVerifiedIntegrationEvent.cs`
- [ ] T020 Create IdentityService startup shell in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Program.cs`
- [ ] T021 Create IdentityService hosting extensions in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/HostingExtensions.cs`
- [ ] T022 Create IdentityService service collection extensions in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`
- [ ] T023 Create IdentityService configuration baseline in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/appsettings.json`
- [ ] T024 Create IdentityService Dockerfile in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Dockerfile`
- [ ] T025 Create IdentityService DbContext factory in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/IdentityDbContextFactory.cs`
- [ ] T026 Define breaking development migration/reset strategy in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/README.md`
- [ ] T027 Create IdentityService documentation folder in `services/identity-service/README.md`

**Checkpoint**: Shared contracts and IdentityService scaffolding exist; user story work can begin.

---

## Phase 3: User Story 1 - Authenticate Through Dedicated Identity Boundary (Priority: P1) MVP

**Goal**: Platform users can register, sign in, receive tokens, and reach role-appropriate app areas through IdentityService.

**Independent Test**: Sign in as buyer, seller, admin, and inactive/suspended account through `/api/v1/identity/**`; confirm correct access or denial without UserService profile reads.

### Implementation for User Story 1

- [ ] T028 [P] [US1] Move `ApplicationUser` identity-owned fields into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Identity/ApplicationIdentityUser.cs`
- [ ] T029 [P] [US1] Move IdentityServer client configuration classes into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Configs/`
- [ ] T030 [US1] Create IdentityService DbContext for ASP.NET Identity and IdentityServer stores in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/IdentityDbContext.cs`
- [ ] T031 [US1] Move login Razor Pages into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Login/`
- [ ] T032 [US1] Move register Razor Pages into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Register/`
- [ ] T033 [US1] Move logout, consent, grants, external login, device, diagnostics, and CIBA pages into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/`
- [ ] T034 [US1] Move IdentityService localization assets into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/wwwroot/localization/`
- [ ] T035 [US1] Preserve ASP.NET Identity lockout behavior in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Login/Index.cshtml.cs`
- [ ] T036 [US1] Preserve suspended/inactive account blocking in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Login/Index.cshtml.cs`
- [ ] T037 [US1] Move account email verification endpoints into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`
- [ ] T038 [US1] Move admin identity account management into `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/AdminIdentityEndpoints.cs`
- [ ] T039 [US1] Publish `IdentityUserCreatedIntegrationEvent` after account creation in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/Accounts/Commands/CreateAccount/CreateAccountCommandHandler.cs`
- [ ] T040 [US1] Publish identity-owned email verification events in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/`
- [ ] T041 [US1] Remove `/identity/**` route and add `/api/v1/identity/**` route to IdentityService cluster in `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`
- [ ] T042 [US1] Add IdentityService cluster destination in `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`
- [ ] T043 [US1] Update JWT issuer/audience configuration for IdentityService-issued tokens in `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`
- [ ] T044 [US1] Update shared frontend auth refresh service route usage in `../hivespace.web/packages/shared/src/features/auth/refresh.service.ts`
- [ ] T045 [US1] Update admin auth route configuration in `../hivespace.web/apps/admin/src/config/index.ts`
- [ ] T046 [US1] Update seller auth route configuration in `../hivespace.web/apps/seller/src/config/index.ts`
- [ ] T047 [US1] Update buyer auth route configuration in `../hivespace.web/apps/buyer/src/config/index.ts`
- [ ] T048 [US1] Update admin account service for identity-owned account/admin routes in `../hivespace.web/apps/admin/src/services/user.service.ts`
- [ ] T049 [US1] Update seller account service for identity-owned email verification routes in `../hivespace.web/apps/seller/src/services/account.service.ts`
- [ ] T050 [US1] Update lockout/suspended-account i18n keys in `../hivespace.web/apps/admin/src/i18n/locales/en/backend-errors.json` and `../hivespace.web/apps/admin/src/i18n/locales/vi/backend-errors.json`
- [ ] T051 [US1] Build IdentityService with `dotnet build src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj` from `../hivespace.microservice`
- [ ] T052 [US1] Build ApiGateway with `dotnet build src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj` from `../hivespace.microservice`

**Checkpoint**: User Story 1 is independently functional when users authenticate through IdentityService and stale `/identity/**` routing is gone.

---

## Phase 4: User Story 2 - Manage Profile Data In User Boundary (Priority: P2)

**Goal**: Authenticated users can manage profile, settings, addresses, and store registration through UserService while identity data remains outside UserService.

**Independent Test**: After sign-in, update profile/settings/address and confirm no IdentityService credentials, roles, lockout, account status, or email verification data is modified by profile-only actions.

### Implementation for User Story 2

- [ ] T053 [P] [US2] Remove password hash, role, email verification, lockout, and account-status ownership from `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs`
- [ ] T054 [P] [US2] Remove `UserStatus` dependency from profile ownership in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Enums/UserStatus.cs`
- [ ] T055 [US2] Update UserService `ApplicationUser` or replacement profile entity in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Identity/ApplicationUser.cs`
- [ ] T056 [US2] Update UserService EF mapping for profile, settings, addresses, and stores in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Data/UserDbContext.cs`
- [ ] T057 [US2] Update UserService profile repository mapping in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Mappers/UserMapper.cs`
- [ ] T058 [US2] Implement idempotent profile creation consumer in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/IdentityUserCreatedConsumer.cs`
- [ ] T059 [US2] Register `IdentityUserCreatedConsumer` and MassTransit endpoints in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Program.cs`
- [ ] T060 [US2] Update profile service to depend only on UserService-owned profile fields in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserService.cs`
- [ ] T061 [US2] Update user settings service flow in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserService.cs`
- [ ] T062 [US2] Update address service flow in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserAddressService.cs`
- [ ] T063 [US2] Keep profile/settings/address routes on UserService in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Controllers/UserController.cs`
- [ ] T064 [US2] Keep address routes on UserService in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Controllers/UserAddressController.cs`
- [ ] T065 [US2] Update buyer profile service route usage in `../hivespace.web/apps/buyer/src/services/profile.service.ts`
- [ ] T066 [US2] Update buyer settings service route usage in `../hivespace.web/apps/buyer/src/services/user-settings.service.ts`
- [ ] T067 [US2] Update buyer address service route usage in `../hivespace.web/apps/buyer/src/services/address.service.ts`
- [ ] T068 [US2] Update shared user profile feature route usage in `../hivespace.web/packages/shared/src/features/user-profile/user-profile.service.ts`
- [ ] T069 [US2] Build UserService with `dotnet build src/HiveSpace.UserService/HiveSpace.UserService.Api/HiveSpace.UserService.Api.csproj` from `../hivespace.microservice`

**Checkpoint**: User Story 2 is independently functional when profile, settings, and addresses work without identity-owned data in UserService.

---

## Phase 5: User Story 3 - Propagate Seller Access From Store Registration (Priority: P3)

**Goal**: Store registration remains UserService-owned and grants seller access through an async IdentityService role assignment request.

**Independent Test**: Register a store as a buyer, confirm store creation succeeds, seller access is denied before token refresh/role propagation, and seller access is allowed after IdentityService assigns the role and the user refreshes authorization state.

### Implementation for User Story 3

- [ ] T070 [P] [US3] Keep store aggregate ownership in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/Store/Store.cs`
- [ ] T071 [US3] Remove direct seller role mutation from `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs`
- [ ] T072 [US3] Publish `SellerRoleAssignmentRequestedIntegrationEvent` after store creation in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/StoreService.cs`
- [ ] T073 [US3] Extend store event publisher for role assignment request in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Messaging/Publishers/StoreEventPublisher.cs`
- [ ] T074 [US3] Keep `StoreCreatedIntegrationEvent` and `StoreUpdatedIntegrationEvent` publication unchanged in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Messaging/Publishers/StoreEventPublisher.cs`
- [ ] T075 [US3] Implement idempotent seller role assignment consumer in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Consumers/SellerRoleAssignmentRequestedConsumer.cs`
- [ ] T076 [US3] Register seller role assignment consumer in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Program.cs`
- [ ] T077 [US3] Update seller registration flow to show token-refresh requirement in `../hivespace.web/apps/seller/src/stores/store.store.ts`
- [ ] T078 [US3] Update seller route guard behavior after store registration in `../hivespace.web/apps/seller/src/router/index.ts`
- [ ] T079 [US3] Update seller registration i18n keys in `../hivespace.web/apps/seller/src/i18n/locales/en/register-store.json` and `../hivespace.web/apps/seller/src/i18n/locales/vi/register-store.json`
- [ ] T080 [US3] Verify CatalogService and OrderService store projection consumers still consume unchanged store events in `../hivespace.microservice/src/HiveSpace.CatalogService/` and `../hivespace.microservice/src/HiveSpace.OrderService/`
- [ ] T081 [US3] Build IdentityService and UserService after seller role propagation changes from `../hivespace.microservice`

**Checkpoint**: User Story 3 is independently functional when store registration drives seller access through IdentityService role assignment.

---

## Phase 6: User Story 4 - Migrate Development Data To Split Ownership (Priority: P4)

**Goal**: Developers can reset or migrate development data into the split ownership model with clear identity/profile/store separation.

**Independent Test**: Apply the development reset/migration path, then sign in and verify identity records, profile/settings/address records, and store records are in their owning service databases.

### Implementation for User Story 4

- [ ] T082 [P] [US4] Create IdentityService EF migration for identity users, roles, claims, lockout fields, account status, email verification, OIDC clients, and operational grants in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/`
- [ ] T083 [P] [US4] Create UserService EF migration for profiles, settings, addresses, and stores without identity-owned fields in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Migrations/`
- [ ] T084 [US4] Update UserService data seeders to remove identity-owned seeding from `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/DataSeeder.cs`
- [ ] T085 [US4] Add IdentityService data seeders for buyer, seller, admin, system admin, roles, account status, and lockout defaults in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/SeedData/`
- [ ] T086 [US4] Add development reset instructions for split databases in `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/README.md`
- [ ] T087 [US4] Update local config sync inputs for IdentityService in `../hivespace.config/docker/`
- [ ] T088 [US4] Update docker compose service definitions for IdentityService in `../hivespace.config/docker/docker-compose.yml`
- [ ] T089 [US4] Update backend appsettings sync source for IdentityService in `../hivespace.config/`
- [ ] T090 [US4] Validate quickstart development migration flow in `specs/0002-split-identity-service/quickstart.md`
- [ ] T091 [US4] Build full backend solution with `dotnet build` from `../hivespace.microservice`

**Checkpoint**: User Story 4 is independently functional when local development data can be reset or migrated into split ownership.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, catalog updates, verification, and cleanup across all stories.

- [ ] T092 [P] Update IdentityService docs in `services/identity-service/README.md`, `services/identity-service/api.md`, `services/identity-service/domain.md`, and `services/identity-service/data-and-events.md`
- [ ] T093 [P] Update UserService docs to remove identity ownership in `services/user-service/README.md`, `services/user-service/api.md`, `services/user-service/domain.md`, and `services/user-service/data-and-events.md`
- [ ] T094 [P] Update ApiGateway docs for route ownership in `services/api-gateway/README.md` and `services/api-gateway/api.md`
- [ ] T095 [P] Update NotificationService event ownership notes in `services/notification-service/data-and-events.md`
- [ ] T096 Update public API catalog for `/api/v1/identity/**`, removed `/identity/**`, split admin routes, and retained UserService profile/store routes in `shared/api-catalog.md`
- [ ] T097 Update event catalog for `IdentityUserCreatedIntegrationEvent`, `SellerRoleAssignmentRequestedIntegrationEvent`, and IdentityService-owned email verification events in `shared/event-catalog.md`
- [ ] T098 Update architecture overview service table and auth flow in `architecture/overview.md`
- [ ] T099 Update service inventory to include IdentityService in `services/_inventory.md`
- [ ] T100 [P] Update frontend shared auth/profile i18n references in `../hivespace.web/packages/shared/src/i18n/locales/en/backend-errors.json` and `../hivespace.web/packages/shared/src/i18n/locales/vi/backend-errors.json`
- [ ] T101 [P] Run frontend shared package build with `pnpm --filter @hivespace/shared build` from `../hivespace.web`
- [ ] T102 [P] Run admin app type-check/build verification from `../hivespace.web/apps/admin`
- [ ] T103 [P] Run seller app type-check/build verification from `../hivespace.web/apps/seller`
- [ ] T104 [P] Run buyer app type-check/build verification from `../hivespace.web/apps/buyer`
- [ ] T105 Run `bash scripts/sync-config.sh` from `../hivespace.microservice` if backend appsettings/local settings changed
- [ ] T106 Run GitNexus detect changes for backend scope after implementation in `../hivespace.microservice`
- [ ] T107 Run GitNexus detect changes for frontend scope after implementation in `../hivespace.web`
- [ ] T108 Run manual validation flow from `specs/0002-split-identity-service/quickstart.md`
- [ ] T109 Remove temporary migration/debug artifacts created during implementation from `../hivespace.microservice` and `../hivespace.web`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately.
- **Foundational (Phase 2)**: Depends on Setup completion - blocks all user stories.
- **US1 (Phase 3)**: Depends on Foundational completion.
- **US2 (Phase 4)**: Depends on Foundational completion; can run in parallel with US1 after shared contracts exist, but full manual validation needs IdentityService auth from US1.
- **US3 (Phase 5)**: Depends on US1 for IdentityService role ownership and on US2 for UserService store/profile ownership.
- **US4 (Phase 6)**: Depends on US1 and US2 schema ownership decisions; can start migration drafting after Foundational.
- **Polish (Phase 7)**: Depends on desired story scope completion.

### User Story Dependencies

- **User Story 1 (P1)**: MVP. Establishes IdentityService authentication and authorization boundary.
- **User Story 2 (P2)**: Requires shared public user ID and account-created contract from Foundational; does not require seller role propagation.
- **User Story 3 (P3)**: Requires IdentityService role ownership and UserService store ownership.
- **User Story 4 (P4)**: Requires final ownership shape from US1/US2 and should be finalized after US3 if seed seller accounts depend on role propagation.

### Parallel Opportunities

- T003-T009 can run in parallel after T001-T002.
- T016-T019 can run in parallel after T010.
- T028-T029 can run in parallel after IdentityService scaffolding exists.
- T053-T054 can run in parallel after Foundational.
- T082-T083 can run in parallel after final schema ownership is agreed.
- T092-T095 and T100-T104 can run in parallel during polish.

---

## Parallel Examples

### User Story 1

```text
Task: "Move ApplicationUser identity-owned fields into ../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Identity/ApplicationIdentityUser.cs"
Task: "Move IdentityServer client configuration classes into ../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Configs/"
```

### User Story 2

```text
Task: "Remove password hash, role, email verification, lockout, and account-status ownership from ../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs"
Task: "Remove UserStatus dependency from profile ownership in ../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Enums/UserStatus.cs"
```

### User Story 4

```text
Task: "Create IdentityService EF migration for identity users, roles, claims, lockout fields, account status, email verification, OIDC clients, and operational grants in ../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/"
Task: "Create UserService EF migration for profiles, settings, addresses, and stores without identity-owned fields in ../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Migrations/"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Setup and Foundational tasks.
2. Implement IdentityService LiteService authentication boundary.
3. Remove `/identity/**` and expose identity through `/api/v1/identity/**`.
4. Validate sign-in, role checks, lockout, and suspended/inactive account behavior.

### Incremental Delivery

1. US1: IdentityService authentication and route ownership.
2. US2: UserService profile/settings/address boundary with account-created profile creation.
3. US3: Store registration role propagation.
4. US4: Development migration/reset path.
5. Polish: docs, catalogs, full verification.

### Notes

- [P] tasks = different files and no dependency on incomplete tasks.
- Each source-code task must follow the target repo's GitNexus impact-analysis rules before editing symbols.
- Catalog updates are intentionally in the final phase and must be completed after task generation.
- Avoid adding `ProfileStatus`; UserService has no separate user/profile status for account availability.
- Do not keep `/identity/**` in the target route set.
