# Data Model: Split Identity Service

## IdentityService

### Identity User

Owns authentication and authorization state.

| Field | Meaning | Rules |
|---|---|---|
| `UserId` | Public user ID shared with UserService | Stable across IdentityService and UserService. |
| `Email` | Login/verification email | Unique within identity ownership. |
| `UserName` | Login/display identifier as required by identity implementation | Unique where existing identity rules require it. |
| `PasswordHash` | Credential secret material | IdentityService only. |
| `AccountStatus` | Account state for sign-in and authorization | Source of truth for auth decisions. |
| `LockoutEnabled` | Whether automatic failed-login lockout can apply | Keep ASP.NET Identity behavior in IdentityService. |
| `LockoutEnd` | Temporary lockout end timestamp | Blocks sign-in while active. |
| `AccessFailedCount` | Failed password attempt counter | Incremented by sign-in flow with lockout-on-failure. |
| `EmailConfirmed` | Email verification state | Source of truth in IdentityService. |
| `Roles` | Role assignments such as Buyer, Seller, Admin, SystemAdmin | Source of truth for access. |
| `Claims` | Authorization claims emitted into tokens | Source of truth for access. |

### OIDC Client and Operational Grant

IdentityService owns OIDC configuration and operational state needed for login, token refresh, grants, and sign-out.

## UserService

### User Profile

Owns non-credential user information and is keyed by the same public `UserId`.

| Field | Meaning | Rules |
|---|---|---|
| `UserId` | Public user ID shared with IdentityService | Reference only; not permission to read identity storage. |
| `FullName` | Profile display/contact name | UserService validation. |
| `PhoneNumber` | Profile contact phone | UserService validation. |
| `DateOfBirth` | Optional profile date of birth | UserService validation. |
| `AvatarFileId` | Media file reference | MediaService owns media lifecycle. |
| `AvatarUrl` | Resolved media URL | Updated from media processing events. |
| `Settings` | Theme/culture preferences | UserService-owned value object. |

### Address

UserService-owned entity associated with a profile/user.

| Field | Meaning | Rules |
|---|---|---|
| `AddressId` | Address identifier | Unique per address. |
| `UserId` | Owning public user ID | Must match an existing UserService profile. |
| `FullName`, `Phone`, `Street`, `Commune`, `Province`, `Country`, `ZipCode` | Delivery/contact details | Existing UserService address rules apply. |
| `IsDefault` | Default address marker | Only one default per user. |

### Store

UserService-owned aggregate for seller store registration and lifecycle.

| Field | Meaning | Rules |
|---|---|---|
| `StoreId` | Store identifier | Unique store record. |
| `OwnerUserId` | Public user ID of store owner | Same public user ID used by IdentityService. |
| `Name` | Store name | Unique per existing store rules. |
| `Description`, `Address`, `LogoFileId`, `LogoUrl` | Store display and contact data | UserService owns store fields; MediaService owns media processing. |
| `StoreStatus` | Store lifecycle | Does not replace identity role status. |

## Cross-Service References

- IdentityService does not own profile, settings, address, or store records.
- UserService does not own credentials, password hashes, roles, claims, account status, or email verification state.
- Both services use the same public `UserId`; each service stores only its owned data.
- Other services must consume public APIs or integration events. No direct cross-service database reads are allowed.

## State Transitions

| Workflow | Transition |
|---|---|
| Account creation | IdentityService creates identity user -> publishes account-created fact -> UserService creates profile. |
| Email verification | IdentityService issues verification -> NotificationService delivers -> IdentityService marks email verified -> publishes verified fact. |
| Store registration | UserService creates store -> publishes store-created fact and seller-role assignment request -> IdentityService grants seller role/claims. |
| Temporary lockout | Failed login attempts increment access failure count -> IdentityService sets lockout until `LockoutEnd` when configured. |
| Account suspension/deactivation | IdentityService changes account status -> sign-in/authorization blocked according to identity rules. |
