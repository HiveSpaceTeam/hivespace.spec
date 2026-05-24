# ADR-0001: Split IdentityService From UserService

- **Status**: Accepted
- **Date**: 2026-05-20
- **Feature**: `0002-split-identity-service`
- **Deciders**: Project maintainers

## Context

UserService currently owns both identity/auth responsibilities and profile/store responsibilities. The requested migration separates credentials, authentication, authorization, roles, IdentityServer, ASP.NET Identity, identity users, account status, and email verification from profile, address, settings, and store ownership.

The project is still in development, so route changes and breaking development migration are acceptable. The constitution requires each service to own its database, forbids direct cross-service database reads, and requires integration contracts for cross-service facts.

## Decision

Create a deployable LiteService `IdentityService` that owns credentials, identity users, sign-in/sign-out, token issuance and refresh, OIDC clients/grants, roles, claims, temporary lockout, account status, and email verification. Its `ApplicationUser` extends ASP.NET Identity `IdentityUser` and only adds `Status`, `CreatedAt`, `UpdatedAt`, `LastLoginAt`, `RoleName`, and nullable `StoreId` as custom fields. The service remains LiteService-shaped, but feature behavior uses the backend CQRS pattern: account, email verification, role, token, and admin identity workflows live as command/query handlers under `HiveSpace.IdentityService.Core/Features`, while API endpoints, consumers, and Razor Page handlers stay thin.

Keep `UserService` as the owner of profiles, settings, addresses, store registration, and store lifecycle. UserService may keep `Email` and `UserName` as non-authoritative profile/display/contact copies, but IdentityService remains authoritative for authentication and uniqueness. IdentityService and UserService use the same public user ID for the same user while keeping separate owned data.

Run IdentityService locally on `http://localhost:5001` and UserService locally on `http://localhost:5007`. Split seed data by ownership: IdentityService seeds roles, claims, identity accounts, account status, lockout defaults, seller authorization references, and OIDC configuration; UserService seeds matching profiles, settings, addresses, stores, and owner/store relationships using the same public user IDs.

Use asynchronous messages for cross-boundary workflows:

- IdentityService publishes an account-created fact for UserService profile creation.
- UserService publishes the existing `StoreCreatedIntegrationEvent` after successful store registration, and IdentityService consumes that fact to grant seller access.
- Email verification events become IdentityService-owned.

No saga is introduced for this split.

Do not keep or move the old `/identity/**` route. Frontend OIDC clients use the IdentityService authority URL directly for IdentityServer protocol traffic through standard public endpoints such as `/.well-known/**`, `/connect/**`, and interactive Razor Pages such as `/Account/**`. ApiGateway does not forward these OIDC protocol/page endpoints. IdentityService-owned account REST endpoints stay in account/admin route families where needed, such as `/api/v1/accounts/**`; do not create `/api/v1/identity/**`.

## Consequences

### Positive

- Identity and authorization decisions have one authoritative owner.
- Profile/store data is no longer coupled to credential storage.
- Store registration can remain in the user domain while seller access is granted by identity ownership.
- Shared public user IDs reduce client and projection churn during the development migration.
- Temporary lockout and admin suspend/deactivate behavior stay with the service that controls sign-in.

### Negative / Trade-offs

- Gateway routes, frontend clients, service docs, API catalog, and event catalog must all change together.
- Removing `/identity/**` means existing auth clients must be updated during the same development migration to use the IdentityService authority URL and IdentityServer issuer/public endpoint layout directly.
- Eventual consistency is introduced for profile creation and seller role propagation from store-created events.
- Development data reset or migration must split existing UserService data into two ownership models.

### Risks

- Seller role assignment may lag behind store creation. Mitigation: require token refresh after propagation and make failures observable.
- Account creation may succeed while profile creation fails. Mitigation: idempotent UserService consumer with retry/dead-letter visibility.
- Old clients may call stale routes. Mitigation: update frontend services and catalogs in the same feature.

## Alternatives Considered

| Option | Why rejected |
|---|---|
| Keep one UserService with internal modules | User explicitly chose two deployable services and service-boundary isolation. |
| Create IdentityService as a full layered service | LiteService plus CQRS handlers is enough for the focused identity boundary and avoids unnecessary separate Application/Infrastructure projects. |
| Put suspend/account status in UserService | Sign-in and authorization would depend on profile-owned state. |
| Move profile and store data into IdentityService | Profile/store data is business-domain data, not identity data. |
| Use separate profile IDs linked to identity IDs | Adds mapping and projection churn without enough benefit during development. |
| Use synchronous service calls for seller role assignment | Couples store registration success to IdentityService write availability and encourages cross-boundary orchestration in request flow. |
| Add a separate seller-role assignment event | Duplicates the existing store-created fact because store creation always grants seller access and `StoreCreatedIntegrationEvent` already carries the owner and store identifiers. |
| Add a MassTransit saga | The workflows are simple owner transaction plus async message, with no compensating multi-service state machine required. |
| Preserve `/identity/**` | User clarified there should be no `/identity/**` route. |
| Create `/api/v1/identity/**` for IdentityServer endpoints | IdentityServer should expose standard public OIDC endpoints rather than app-versioned API routes. |
| Forward IdentityServer public endpoints through ApiGateway | Frontend OIDC clients can use the IdentityService authority URL directly, so gateway forwarding for `/.well-known/**`, `/connect/**`, and `/Account/**` adds unnecessary routing responsibility. |

## Documentation Updates

- `services/identity-service/` documents the new identity boundary.
- `services/user-service/` removes identity ownership and documents profile/store ownership.
- `services/api-gateway/` and `shared/api-catalog.md` document direct IdentityService authority endpoints, removed `/identity/**`, and split account/admin route ownership.
- `shared/event-catalog.md` documents `IdentityUserCreatedIntegrationEvent`, IdentityService-owned email verification events, and IdentityService consumption of `StoreCreatedIntegrationEvent`.
