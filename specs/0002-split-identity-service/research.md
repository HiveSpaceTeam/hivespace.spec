# Research: Split Identity Service

## Decision: Create a deployable LiteService IdentityService

IdentityService will be implemented as a LiteService that owns credentials, ASP.NET Identity user state, IdentityServer/OIDC, token/grant data, roles, claims, temporary lockout, account status, and email verification.

**Rationale**: These responsibilities make authorization decisions and directly affect account security. Keeping them together avoids split-brain account state. A LiteService shape is sufficient because IdentityService is focused on identity infrastructure and account workflows, not a broad commerce domain model.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Internal module split inside UserService | User explicitly selected two deployable services. |
| UserService remains account-status owner | Would force identity authorization to mirror another service's state. |
| Shared identity/profile database | Violates service-owned database rule. |
| Full layered service shape | Adds ceremony before there is a broad identity domain requiring full DDD layering. |

## Decision: Keep lockout and suspend status in IdentityService

IdentityService owns temporary account lockout fields inherited from ASP.NET Identity (`AccessFailedCount`, `LockoutEnabled`, `LockoutEnd`) and owns the project-level account status used to suspend/deactivate accounts.

**Rationale**: Current login behavior uses ASP.NET Identity lockout on failed sign-in attempts and separately blocks users whose status is not active. Both rules affect sign-in and authorization, so both belong in IdentityService after the split.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Move status to UserService | Would make identity authorization depend on profile-owned data. |
| Treat lockout and suspension as the same state | Lockout is temporary failed-login protection; suspension/deactivation is admin/business account status. |

## Decision: Use IdentityServer public endpoints, not `/identity/**` or `/api/v1/identity/**`

OIDC protocol traffic will use the public endpoints exposed by IdentityServer, including discovery, `/connect/**`, and interactive account Razor Pages such as `/Account/**`. Account REST routes that remain necessary stay under account/admin route families owned by IdentityService, such as `/api/v1/accounts/**` and identity-affecting admin routes. The split will not preserve or move the old `/identity/**` route, and it will not introduce `/api/v1/identity/**`.

**Rationale**: Route changes are accepted for this development-phase split. IdentityServer protocol endpoints should not be hidden under an application API prefix because OIDC discovery and client libraries expect the issuer's standard public endpoint layout.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Move `/identity/**` to IdentityService | User clarified there should be no `/identity/**` route. |
| Create `/api/v1/identity/**` for OIDC | Conflicts with normal IdentityServer public endpoint behavior and OIDC client expectations. |
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

## Decision: Request seller role assignment asynchronously

After store registration succeeds, UserService publishes a seller-role assignment request consumed by IdentityService.

**Rationale**: Store creation is user-domain work. Role/claim assignment is identity-domain work. Messaging preserves the boundary and provides retry/dead-letter observability.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| UserService writes identity roles directly | Violates service-owned data. |
| Synchronous HTTP call required for registration success | Couples store creation availability to IdentityService write path and complicates rollback. |

## Decision: No saga

No MassTransit saga state machine is required.

**Rationale**: The workflows are single owner transactions followed by async facts/requests. There is no multi-service sequence requiring compensation or timeout-driven saga state.

**Alternatives considered**:

| Alternative | Reason rejected |
|---|---|
| Store-registration saga | Adds orchestration state for a simple create-store then assign-role flow. |
| Account-creation saga | Profile creation can be retried idempotently from the account-created event. |
