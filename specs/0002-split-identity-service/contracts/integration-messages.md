# Integration Message Contract: Split Identity Service

## New Messages

### IdentityUserCreatedIntegrationEvent

- **Producer**: IdentityService
- **Consumer**: UserService
- **Purpose**: Tell UserService to create a matching profile after identity account creation.

| Field | Meaning |
|---|---|
| `UserId` | Public user ID shared across IdentityService and UserService. |
| `Email` | Account email for initial profile/contact copy if needed. |
| `UserName` | Account username/display seed if needed. |
| `FullName` | Optional initial profile name if collected during registration. |
| `OccurredAt` | Event timestamp. |
| `CorrelationId` | End-to-end request/message correlation. |

Consumer rules:

- UserService must be idempotent by `UserId`.
- If a profile already exists, processing must not create duplicates.
- Missing required identity event fields must remain observable through retry/dead-letter behavior.

### SellerRoleAssignmentRequestedIntegrationEvent

- **Producer**: UserService
- **Consumer**: IdentityService
- **Purpose**: Request seller role/claim assignment after store registration succeeds.

| Field | Meaning |
|---|---|
| `UserId` | Public user ID of the store owner. |
| `StoreId` | Store created by UserService. |
| `Role` | Requested role, expected to be `Seller`. |
| `RequestedAt` | Request timestamp. |
| `CorrelationId` | End-to-end request/message correlation. |

Consumer rules:

- IdentityService must be idempotent by `UserId` + `StoreId` + requested role.
- If the identity user does not exist, the failure must remain observable.
- After role assignment, the user must refresh authorization state before seller-only access is guaranteed.

## Existing Messages With Changed Ownership

| Contract | New owner | Notes |
|---|---|---|
| `UserEmailVerificationRequestedIntegrationEvent` | IdentityService | NotificationService can continue delivery, but event production moves from UserService to IdentityService. |
| `UserEmailVerifiedIntegrationEvent` | IdentityService | Email verification state is identity-owned. |

## Existing Messages Reused Unchanged

| Contract | Owner | Notes |
|---|---|---|
| `UserUpdatedIntegrationEvent` | UserService | Profile/display projection updates remain UserService-owned. |
| `StoreCreatedIntegrationEvent` | UserService | Store references remain UserService-owned. |
| `StoreUpdatedIntegrationEvent` | UserService | Store references remain UserService-owned. |

## Catalog Update Required

`shared/event-catalog.md` must be updated by `$update-catalog` after task generation with final contract names, owners, and consumer sets.
