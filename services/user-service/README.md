# UserService

## Responsibility

UserService owns identity and account data. It hosts Duende IdentityServer for OIDC login and token issuance.

Source path:

```text
../hivespace.microservice/src/HiveSpace.UserService
```

## Owns

- User accounts and ASP.NET Identity state.
- OIDC clients, grants, login/logout, token refresh, and email verification.
- Roles and claims used by frontend guards and backend authorization.
- User profile and settings.
- User addresses.
- Store registration and seller-role transition.
- User/store integration events.

## Must Not Own

- Product catalog.
- Cart, checkout, orders, fulfillment, or coupons.
- Payment processing or wallet state.
- Notification delivery attempts.
- Media processing.

## Architecture

UserService is the main legacy exception in the backend. Existing identity functionality can use controllers/Razor Pages/service-layer code because of ASP.NET Identity and Duende IdentityServer integration. New non-identity feature work should still prefer the current backend CQRS/Minimal API direction where practical.

## Runtime

| Item | Value |
|---|---|
| Local HTTPS | `https://localhost:5001` |
| Gateway prefixes | `/identity`, `/api/v1/users`, `/api/v1/accounts`, `/api/v1/admins`, `/api/v1/stores` |
| Database | SQL Server, UserService-owned identity and user schema |
| Auth provider | Duende IdentityServer |

## Planning Notes

- Store creation changes user authorization state; seller app must force token refresh after successful registration.
- User profile/settings APIs are shared by admin, seller, and buyer apps.
- Address APIs are buyer-facing but remain UserService-owned because addresses belong to the user profile.
- Downstream services should consume user/store events or maintain projections instead of querying UserService database.

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
