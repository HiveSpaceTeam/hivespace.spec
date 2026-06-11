# Data Model: Email Verification Before First Sign-In

## IdentityService lifecycle model

### `ApplicationUser`

IdentityService remains the source of truth for account lifecycle.

| Field / concept | Type | Notes |
| --- | --- | --- |
| `Id` | Public user ID | Shared public identity across services. |
| `Email` | string | Normalized for uniqueness and resend-throttle lookup. |
| `PasswordHash` | string | Present only for email/password accounts. |
| `Status` | enum | Must distinguish at least `Pending` and `Active`; existing inactive/suspended states remain identity-owned. |
| `EmailConfirmed` | bool | False for new pending email/password accounts until verification succeeds. |
| `CreatedAt` | timestamp | Registration audit. |
| `UpdatedAt` | timestamp | Lifecycle mutation audit. |
| `ActivatedAt` | timestamp? | Set when a pending account becomes active after verification. |
| `AppContext` | enum/string | Buyer or seller context for return-path and copy decisions where already supported. |

### Lifecycle states

| State | Meaning | Allowed transitions |
| --- | --- | --- |
| `Pending` | Email/password registration succeeded, but sign-in is still blocked until verification completes. | `Pending -> Active`, `Pending -> Pending` on resend |
| `Active` | Verified and eligible for normal sign-in. | Existing status flows continue |
| `Inactive` / `Suspended` | Existing identity-owned account status states outside this feature's primary flow. | Existing status flows continue |

### Invariants

- Email/password buyer and seller registration creates `ApplicationUser` in `Pending`.
- Pending accounts must not receive an authenticated browser session.
- Only successful verification can transition a pending account to `Active`.
- Verification completion must be idempotent.
- Existing Google-based creation/linking behavior stays unchanged.

## Verification delivery and anti-abuse model

### `VerificationRequest`

Logical identity-owned verification send operation.

| Field / concept | Type | Notes |
| --- | --- | --- |
| `UserId` | Public user ID | Present when the email belongs to a pending account. |
| `Email` | string | Normalized email target. |
| `Reason` | enum | `InitialRegistration` or `Resend`. |
| `RequestedAt` | timestamp | Audit and delivery trace. |
| `AppContext` | enum/string | Buyer or seller for template and routing context. |

### `VerificationResendCooldown`

Per-email anti-abuse record.

| Field / concept | Type | Notes |
| --- | --- | --- |
| `EmailKey` | string | Normalized email or hash thereof. |
| `CooldownEndsAt` | timestamp | Requests before this time return generic success without another send. |
| `LastRequestedAt` | timestamp | Supports observability and debugging. |

Implementation note: the cooldown record can live in Redis or another existing backend throttling mechanism; it does not change service ownership.

## Readiness handoff model

### `IdentityUserReadyIntegrationEvent`

Single downstream provisioning fact published by IdentityService when the account becomes usable.

| Field | Type | Purpose |
| --- | --- | --- |
| `UserId` | Public user ID | Correlates the downstream profile creation target. |
| `Email` | string | Supports downstream profile bootstrap where needed. |
| `UserName` | string? | Supports profile bootstrap when display/full-name data is absent. |
| `FullName` | string? | Carries the best available display name for downstream profile creation. |
| `ReadyAt` | timestamp | Audit and idempotency support. |

This event replaces `IdentityUserCreatedIntegrationEvent` entirely.

### Downstream projection model

UserService remains the owner of the user profile.

| Entity | Owner | Feature effect |
| --- | --- | --- |
| `User` profile | UserService | Created from `IdentityUserReadyIntegrationEvent` when downstream provisioning may begin. |
| Google-created profile flows | UserService | Also use `IdentityUserReadyIntegrationEvent` because the account is immediately usable. |

### Readiness invariants

- `IdentityUserReadyIntegrationEvent` is published exactly when downstream provisioning may begin.
- For email/password accounts, readiness occurs only after successful verification.
- For newly created Google accounts, readiness occurs immediately after successful account creation.
- UserService profile creation from readiness must be idempotent by public user ID.
- Repeated verification attempts or repeated ready-event delivery must not create duplicate downstream records or duplicate side effects.
