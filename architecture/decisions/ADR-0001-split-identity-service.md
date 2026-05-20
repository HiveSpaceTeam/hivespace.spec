# ADR-0001: Split IdentityService From UserService

- **Status**: Draft
- **Date**: 2026-05-20
- **Feature**: `0002-split-identity-service`
- **Deciders**: Project maintainers

## Context

UserService currently owns both identity/auth responsibilities and profile/store responsibilities. The requested migration separates credentials, authentication, authorization, roles, IdentityServer, ASP.NET Identity, identity users, account status, and email verification from profile, address, settings, and store ownership.

The project is still in development, so route changes and breaking development migration are acceptable. The constitution requires each service to own its database, forbids direct cross-service database reads, and requires integration contracts for cross-service facts.

## Decision

Create a deployable LiteService `IdentityService` that owns credentials, identity users, sign-in/sign-out, token issuance and refresh, OIDC clients/grants, roles, claims, temporary lockout, account status, and email verification.

Keep `UserService` as the owner of profiles, settings, addresses, store registration, and store lifecycle. IdentityService and UserService use the same public user ID for the same user while keeping separate owned data.

Use asynchronous messages for cross-boundary workflows:

- IdentityService publishes an account-created fact for UserService profile creation.
- UserService publishes a seller-role assignment request after successful store registration.
- Email verification events become IdentityService-owned.

No saga is introduced for this split.

Do not keep or move the old `/identity/**` route. IdentityService browser-facing routes use the versioned API namespace, planned as `/api/v1/identity/**`.

## Consequences

### Positive

- Identity and authorization decisions have one authoritative owner.
- Profile/store data is no longer coupled to credential storage.
- Store registration can remain in the user domain while seller access is granted by identity ownership.
- Shared public user IDs reduce client and projection churn during the development migration.
- Temporary lockout and admin suspend/deactivate behavior stay with the service that controls sign-in.

### Negative / Trade-offs

- Gateway routes, frontend clients, service docs, API catalog, and event catalog must all change together.
- Removing `/identity/**` means existing auth clients must be updated during the same development migration.
- Eventual consistency is introduced for profile creation and seller-role propagation.
- Development data reset or migration must split existing UserService data into two ownership models.

### Risks

- Seller role assignment may lag behind store creation. Mitigation: require token refresh after propagation and make failures observable.
- Account creation may succeed while profile creation fails. Mitigation: idempotent UserService consumer with retry/dead-letter visibility.
- Old clients may call stale routes. Mitigation: update frontend services and catalogs in the same feature.

## Alternatives Considered

| Option | Why rejected |
|---|---|
| Keep one UserService with internal modules | User explicitly chose two deployable services and service-boundary isolation. |
| Create IdentityService as a full layered service | LiteService is enough for the focused identity boundary and avoids unnecessary domain-layer ceremony. |
| Put suspend/account status in UserService | Sign-in and authorization would depend on profile-owned state. |
| Move profile and store data into IdentityService | Profile/store data is business-domain data, not identity data. |
| Use separate profile IDs linked to identity IDs | Adds mapping and projection churn without enough benefit during development. |
| Use synchronous service calls for seller role assignment | Couples store registration success to IdentityService write availability and encourages cross-boundary orchestration in request flow. |
| Add a MassTransit saga | The workflows are simple owner transaction plus async message, with no compensating multi-service state machine required. |
| Preserve `/identity/**` | User clarified there should be no `/identity/**` route. |

## Follow-Up

- Add or update `services/identity-service/` docs.
- Update `services/user-service/` docs to remove identity ownership.
- Update `services/api-gateway/` docs and `shared/api-catalog.md` for route ownership.
- Update `shared/event-catalog.md` for new/changed identity and role propagation messages.
- Update backend and frontend implementation tasks from `specs/0002-split-identity-service/tasks.md`.
