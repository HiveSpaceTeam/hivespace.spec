# API Route Contract: Split Identity Service

## Route Ownership

| Gateway path | Target owner | Change |
|---|---|---|
| `/api/v1/identity/**` | IdentityService | New versioned identity/account route family for browser REST and OIDC-adjacent calls. |
| `/api/v1/accounts/**` | IdentityService | Existing account routes are replaced by `/api/v1/identity/**` unless tasks retain a compatibility alias. |
| `/api/v1/admins/**` | IdentityService and UserService split by action | Identity-affecting admin actions move to IdentityService; profile/store review actions stay or move to UserService-owned routes. |
| `/api/v1/users/**` | UserService | Profile, settings, and address routes remain UserService-owned. |
| `/api/v1/stores/**` | UserService | Store registration and store lifecycle remain UserService-owned. |

`/identity/**` is intentionally not part of the target route contract.

## IdentityService-Owned Public Capabilities

| Capability | Auth | Notes |
|---|---|---|
| OIDC authorize/login/callback/logout/token/refresh | Anonymous or authenticated per OIDC flow | Exposed through the versioned IdentityService route family; do not keep `/identity/**`. |
| Account registration | Anonymous | Publishes account-created fact after success. |
| Email verification request | Authenticated user/admin as applicable | Producer becomes IdentityService. |
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
