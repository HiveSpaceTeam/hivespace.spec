# Docs And Catalog Tasks

## Service Docs

### Create

- [ ] D001 [US1] Create IdentityService service docs
  - File: `services/identity-service/README.md`, `services/identity-service/api.md`, `services/identity-service/domain.md`, `services/identity-service/data-and-events.md`
  - Document responsibility, source path, local port `5001`, direct IdentityServer authority endpoints, account/admin REST route ownership, `ApplicationUser`, identity SeedData, events published/consumed, and data ownership boundaries.
  - Do not document profile, addresses, settings, or stores as IdentityService-owned data.
  - Acceptance: IdentityService docs are sufficient planning context for future work.

### Update

- [ ] D002 [US2] Update UserService service docs
  - File: `services/user-service/README.md`, `services/user-service/api.md`, `services/user-service/domain.md`, `services/user-service/data-and-events.md`
  - Remove identity, credentials, OIDC, roles, claims, lockout, account status, token/grant, and email verification ownership.
  - Document local port `5007`, profile/settings/address/store ownership, UserService SeedData, `IdentityUserCreatedIntegrationEvent` consumption, and `StoreCreatedIntegrationEvent` publication.
  - Acceptance: UserService docs no longer conflict with IdentityService ownership.

- [ ] D003 [US1] Update ApiGateway docs
  - File: `services/api-gateway/README.md`, `services/api-gateway/api.md`
  - Remove `/identity/**`, document no gateway forwarding for `/.well-known/**`, `/connect/**`, and `/Account/**`, add IdentityService-owned REST routes and UserService `5007` route ownership.
  - Keep gateway scope limited to routing; do not add business logic responsibilities.
  - Acceptance: gateway docs match `contracts/api-routes.md`.

- [ ] D004 [US1] Update NotificationService event docs
  - File: `services/notification-service/data-and-events.md`
  - Update `UserEmailVerificationRequestedIntegrationEvent` producer to IdentityService and keep NotificationService as delivery owner.
  - Do not make NotificationService responsible for identity decisions or email verification state.
  - Acceptance: notification docs reflect changed producer ownership only.

## Shared Catalogs

### Update

- [ ] D005 [US1] Update API catalog route ownership
  - File: `shared/api-catalog.md`
  - Document direct IdentityService authority endpoints, removed `/identity/**`, no `/api/v1/identity/**`, retained account/admin identity REST route families, UserService-owned `/api/v1/users/**` and `/api/v1/stores/**`, and target ports.
  - Do not duplicate endpoints under conflicting owners.
  - Acceptance: API catalog has no stale UserService ownership for identity/account/auth endpoints.

- [ ] D006 [US1] Update event catalog identity/user events
  - File: `shared/event-catalog.md`
  - Add `IdentityUserCreatedIntegrationEvent` with IdentityService owner and UserService consumer.
  - Change email verification event owner to IdentityService.
  - Add IdentityService as a `StoreCreatedIntegrationEvent` consumer for seller onboarding while leaving store event owner as UserService.
  - Acceptance: event catalog matches `contracts/integration-messages.md`.

## Architecture Docs

### Update

- [ ] D007 [US1] Update architecture overview and service inventory
  - File: `architecture/overview.md`, `services/_inventory.md`
  - Add IdentityService to backend service tables with port `5001`; change UserService to profile/settings/address/store responsibility on port `5007`.
  - Update authentication flow to use direct IdentityService authority.
  - Acceptance: overview and inventory describe the split service topology.

- [ ] D008 [US4] Update ADR follow-up for ports and SeedData
  - File: `architecture/decisions/ADR-0001-split-identity-service.md`
  - Add the target local port decision and separate IdentityService/UserService SeedData ownership if not already captured.
  - Keep status `Draft` unless maintainers decide otherwise.
  - Acceptance: ADR explains why IdentityService uses `5001`, UserService uses `5007`, and both services own separate seeders.

### Verify

- [ ] D009 [US4] Verify reused supporting services remain documentation-only context
  - File: `services/catalog-service/`, `services/order-service/`
  - Confirm no docs edits are needed because existing store projection contracts are reused unchanged.
  - If a real contract mismatch is found during implementation, update this task set before editing those service docs.
  - Acceptance: CatalogService and OrderService remain verification-only supporting services.
