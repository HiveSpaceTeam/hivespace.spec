# API Route Contract: Split Identity Service

## Route Ownership

| Gateway path | Target owner | Change |
|---|---|---|
| `/.well-known/**` | IdentityService | IdentityServer discovery endpoints remain standard public OIDC endpoints. |
| `/connect/**` | IdentityService | IdentityServer authorize, token, refresh, userinfo, revocation, introspection, device, callback, and sign-out endpoints remain standard public OIDC endpoints. |
| `/Account/**` | IdentityService | IdentityServer interactive Razor Pages for login, registration, consent, external login, device flow, diagnostics, and access denied move to IdentityService without an `/api/v1/identity` prefix. |
| `/api/v1/accounts/**` | IdentityService | Existing account REST routes move to IdentityService ownership where still needed for registration, email verification, and account actions. |
| `/api/v1/admins/**` | IdentityService and UserService split by action | Identity-affecting admin actions move to IdentityService; profile/store review actions stay or move to UserService-owned routes. |
| `/api/v1/users/**` | UserService | Profile, settings, and address routes remain UserService-owned. |
| `/api/v1/stores/**` | UserService | Store registration and store lifecycle remain UserService-owned. |

`/identity/**` and `/api/v1/identity/**` are intentionally not part of the target route contract.

## IdentityService-Owned Public Capabilities

| Capability | Auth | Notes |
|---|---|---|
| OIDC authorize/login/callback/logout/token/refresh | Anonymous or authenticated per OIDC flow | Exposed through IdentityServer's standard public endpoints such as `/.well-known/**`, `/connect/**`, and `/Account/**`; do not keep `/identity/**` or create `/api/v1/identity/**`. |
| Account registration | Anonymous | Served by IdentityService using existing IdentityServer/account UI or account REST routes; publishes account-created fact after success. |
| Email verification request | Authenticated user/admin as applicable | Producer becomes IdentityService; keep route naming aligned with account REST/UI conventions rather than `/api/v1/identity/**`. |
| Email verification confirm | Anonymous | IdentityService owns verification state. |
| Admin create/list/suspend/deactivate identity accounts | Admin policies | Account status and lockout state are identity-owned. |

## UserService-Owned Public Capabilities

| Capability | Auth | Notes |
|---|---|---|
| Get/update current profile | Authenticated user/admin as applicable | UserService owns profile fields only. |
| Get/update settings | Authenticated user/admin as applicable | UserService owns settings. |
| Address CRUD/default address | Authenticated user | UserService owns addresses. |
| Store registration | Authenticated user | Publishes seller-role assignment request after success. |
| Store lifecycle/profile review | Seller/admin policies as applicable | Authorization still comes from IdentityService-issued tokens. |

## Catalog Update Required

`shared/api-catalog.md` and `services/api-gateway/api.md` must be updated by `$update-catalog` after task generation with final route names and owners.
