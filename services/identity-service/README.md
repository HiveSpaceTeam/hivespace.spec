# IdentityService

## Responsibility

IdentityService owns authentication, authorization, credentials, roles, claims, account status, lockout, email verification, and IdentityServer/OIDC state.

Source path:

```text
../hivespace.microservice/src/HiveSpace.IdentityService
```

## Owns

- ASP.NET Identity users, credentials, roles, claims, logins, and tokens.
- Duende IdentityServer clients, grants, protocol endpoints, browser auth REST endpoints, and Google external login/linking.
- Sign-in, sign-out, token cookie issuance, token refresh, and account registration.
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

IdentityService is a LiteService. Identity behavior uses CQRS-style command/query handlers under the service core, while API endpoints, MassTransit consumers, and IdentityServer protocol handlers stay thin.

IdentityServer public protocol endpoints are served directly from the IdentityService authority URL. ApiGateway does not forward `/.well-known/**`, `/connect/**`, or `/Account/**`. HiveSpace browser login, registration, refresh, logout, and Google sign-in/linking use `/api/v1/accounts/**` through ApiGateway. Old user-facing IdentityService account pages and `/Account/**` compatibility redirects are not part of the target UI contract.

## Runtime

| Item | Value |
|---|---|
| Local HTTP authority | `http://localhost:5001` |
| Direct authority endpoints | `/.well-known/**`, `/connect/**`; `/Account/**` is unsupported legacy user-facing URL space |
| Gateway REST prefixes | `/api/v1/accounts`, `/api/v1/admins` for identity-affecting account/admin actions |
| Database | SQL Server, IdentityService-owned identity and IdentityServer schema |
| Auth provider | Duende IdentityServer |

## Planning Notes

- IdentityService and UserService share the same public user ID for the same user, but each service writes only its own database.
- Account creation publishes `IdentityUserCreatedIntegrationEvent`; UserService consumes it to create the matching profile.
- Store registration remains UserService-owned. IdentityService consumes `StoreCreatedIntegrationEvent` idempotently to grant seller role/claims and store reference on the identity account.
- Email verification events are IdentityService-owned; NotificationService only delivers the email/notification.
- Browser auth responses set secure HttpOnly token cookies and a CSRF token; responses must not expose access or refresh tokens to frontend scripts.
- Google sign-in is buyer/seller only. New Google-authenticated accounts are normal user accounts; seller access still comes from UserService store onboarding and `StoreCreatedIntegrationEvent` role propagation.
- Existing email/password accounts may link Google only after verified Google email match, explicit consent, and correct password confirmation. When a verified Google email matches an existing unlinked password account, the user must link, sign in with password, reset password, or use another Google account; IdentityService must not create a duplicate Google-only account for that email. Successful linking marks the matching account email verified.
- Old user-facing IdentityService account pages and `AccountCompatibilityEndpoints` are removed; keep only non-page IdentityServer protocol behavior still required by Duende.
- Seller access may require token refresh after role propagation.
- The split is documented in [ADR-0001](../../architecture/decisions/ADR-0001-split-identity-service.md).
- Standardized integration event naming, inheritance, and publisher policy are documented in [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).
- Gateway-mediated cookie browser sessions are documented in [ADR-0003](../../architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
