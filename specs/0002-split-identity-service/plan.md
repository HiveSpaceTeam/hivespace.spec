# Implementation Plan: Split Identity Service

- **Branch:** `0002-split-identity-service`
- **Date:** 2026-05-21
- **Spec:** [spec.md](spec.md)

> Constitution loaded from `.specify/memory/constitution.md` before planning.

---

## Phase 0 - Research

### Existing context read before planning

- [x] `architecture/overview.md`
- [x] `services/_inventory.md`
- [x] `services/user-service/README.md`
- [x] `services/user-service/api.md`
- [x] `services/user-service/domain.md`
- [x] `services/user-service/data-and-events.md`
- [x] `services/api-gateway/README.md`
- [x] `services/api-gateway/api.md`
- [x] `services/notification-service/README.md`
- [x] `services/notification-service/data-and-events.md`
- [x] `shared/event-catalog.md`
- [x] `shared/api-catalog.md`
- [x] `shared/coding-conventions.md`
- [x] `shared/glossary.md`
- [x] Backend source check for current `ApplicationUser`, ASP.NET Identity lockout fields, login flow, and admin status flow

### Technical unknowns

- Identity/User data ownership after service split: resolved in [research.md](research.md).
- Cross-service profile creation and seller role propagation: resolved in [research.md](research.md).
- Public route ownership after accepting route changes, including removal of `/identity/**` and rejection of `/api/v1/identity/**`: resolved in [contracts/api-routes.md](contracts/api-routes.md).
- IdentityService service shape: resolved as LiteService in [research.md](research.md).
- Account lockout and suspension ownership: resolved as IdentityService-owned in [data-model.md](data-model.md).
- Breaking development migration/reset path: resolved in [quickstart.md](quickstart.md).
- Whether to introduce a saga: resolved as no saga in [research.md](research.md).

### Research notes

The selected design creates a deployable LiteService `IdentityService` for identity, auth, roles, claims, email verification, temporary lockout, suspend/deactivate account status, and token/grant ownership. The remaining `UserService` owns profile, settings, addresses, and stores without a separate profile status. The two services use the same public user ID and communicate through ordinary MassTransit events/commands with outbox-backed publication and idempotent consumers. See [ADR-0001](../../architecture/decisions/ADR-0001-split-identity-service.md).

---

## Phase 1 - Architecture & Data Model

### Service placement

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| IdentityService | Owning service | New deployable LiteService owns credentials, IdentityServer/OIDC, ASP.NET Identity user state, tokens/grants, roles, claims, lockout, account status, and email verification. | Add new service docs; add new gateway/API route ownership; add identity-owned messages to event catalog after tasks. |
| UserService | Owning service | Existing service is narrowed to profile, settings, addresses, stores, and store registration without a separate profile status. | Update existing service docs; remove identity ownership; document consumed identity events and produced role-assignment command. |
| ApiGateway | Changed supporting service | Gateway route ownership changes because identity/account/admin identity routes move to IdentityService under versioned API routes while profile/store routes stay with UserService. | Update gateway docs and `shared/api-catalog.md` during catalog update. |
| NotificationService | Changed supporting service | Email verification notification flow must consume identity-owned verification request facts instead of UserService-owned facts. | Update event consumer references only if message ownership or semantics change. |
| CatalogService | Reused supporting service | Continues to consume store reference projections after store events remain UserService-owned. | Verification-only unless store event semantics change. |
| OrderService | Reused supporting service | Continues to consume store reference projections. | Verification-only unless store event semantics change. |
| Frontend apps and `@hivespace/shared` | Changed clients | Auth, account, profile, admin, and seller registration calls must follow changed route ownership. | Update client service modules and i18n/task references during implementation planning. |

### Target ownership

| Capability | Owner after split | Notes |
|---|---|---|
| Credentials, password hash, login/logout | IdentityService | No UserService direct database access. |
| OIDC clients, grants, refresh tokens | IdentityService | IdentityServer protocol traffic uses standard public IdentityServer endpoints; no `/identity/**` or `/api/v1/identity/**` route is planned. |
| Roles, claims, authorization account status | IdentityService | Seller access depends on IdentityService role/claims. |
| Temporary lockout after failed sign-in | IdentityService | Keep ASP.NET Identity lockout fields and login behavior. |
| Suspend/deactivate account status | IdentityService | Map current `UserStatus.Active/Inactive` behavior into identity-owned account status. |
| Email verification | IdentityService | NotificationService may still deliver email. |
| Profile, avatar references, settings | UserService | Same public user ID as IdentityService. |
| Addresses | UserService | Existing address rules stay profile-owned. |
| Stores and store lifecycle | UserService | Store registration emits seller-role assignment command. |

### New aggregates

No new commerce aggregate is introduced in the spec repo. Implementation will physically separate existing identity and profile/store records:

- `IdentityUser` remains identity-owned and may continue to use the existing ASP.NET Identity-compatible model.
- `UserProfile` becomes the UserService-owned profile root keyed by the same public user ID.
- `Store` remains a UserService aggregate root.
- `Address` remains a UserService entity under the profile/user boundary.

### New repository interfaces

Implementation should create or move repository abstractions according to the target backend repo structure:

- IdentityService: identity user/role account management through ASP.NET Identity stores and IdentityServer operational/config stores.
- UserService: profile repository, address repository behavior if split from user aggregate, and store repository.

Repository names and exact file paths are deferred to source-repo inspection during task generation.

### Integration events and commands

All outgoing integration messages must use the transactional outbox. Consumers must be idempotent and failures must be observable.

| Contract | Producer | Consumer(s) | Purpose | Catalog status |
|---|---|---|---|---|
| `IdentityUserCreatedIntegrationEvent` | IdentityService | UserService | Create matching UserService profile with the same public user ID. | New event; add to `shared/event-catalog.md` in catalog update. |
| `SellerRoleAssignmentRequestedIntegrationEvent` | UserService | IdentityService | Request seller role/claim after successful store registration. | New command-like event; add to `shared/event-catalog.md` in catalog update. |
| `UserEmailVerificationRequestedIntegrationEvent` | IdentityService | NotificationService | Trigger verification email delivery from identity-owned email verification workflow. | Existing name may be reused with owner changed; catalog update required. |
| `UserEmailVerifiedIntegrationEvent` | IdentityService | Interested consumers | Signal verified email state. | Existing name may be reused with owner changed; catalog update required. |
| `UserUpdatedIntegrationEvent` | UserService | NotificationService/user projections | Refresh profile/display projections after profile change. | Reused, owner remains UserService. |
| `StoreCreatedIntegrationEvent` | UserService | CatalogService, OrderService | Refresh store reference projections after store registration. | Reused, owner remains UserService. |
| `StoreUpdatedIntegrationEvent` | UserService | CatalogService, OrderService | Refresh store reference projections after store update. | Reused, owner remains UserService. |

### API endpoints and route ownership

See [contracts/api-routes.md](contracts/api-routes.md).

| Area | Target owner | Route direction |
|---|---|---|
| Identity/OIDC | IdentityService | Use IdentityServer standard public endpoints such as `/.well-known/**`, `/connect/**`, and `/Account/**`; do not keep `/identity/**` or create `/api/v1/identity/**`. |
| Account registration/login/email verification | IdentityService | IdentityServer UI uses `/Account/**`; account REST routes that remain necessary stay IdentityService-owned under `/api/v1/accounts/**`. |
| Admin identity/account management | IdentityService | Move identity-affecting admin actions to identity-owned admin routes. |
| Profile/settings/address | UserService | Keep user-owned routes under `/api/v1/users/**` where practical. |
| Stores | UserService | Keep `/api/v1/stores/**`. |

Catalog status: route changes are accepted for this feature and must be applied through `$update-catalog` after task generation, per repository workflow.

### Data model

See [data-model.md](data-model.md).

### Contracts

- [contracts/api-routes.md](contracts/api-routes.md)
- [contracts/integration-messages.md](contracts/integration-messages.md)

### Saga design

No saga required. The feature introduces ordinary async messages between two services, but no long-running state machine with multi-step compensation, timeout-driven orchestration, or saga-owned state. Store registration remains a UserService transaction followed by a seller-role assignment request. Failures are observable and retryable through messaging infrastructure.

### Architecture decision

Created [ADR-0001: Split IdentityService From UserService](../../architecture/decisions/ADR-0001-split-identity-service.md). The ADR is required because this changes service boundaries, data ownership, route ownership, and messaging responsibilities.

---

## Phase 2 - Implementation Plan

### Backend layer order

**Shared contracts**

- [ ] Add `IdentityUserCreatedIntegrationEvent`.
- [ ] Add `SellerRoleAssignmentRequestedIntegrationEvent`.
- [ ] Move or re-own email verification events so IdentityService is the producer.
- [ ] Keep existing store projection events UserService-owned unless implementation discovers a contract mismatch.

**IdentityService LiteService**

- [ ] Create the deployable IdentityService as a LiteService from current UserService identity implementation.
- [ ] Own credentials, ASP.NET Identity user state, temporary lockout fields, account status, roles, claims, OIDC clients/grants, login/logout, token refresh, and email verification.
- [ ] Preserve login behavior that increments failed access counts, locks out when configured, and blocks suspended/inactive accounts.
- [ ] Publish `IdentityUserCreatedIntegrationEvent` after account creation.
- [ ] Consume `SellerRoleAssignmentRequestedIntegrationEvent` idempotently and grant seller role/claims.
- [ ] Publish email verification requested/verified events from identity-owned workflows.

Target folder structure:

```text
../hivespace.microservice/src/HiveSpace.IdentityService/
|-- HiveSpace.IdentityService.Api/
|   |-- Endpoints/
|   |-- Consumers/
|   |-- Extensions/
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
    |-- Exceptions/
    |-- Interfaces/
    |-- Persistence/
    |   |-- EntityConfigurations/
    |   |-- Migrations/
    |   `-- SeedData/
    `-- Services/
```

**UserService domain/application**

- [ ] Remove identity ownership from UserService and retain profile, settings, addresses, and stores.
- [ ] Create UserService profile records from `IdentityUserCreatedIntegrationEvent` using the same public user ID.
- [ ] Keep store registration in UserService and publish seller-role assignment request after store creation.
- [ ] Ensure UserService does not introduce profile status for sign-in, lockout, suspension, or role authorization decisions.

**Core/persistence layer**

- [ ] Separate IdentityService and UserService databases/migrations.
- [ ] Implement breaking development reset/migration path for identity records, lockout fields, account status, profiles, addresses, settings, stores, and roles.
- [ ] Configure transactional outbox for all new outgoing messages.
- [ ] Add idempotency/dedup behavior for new message consumers.
- [ ] Preserve correlation IDs across account creation, profile creation, store registration, and role assignment.

**API layer**

- [ ] Move identity and account endpoints to IdentityService.
- [ ] Keep profile/settings/address/store endpoints in UserService.
- [ ] Update authorization policy wiring so downstream services validate IdentityService-issued JWTs.
- [ ] Update ApiGateway route table for changed route ownership.
- [ ] Update health checks for both services.

### Frontend plan

### Surface(s)

- [x] buyer
- [x] seller
- [x] admin

### Files to update (mandatory order: types -> services -> stores -> components/views -> routes -> i18n)

| Area | Notes |
|---|---|
| Shared auth types/services | Point login, token refresh, account, and email verification calls at IdentityService-owned routes. |
| Shared profile/settings/address services | Keep profile-owned calls on UserService routes. |
| Buyer app | Update auth, profile, address, and checkout address dependencies if route names change. |
| Seller app | Update auth and store registration flow; require token refresh after seller-role propagation. |
| Admin app | Split identity account management from profile/store review workflows, including suspended/inactive status. |
| i18n | Update English and Vietnamese copy for any changed account/profile/admin flows and delayed seller-role propagation state. |

### Verification plan

Backend:

- [ ] Build IdentityService project after split.
- [ ] Build UserService project after identity removal.
- [ ] Build ApiGateway after route changes.
- [ ] Run targeted tests for account creation -> profile creation.
- [ ] Run targeted tests for store registration -> seller-role assignment.
- [ ] Run targeted tests for email verification ownership.
- [ ] Run targeted tests for failed login lockout and suspended/inactive account blocking.

Frontend:

- [ ] Run type-check/build for shared package and affected buyer/seller/admin apps.
- [ ] Manually verify sign-in, lockout/suspended-account feedback, profile/settings/address, store registration, token refresh, and admin account workflows.

Docs/catalog:

- [ ] Run `$speckit-tasks`.
- [ ] Run `$update-catalog` after task generation to update `shared/api-catalog.md`, `shared/event-catalog.md`, service docs, and gateway docs.

---

## Phase 3 - Constitution Compliance Check

- [x] Planning-only repo: no runnable product code added here.
- [x] Service ownership respected: IdentityService owns identity/auth; UserService owns profile/address/settings/store.
- [x] IdentityService planned as a LiteService rather than a full layered service.
- [x] Each service owns its own database and migrations.
- [x] No cross-service database reads planned.
- [x] New integration messages go through MassTransit and transactional outbox.
- [x] Consumers are required to be idempotent and observable on failure.
- [x] No saga artifact created because no MassTransit saga state machine is introduced.
- [x] ADR created for service-boundary and data-ownership change.
- [x] Catalog updates identified for post-task `$update-catalog` phase.
- [x] Frontend work follows shared-first order and requires English/Vietnamese i18n updates when user-facing copy changes.

### Post-design gate

PASS. No constitution violation remains. The accepted breaking migration reduces compatibility work during development, but ownership, messaging, catalog, and documentation changes are explicit.
