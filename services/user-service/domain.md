# UserService Domain Model

## Purpose

UserService owns the identity domain for HiveSpace: account state, roles, profile settings, addresses, store registration, and the seller-role transition that follows store creation.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Domain
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `User` | Aggregate root | Account/profile aggregate with email, username, password hash, role, status, email confirmation, optional store link, profile fields, settings, and addresses |
| `Address` | Entity under `User` | User-owned delivery/contact address with full name, phone, street, commune, province, country, optional zip code, type, and default flag |
| `Store` | Aggregate root | Seller store registered by an active user; stores display data, logo media reference, address, owner, and lifecycle status |
| `Role` | Value object | Supported account roles: `Seller`, `Admin`, `SystemAdmin` |
| `UserSettings` | Value object | User theme and culture preferences |
| `Email`, `PhoneNumber`, `DateOfBirth` | Value objects | Validated profile primitives |

## Business Rules

- Users require a valid email, username, password hash, and full name.
- Username is trimmed, must be 3-50 characters, and may contain only letters, digits, `_`, `-`, `@`, and `.`.
- Full name is trimmed and must be 2-100 characters when creating a user.
- Admin and system admin accounts are created with confirmed email; other users are created unconfirmed unless later verified.
- User status is `Active` or `Inactive`; inactive users cannot register stores.
- A user can own at most one store. Store registration assigns the new store ID to the user and changes the user role to `Seller`.
- Store names are unique case-insensitively, 2-100 characters, and cannot contain `<`, `>`, `"`, `|`, `\`, `/`, `*`, `?`, or `:`.
- Store logo references are stored as `LogoFileId` until MediaService processing provides a URL.
- Store description is optional and limited to 500 characters; store address is required and limited to 500 characters.
- Addresses must include full name, phone number, street, commune, province, and country; length limits are enforced by the `Address` entity.
- A user cannot remove their only address and cannot remove the default address.
- Setting one address as default clears default status from every other address for that user.
- Phone numbers normalize to digit-only international-style values and must be 11-15 digits after normalization.
- Date of birth cannot be in the future or more than 120 years in the past.

## Lifecycle

| Lifecycle | States / transitions |
|---|---|
| User status | `Active` and `Inactive` through `Activate()` / `Deactivate()` |
| Email verification | `ConfirmEmail()` marks email as confirmed |
| Store status | `Active` and `Inactive` through `Activate()` / `Deactivate()` |
| Seller transition | `StoreManager.RegisterStoreAsync(...)` validates owner and store uniqueness, creates the store, then calls `User.AssignStore(...)` |
| Profile settings | Theme and culture updates replace the immutable `UserSettings` value object |

## Cross-Service Facts

- UserService publishes user events for downstream `user_refs` projections.
- UserService publishes store events for downstream `store_refs` projections in CatalogService and OrderService.
- MediaService owns file processing; UserService stores avatar/logo file IDs and resolved URLs only as profile/store references.
- UserService does not own catalog, order, payment, or notification delivery decisions.
