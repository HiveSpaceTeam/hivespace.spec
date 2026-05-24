# Backend Tasks

## Source Repo Preparation

### Verify

- [ ] B001 Verify backend repo instructions
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.microservice/CLAUDE.md`
  - Read both files before backend edits and apply stricter source-repo rules for CQRS, Minimal API, GitNexus impact checks, package versions, MassTransit consumers, and temporary file cleanup.
  - Do not edit source repo code before completing the required instruction read.
  - Acceptance: notes for implementation confirm both instruction files were read and no backend rule conflicts with this task set.

- [ ] B002 Verify existing UserService identity/profile source layout
  - File: `../hivespace.microservice/src/HiveSpace.UserService/`
  - Inspect current identity Razor Pages, controllers, services, `ApplicationUser`/identity stores, `User` aggregate, store registration flow, event publishers, and `DataSeeder.cs`.
  - Identify direct callers/dependencies before moving or deleting symbols; run GitNexus impact analysis for modified symbols in the source repo.
  - Acceptance: implementation notes list the source files to move, update, remove, and preserve.

- [ ] B003 Verify existing generated IdentityService source layout
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/`
  - Inspect whether the LiteService scaffold already exists and whether it has only `Api` and `Core` projects.
  - Do not add separate Application or Infrastructure projects for IdentityService.
  - Acceptance: IdentityService target layout matches the LiteService shape in `plan.md`.

## Shared Backend Contracts

### Create

- [ ] B004 [US1] Create `IdentityUserCreatedIntegrationEvent`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/IdentityUserCreatedIntegrationEvent.cs`
  - Include fields: `UserId`, `Email`, `UserName`, nullable `FullName`, `OccurredAt`, `CorrelationId`.
  - Keep the contract as a DTO-style integration event; do not expose identity domain entities.
  - Acceptance: contract compiles and can be published by IdentityService and consumed by UserService.

### Update

- [ ] B005 [US1] Update email verification event ownership references
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/UserEmailVerificationRequestedIntegrationEvent.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/UserEmailVerifiedIntegrationEvent.cs`
  - Ensure comments/namespaces/usages do not imply UserService owns email verification.
  - Keep payload shape compatible unless implementation discovers a required field mismatch.
  - Acceptance: IdentityService can publish both events without UserService identity dependencies.

- [ ] B006 [US3] Verify `StoreCreatedIntegrationEvent` supports seller assignment
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Stores/StoreCreatedIntegrationEvent.cs`
  - Confirm payload exposes store `Id` and owner `OwnerId` needed by IdentityService.
  - Do not add `SellerRoleAssignmentRequestedIntegrationEvent`; reuse `StoreCreatedIntegrationEvent`.
  - Acceptance: IdentityService consumer can map `OwnerId` to `ApplicationUser.Id` and `Id` to `ApplicationUser.StoreId`.

## IdentityService

### Create

- [ ] B007 [US1] Create IdentityService startup shell
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Program.cs`
  - Register hosting/service collection extensions, health checks, Minimal API endpoint mapping, Razor Pages for `/Account/**`, MassTransit consumers, and IdentityService SeedData startup call.
  - Keep endpoint handlers thin and delegate behavior to Core feature handlers.
  - Acceptance: project starts with IdentityService dependencies wired and no UserService-hosted IdentityServer dependency.

- [ ] B008 [US1] Create IdentityService service registration extensions
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`
  - Register ASP.NET Identity, IdentityServer stores/config, authentication/authorization, CQRS request/handler abstractions and pipeline behavior, MassTransit outbox, EF Core `IdentityDbContext`, and options binding.
  - Do not add package versions in `.csproj`; use centrally managed package versions.
  - Acceptance: `dotnet build` resolves IdentityService DI registrations.

- [ ] B009 [US1] Create IdentityService hosting extensions
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/HostingExtensions.cs`
  - Map health checks, IdentityServer, Razor Pages, IdentityService endpoints, and MassTransit/SeedData startup behavior.
  - Ensure startup calls `HiveSpace.IdentityService.Core.Infrastructure.DataSeeder.EnsureSeedDataAsync(app)` after migrations.
  - Acceptance: IdentityService pipeline exposes OIDC UI/protocol endpoints directly and identity-owned REST endpoints only where planned.

- [ ] B010 [US1] Create `ApplicationUser`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Identity/ApplicationUser.cs`
  - Inherit from ASP.NET Identity user type used by the repo and include only custom fields: `Status`, `CreatedAt`, `UpdatedAt`, `LastLoginAt`, `RoleName`, nullable `StoreId`.
  - Add methods for login success, activation/suspension, and seller assignment if local patterns support entity methods.
  - Do not include profile, address, settings, avatar, or store aggregate fields.
  - Acceptance: EF can map identity custom fields and UserService no longer owns them.

- [ ] B011 [US1] Create IdentityService persistence
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/IdentityDbContext.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/IdentityDbContextFactory.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/EntityConfigurations/`
  - Configure ASP.NET Identity, IdentityServer stores, custom `ApplicationUser` fields, snake_case table/column rules where existing service conventions require them, and migrations support.
  - Keep IdentityService database separate from UserService.
  - Acceptance: EF migration scaffolding targets IdentityService only.

- [ ] B012 [US1] Create IdentityService account/email/admin feature handlers
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/Accounts/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/Roles/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/Tokens/`
  - Create command/query folders and handlers for account creation, login support, email verification request/confirm, admin identity account management, role assignment, and token support where source implementation requires handlers.
  - Publish `IdentityUserCreatedIntegrationEvent` after account creation and email verification events from identity-owned workflows through the outbox.
  - Acceptance: Identity behavior lives in Core handlers and API/Razor handlers delegate to them.

- [ ] B013 [US3] Create IdentityService store-created consumer
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Consumers/StoreCreatedConsumer.cs`
  - Consume `StoreCreatedIntegrationEvent`, delegate role assignment to a Core role command handler, map `OwnerId` to `ApplicationUser.Id`, map event `Id` to `ApplicationUser.StoreId`, set `RoleName = "Seller"`, grant seller role/claims, and update `UpdatedAt`.
  - Consumer must be idempotent by owner/store identity and must not silently return for missing required user data; preserve retry/dead-letter observability.
  - Acceptance: reprocessing the same event does not duplicate roles/claims and failures are observable.

### Move/Rename

- [ ] B014 [US1] Move/Rename IdentityServer UI and configuration into IdentityService
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Pages/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Configs/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/wwwroot/localization/`
  - Move login, register, logout, consent, grants, external login, device, diagnostics, CIBA pages, IdentityServer client config, and localization assets needed by `/Account/**`.
  - Preserve behavior for failed-login lockout, suspended/inactive blocking, token issuance, and refresh.
  - Acceptance: OIDC UI/protocol flow is served by IdentityService and no `/identity/**` gateway route is required.

### Remove

- [ ] B015 [US1] Remove identity hosting from UserService
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/`
  - Remove ASP.NET Identity and IdentityServer hosting/setup, identity stores/managers, OIDC client/grant configuration, account Razor Pages registration, and identity-owned configuration binding from UserService startup.
  - Keep JWT validation against IdentityService for protected UserService profile/store APIs.
  - Acceptance: UserService no longer hosts IdentityServer or owns credential/token infrastructure.

### Update

- [ ] B016 [US1] Update IdentityService account/admin endpoints
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/AdminIdentityEndpoints.cs`
  - Move email verification REST endpoints and identity-affecting admin account APIs to IdentityService-owned endpoints dispatching Core commands/queries.
  - Do not create `/api/v1/identity/**`; keep account/admin route families where needed.
  - Acceptance: account REST endpoints compile and route ownership matches `contracts/api-routes.md`.

## UserService

### Update

- [ ] B017 [US2] Update User aggregate for profile-only ownership
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs`
  - Remove password hash, role, claims, email verification source-of-truth, lockout, account status, seller authorization `StoreId`, and login timestamp ownership.
  - Keep profile fields, settings, address relationship, and non-authoritative `Email`/`UserName` copies.
  - Acceptance: UserService domain cannot allow, deny, lock, suspend, or authorize authentication.

- [ ] B018 [US2] Update UserService persistence mapping
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Data/UserDbContext.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Migrations/`
  - Map profile-owned fields, settings, addresses, stores, and non-authoritative email/username copies; remove ASP.NET Identity model/store mapping from UserService.
  - Keep UserService database separate from IdentityService and do not read IdentityService database.
  - Acceptance: UserService migration contains no credentials, roles, claims, lockout, account status, token, grant, or email verification ownership.

- [ ] B019 [US2] Create UserService account-created consumer
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/IdentityUserCreatedConsumer.cs`
  - Consume `IdentityUserCreatedIntegrationEvent`, create a matching profile with the same public `UserId`, copy initial `Email`, `UserName`, and optional `FullName` as profile/contact data, and skip duplicate profile creation idempotently.
  - Do not write identity data or silently swallow invalid required event data.
  - Acceptance: account creation can be replayed without duplicate profiles and failures remain retry/dead-letter observable.

- [ ] B020 [US2] Update UserService profile/settings/address/store services
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserService.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/UserAddressService.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/StoreService.cs`
  - Remove credential, role, lockout, email verification, and account-status mutation from profile-only actions.
  - Keep profile, settings, address rules, and store registration validation under UserService ownership.
  - Acceptance: profile-only workflows do not modify identity-owned records.

- [ ] B021 [US2] Update UserService controllers/routes for retained user-owned APIs
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Controllers/UserController.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Controllers/UserAddressController.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Controllers/StoreController.cs`
  - Keep `/api/v1/users/**` and `/api/v1/stores/**` for profile/settings/address/store behavior.
  - Remove account/email/admin identity endpoints that moved to IdentityService.
  - Acceptance: UserService public surface matches user-owned route contract.

## Store Role Propagation

### Update

- [ ] B022 [US3] Update store-created event publication
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Services/StoreService.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Messaging/Publishers/StoreEventPublisher.cs`
  - Publish only the existing `StoreCreatedIntegrationEvent` after successful store creation and keep `StoreUpdatedIntegrationEvent` compatible for existing projections.
  - Do not publish a separate seller-role assignment event and do not mutate IdentityService roles directly.
  - Acceptance: CatalogService, OrderService, NotificationService, and IdentityService can consume the unchanged store-created fact.

- [ ] B023 [US3] Update store-created consumer and messaging endpoint registrations
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Program.cs`
  - Register IdentityService `StoreCreatedConsumer`, UserService `IdentityUserCreatedConsumer`, MassTransit endpoints, outbox, retry, and idempotency/dedup behavior following source repo conventions.
  - Preserve correlation IDs across account creation, profile creation, store registration, and seller role assignment.
  - Acceptance: consumers are discoverable at runtime and publish/consume through MassTransit outbox.

- [ ] B024 [US3] Verify reused store projection consumers
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/`, `../hivespace.microservice/src/HiveSpace.OrderService/`, `../hivespace.microservice/src/HiveSpace.NotificationService/`
  - Confirm existing consumers still accept `StoreCreatedIntegrationEvent` and `StoreUpdatedIntegrationEvent` unchanged.
  - Do not edit reused supporting services unless a real contract mismatch is found.
  - Acceptance: supporting service projections require no docs/catalog changes beyond the changed consumer set.

## Development Data And Migrations

### Create

- [ ] B025 [US4] Create IdentityService migration/reset path
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/README.md`
  - Add migrations for identity users, roles, claims, `Status`, `CreatedAt`, `UpdatedAt`, `LastLoginAt`, `RoleName`, nullable `StoreId`, lockout fields, email verification, OIDC clients, and operational grants.
  - Document breaking development reset steps.
  - Acceptance: IdentityService database can be recreated independently.

- [ ] B026 [US4] Create UserService migration/reset path
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Migrations/`
  - Add migration for profiles, non-authoritative `Email`/`UserName` copies, settings, addresses, and stores without identity-owned fields.
  - Do not preserve identity tables in UserService unless a temporary migration bridge is explicitly documented.
  - Acceptance: UserService database can be recreated independently without credentials/roles/token data.

### Update

- [ ] B027 [US4] Update both service SeedData entrypoints
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Infrastructure/DataSeeder.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/DataSeeder.cs`
  - IdentityService seeds roles, buyer/seller/admin/system admin `ApplicationUser` records, claims, `Status`, email confirmation defaults, lockout defaults, seller `StoreId` references, and OIDC configuration.
  - UserService seeds matching profiles, settings, addresses, stores, avatar/logo references, and owner/store relationships only.
  - Use stable shared public user IDs and never write another service's database.
  - Acceptance: reset seed data supports sign-in, profile access, addresses, stores, and seller authorization after propagation.
