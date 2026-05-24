# Research: Split Identity Service

## Decision: Create a deployable LiteService IdentityService

IdentityService will be implemented as a LiteService that owns credentials, ASP.NET Identity user state, IdentityServer/OIDC, token/grant data, roles, claims, temporary lockout, account status, and email verification. Feature behavior inside the LiteService will still follow the backend CQRS direction: account, email verification, role, token, and admin identity workflows are implemented as command/query handlers under `HiveSpace.IdentityService.Core/Features`, with API endpoints and Razor Page handlers kept thin.

**Rationale**: These responsibilities make authorization decisions and directly affect account security. Keeping them together avoids split-brain account state. A LiteService shape is sufficient because IdentityService is focused on identity infrastructure and account workflows, not a broad commerce domain model. Using CQRS inside the LiteService keeps behavior consistent with other backend services while avoiding unnecessary separate Application/Infrastructure projects.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Internal module split inside UserService | User explicitly selected two deployable services. |
| UserService remains account-status owner | Would force identity authorization to mirror another service's state. |
| Shared identity/profile database | Violates service-owned database rule. |
| Full layered service shape | Adds ceremony before there is a broad identity domain requiring full DDD layering. |

## Decision: Use IdentityService port 5001 and UserService port 5007 locally

IdentityService will run locally on `http://localhost:5001`. UserService will run locally on `http://localhost:5007`.

**Rationale**: IdentityService becomes the authority for OIDC discovery, issuer, token, account UI, and identity-owned account/admin REST flows. Keeping IdentityService on `5001` preserves the existing local identity authority convention while moving UserService's narrowed profile/store API to a new port.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| IdentityService on `5007` and UserService on `5001` | Inverts the requested target and leaves the identity authority on the newer UserService profile port. |
| Put both services behind only ApiGateway ports | OIDC clients need a concrete IdentityService authority for direct IdentityServer endpoints, and the services still need distinct local ports for development. |

## Decision: Maintain separate SeedData ownership in both services

IdentityService SeedData will seed identity-owned roles, users, account status, lockout defaults, claims, seller authorization references, and OIDC configuration where applicable. UserService SeedData will seed matching profiles, settings, addresses, stores, and store owner references only. Both seeders use stable shared public user IDs for development fixtures, but each service writes only to its own database.

**Rationale**: The split requires development data to reflect service ownership. SeedData is part of the reset/migration path, so it must be updated in both services rather than only moving identity data.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Seed all development users in IdentityService only | Leaves UserService without profile, address, settings, and store fixtures needed for local workflows. |
| Seed all development users in UserService only | Reintroduces identity ownership into UserService. |
| Cross-service seed writes from one service | Violates service-owned database boundaries. |

## Decision: Keep lockout and suspend status in IdentityService

IdentityService owns temporary account lockout fields inherited from ASP.NET Identity (`AccessFailedCount`, `LockoutEnabled`, `LockoutEnd`) and owns the project-level account status used to suspend/deactivate accounts.

**Rationale**: Current login behavior uses ASP.NET Identity lockout on failed sign-in attempts and separately blocks users whose status is not active. Both rules affect sign-in and authorization, so both belong in IdentityService after the split.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Move status to UserService | Would make identity authorization depend on profile-owned data. |
| Treat lockout and suspension as the same state | Lockout is temporary failed-login protection; suspension/deactivation is admin/business account status. |

## Decision: Use direct IdentityService authority for IdentityServer endpoints, not ApiGateway forwarding

OIDC protocol traffic will use the IdentityService authority URL configured in the frontend clients. Discovery, `/connect/**`, and interactive account Razor Pages such as `/Account/**` are served directly by IdentityService rather than forwarded through ApiGateway. Account REST routes that remain necessary stay under account/admin route families owned by IdentityService, such as `/api/v1/accounts/**` and identity-affecting admin routes. The split will not preserve or move the old `/identity/**` route, and it will not introduce `/api/v1/identity/**`.

**Rationale**: Route changes are accepted for this development-phase split. Frontend OIDC clients already carry an authority URL, and direct authority access avoids making ApiGateway responsible for IdentityServer protocol/page forwarding. IdentityServer protocol endpoints should not be hidden under an application API prefix because OIDC discovery and client libraries expect the issuer's standard public endpoint layout.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Move `/identity/**` to IdentityService | User clarified there should be no `/identity/**` route. |
| Create `/api/v1/identity/**` for OIDC | Conflicts with normal IdentityServer public endpoint behavior and OIDC client expectations. |
| Forward `/.well-known/**`, `/connect/**`, and `/Account/**` through ApiGateway | Frontend clients can use the IdentityService authority URL directly, so gateway forwarding is unnecessary for OIDC protocol/page endpoints. |
| Keep both old and new route families | Adds compatibility surface that is unnecessary during breaking development migration. |

## Decision: Keep UserService for profile, settings, addresses, and stores

UserService will retain user profile, settings, addresses, store registration, and store lifecycle.

**Rationale**: These are user-domain and seller onboarding records, not authentication primitives. Store registration remains the business workflow that requests authorization changes after success.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Move store registration to IdentityService | Stores are business/domain data, not identity data. |
| Create a third ProfileService | Adds service count and migration work without a current business need. |

## Decision: Use the same public user ID across both services

IdentityService and UserService will use the same public user ID for the same user, while each service owns separate data.

**Rationale**: This keeps existing client references, projections, and store ownership references simpler during a breaking development migration.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Separate profile ID linked to identity ID | More mapping work and greater downstream reference churn. |
| Regenerate all user IDs | Unnecessary because route changes are accepted but reference churn is still avoidable. |

## Decision: Create profiles from identity account-created events

IdentityService publishes an account-created fact. UserService consumes it and creates the matching profile record.

**Rationale**: IdentityService remains the source for account creation, while UserService owns profile persistence without direct database coupling.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Client calls both services | Creates partial-failure complexity in the browser and duplicates orchestration logic. |
| Migration-only profile creation | Does not support new runtime account creation. |

## Decision: Grant seller role from StoreCreatedIntegrationEvent

After store registration succeeds, UserService publishes the existing `StoreCreatedIntegrationEvent`. IdentityService consumes that store-created fact and grants seller access to the store owner.

**Rationale**: Store creation is user-domain work. Role/claim assignment is identity-domain work. The existing store-created event already carries the owner user ID and store ID needed by IdentityService. Reusing it avoids a duplicate command-like integration event while preserving the boundary and retry/dead-letter observability.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| UserService writes identity roles directly | Violates service-owned data. |
| Synchronous HTTP call required for registration success | Couples store creation availability to IdentityService write path and complicates rollback. |
| Add a separate seller-role assignment event | Duplicates the store-created fact for this feature because store creation always grants seller access and `StoreCreatedIntegrationEvent` already has `OwnerId` and store `Id`. |

## Decision: No saga

No MassTransit saga state machine is required.

**Rationale**: The workflows are single owner transactions followed by async facts/requests. There is no multi-service sequence requiring compensation or timeout-driven saga state.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Store-registration saga | Adds orchestration state for a simple create-store then assign-role flow. |
| Account-creation saga | Profile creation can be retried idempotently from the account-created event. |
