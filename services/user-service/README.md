# UserService

## Responsibility

UserService owns profile, settings, addresses, store registration, and store lifecycle data. Authentication and authorization state moved to IdentityService.

Source path:

```text
../hivespace.microservice/src/HiveSpace.UserService
```

## Owns

- User profile and settings.
- User addresses.
- Store registration and seller-role transition.
- User-owned profile and store seed data.
- User/store profile and projection integration events.

## Must Not Own

- Product catalog.
- Cart, checkout, orders, fulfillment, or coupons.
- Payment processing or wallet state.
- Notification delivery attempts.
- Media processing.
- Credentials, roles, claims, lockout, account status, token/session issuance, IdentityServer grants, or email verification state.

## Architecture

UserService is narrowed to user-domain data after the IdentityService split. New feature work should use the current backend CQRS/Minimal API direction where practical.

## Runtime

| Item | Value |
|---|---|
| Local HTTP | `http://localhost:5007` |
| Gateway prefixes | `/api/v1/users`, `/api/v1/stores`, profile/store admin routes where applicable |
| Database | SQL Server, UserService-owned profile/settings/address/store schema |
| Auth provider | IdentityService-issued JWTs |

## Planning Notes

- Store creation publishes `StoreCreatedIntegrationEvent`; IdentityService consumes it to grant seller access, so seller app must force token refresh after propagation.
- UserService consumes `IdentityUserCreatedIntegrationEvent` to create the matching profile for an identity-owned account.
- User profile/settings APIs are shared by admin, seller, and buyer apps.
- Buyer avatar changes use the existing profile update API with an optional `avatarFileId`; MediaService still owns upload, storage, and processing.
- Address APIs are buyer-facing but remain UserService-owned because addresses belong to the user profile.
- Downstream services should consume user/store events or maintain projections instead of querying UserService database.
- The split is documented in [ADR-0001](../../architecture/decisions/ADR-0001-split-identity-service.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
