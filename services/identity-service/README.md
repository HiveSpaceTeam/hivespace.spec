# IdentityService

## Responsibility

IdentityService owns authentication, authorization, credentials, roles, claims, account status, lockout, email verification, and IdentityServer/OIDC state.

Source path:

```text
../hivespace.microservice/src/HiveSpace.IdentityService
```

## Owns

- ASP.NET Identity users, credentials, roles, claims, logins, and tokens.
- Duende IdentityServer clients, grants, protocol endpoints, and interactive account pages.
- Sign-in, sign-out, token issuance, token refresh, and account registration.
- Account status used for authentication and authorization decisions.
- Temporary failed-login lockout state.
- Email verification state and verification workflows.
- Identity-owned seed users, roles, claims, account status, and OIDC configuration.
- Seller role propagation after UserService publishes a store-created fact.

## Must Not Own

- User profile fields, user settings, addresses, or store lifecycle data.
- Product catalog, cart, checkout, orders, coupons, payments, wallets, notifications, or media processing.
- Business-domain authorization decisions that belong to downstream services after a token is issued.

## Architecture

IdentityService is a LiteService. Identity behavior uses CQRS-style command/query handlers under the service core, while API endpoints, MassTransit consumers, and IdentityServer Razor Page handlers stay thin.

IdentityServer public protocol and page endpoints are served directly from the IdentityService authority URL. ApiGateway does not forward `/.well-known/**`, `/connect/**`, or `/Account/**`.

## Runtime

| Item | Value |
|---|---|
| Local HTTP authority | `http://localhost:5001` |
| Direct authority endpoints | `/.well-known/**`, `/connect/**`, `/Account/**` |
| Gateway REST prefixes | `/api/v1/accounts`, `/api/v1/admins` for identity-affecting account/admin actions |
| Database | SQL Server, IdentityService-owned identity and IdentityServer schema |
| Auth provider | Duende IdentityServer |

## Planning Notes

- IdentityService and UserService share the same public user ID for the same user, but each service writes only its own database.
- Account creation publishes `IdentityUserCreatedIntegrationEvent`; UserService consumes it to create the matching profile.
- Store registration remains UserService-owned. IdentityService consumes `StoreCreatedIntegrationEvent` idempotently to grant seller role/claims and store reference on the identity account.
- Email verification events are IdentityService-owned; NotificationService only delivers the email/notification.
- Seller access may require token refresh after role propagation.
- The split is documented in [ADR-0001](../../architecture/decisions/ADR-0001-split-identity-service.md).
- Standardized integration event naming, inheritance, and publisher policy are documented in [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
