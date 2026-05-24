# Data Model: Split Identity Service

## IdentityService

### ApplicationUser

IdentityService owns `ApplicationUser`, which extends ASP.NET Identity `IdentityUser` and owns authentication, authorization, audit, and identity-side authorization-reference state.

| Field | Meaning | Rules |
|---|---|---|
| `UserId` | Public user ID shared with UserService | Stable across IdentityService and UserService. |
| `Email` | Login/verification email | Unique within identity ownership. |
| `UserName` | Login/display identifier as required by identity implementation | Unique where existing identity rules require it. |
| `PasswordHash` | Credential secret material | IdentityService only. |
| `Status` | Account state for sign-in and authorization | Source of truth for auth decisions; replaces UserService `UserStatus` ownership. |
| `LockoutEnabled` | Whether automatic failed-login lockout can apply | Keep ASP.NET Identity behavior in IdentityService. |
| `LockoutEnd` | Temporary lockout end timestamp | Blocks sign-in while active. |
| `AccessFailedCount` | Failed password attempt counter | Incremented by sign-in flow with lockout-on-failure. |
| `EmailConfirmed` | Email verification state | Source of truth in IdentityService. |
| `Roles` | Role assignments such as Buyer, Seller, Admin, SystemAdmin | Source of truth for access. |
| `Claims` | Authorization claims emitted into tokens | Source of truth for access. |
| `CreatedAt` | Identity account creation timestamp | Set by IdentityService when the identity account is created. |
| `UpdatedAt` | Last identity account update timestamp | Nullable; updated only by IdentityService identity workflows. |
| `LastLoginAt` | Last successful login timestamp | Nullable; updated by IdentityService sign-in flow. |
| `RoleName` | Primary/current role name used for compatibility or token/claim projection | IdentityService-owned; set to `Seller` when store creation grants seller access. |
| `StoreId` | Seller store reference used for authorization context | Nullable; set from `StoreCreatedIntegrationEvent.Id`. This is a reference only; UserService still owns store records and lifecycle. |

### OIDC Client and Operational Grant

IdentityService owns OIDC configuration and operational state needed for login, token refresh, grants, and sign-out.

### Identity SeedData

IdentityService owns seed data for identity records only.

| Seed record | Rules |
|---|---|
| Roles | Seed buyer/user, seller, admin, and system admin roles through ASP.NET Identity role APIs. |
| Users | Seed buyer, seller, admin, and system admin `ApplicationUser` records with the stable public user IDs also used by UserService profile SeedData. |
| Account state | Seed `Status`, `EmailConfirmed`, lockout defaults, claims, role assignments, and seller `StoreId` references where required for development fixtures. |
| OIDC configuration | Seed IdentityServer clients/configuration and operational defaults where the implementation keeps them in database-backed stores. |

IdentityService SeedData must not create profiles, addresses, settings, or store records.

## UserService

### User

UserService keeps profile-owned fields directly on the existing `User` aggregate in `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain/Aggregates/User/User.cs`. There is no UserService `ApplicationUser`, ASP.NET Identity user model, or identity-to-profile mapper after the split.

| Field | Meaning | Rules |
|---|---|---|
| `UserId` | Public user ID shared with IdentityService | Reference only; not permission to read identity storage. |
| `Email` | Profile/contact email copy | Non-authoritative copy from IdentityService account data; not used for login uniqueness or email verification ownership. |
| `UserName` | Profile/display username copy | Non-authoritative copy from IdentityService account data; not used for authentication uniqueness ownership. |
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

### User SeedData

UserService owns seed data for profile and store records only.

| Seed record | Rules |
|---|---|
| Profiles | Seed matching `User` profile rows for development buyer, seller, admin, and system admin accounts using the same public user IDs as IdentityService. |
| Settings and addresses | Seed user-owned preferences and address fixtures only. |
| Stores | Seed store records, owner references, logo/avatar references, and store lifecycle state without mutating identity roles or claims. |

UserService SeedData must not create credentials, password hashes, roles, claims, account status, lockout state, email verification state, OIDC clients, or operational grants.

## Cross-Service References

- IdentityService does not own profile, settings, address, or store records.
- UserService does not own credentials, password hashes, roles, claims, account status, login timestamps, seller authorization state, or email verification state.
- UserService may store `Email` and `UserName` as profile copies only; IdentityService remains authoritative for authentication and uniqueness.
- Both services use the same public `UserId`; each service stores only its owned data.
- Other services must consume public APIs or integration events. No direct cross-service database reads are allowed.

## State Transitions

| Workflow | Transition |
|---|---|
| Account creation | IdentityService creates identity user -> publishes account-created fact -> UserService creates profile. |
| Email verification | IdentityService issues verification -> NotificationService delivers -> IdentityService marks email verified -> publishes verified fact. |
| Store registration | UserService creates store -> publishes `StoreCreatedIntegrationEvent` -> IdentityService consumes the store-created fact and grants seller role/claims. |
| Temporary lockout | Failed login attempts increment access failure count -> IdentityService sets lockout until `LockoutEnd` when configured. |
| Account suspension/deactivation | IdentityService changes account status -> sign-in/authorization blocked according to identity rules. |
| Development seeding | IdentityService seeds identity-owned records on port `5001`; UserService seeds user-owned records on port `5007`; both use stable shared public user IDs without cross-service database writes. |
