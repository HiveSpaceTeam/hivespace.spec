# Data Model: Google Login and Account Linking

## Overview

This feature introduces no new business aggregate. IdentityService continues to own all account and credential data. Google linkage is identity-owned authentication data.

## Entities

### HiveSpace Account

Existing model: `ApplicationUser`

| Field / state | Meaning | Change for this feature |
| --- | --- | --- |
| `Id` | Public identity user ID shared with UserService profile creation | Reused unchanged |
| `Email` / `UserName` | Identity-owned unique login email | Used to match Google verified email to an existing local password account before starting account linking |
| `EmailConfirmed` | Identity-owned email verification state | Set to true after successful password-confirmed Google linking with the same verified Google email |
| `Status` | Active/suspended/inactive account status | Enforced before Google sign-in or linking can issue a session |
| Lockout fields | Failed sign-in/lockout state | Reused for wrong password attempts during linking |
| `RoleName` / roles | Buyer/seller/admin authorization state | New Google users start as normal user accounts; admin creation through Google is forbidden |
| `StoreId` | Seller store reference granted after store-created event | Reused unchanged; Google sign-in does not set it |

### Google Login Link

Existing storage: ASP.NET Identity external login record, typically `AspNetUserLogins`.

| Field | Meaning | Validation |
| --- | --- | --- |
| `LoginProvider` | External provider name | Must be `Google` for this feature |
| `ProviderKey` | Stable Google subject/provider user ID | Required; must not be exposed to browser scripts |
| `ProviderDisplayName` | Human-readable provider display | `Google` |
| `UserId` | Linked HiveSpace account | Must refer to one active allowed account |

Relationships:

- One HiveSpace account may have zero or one Google login link for this feature.
- A Google provider key may link to only one HiveSpace account.
- Email is used only to start a matching/linking decision; the durable link uses provider key.
- When a verified Google email matches an existing unlinked local password account, the matching account blocks creation of a separate Google-only HiveSpace account for that email.

### Account Linking Request

Temporary IdentityService-owned state.

| Field | Meaning | Validation |
| --- | --- | --- |
| Provider | External provider under consideration | Must be Google |
| Provider key | Stable Google subject/provider user ID | Required and stored only in temporary server-side or HttpOnly state |
| Verified email | Google email used for matching | Required and must be verified by Google |
| Target account ID | Existing HiveSpace account matched by email | Required before password confirmation |
| App | Buyer or seller app context | Admin is not allowed |
| Return URL | Safe frontend return target | Must pass safe-return validation |
| Expires at | Short lifetime for linking | Expired requests cannot be confirmed |

State transitions:

```text
No request
  -> PendingLink (Google callback finds matching unlinked local password account)
  -> LinkedAndSignedIn (user consents and password confirmation succeeds)
  -> Cancelled (user declines or cancels)
  -> Expired (temporary request lifetime passes)
```

### Browser Session

Existing account-session model.

| Field / cookie | Meaning | Change for this feature |
| --- | --- | --- |
| Access token cookie | HttpOnly browser access token | Issued after successful Google sign-in or linking |
| Refresh token cookie | HttpOnly browser refresh token/handle | Issued after successful Google sign-in or linking |
| CSRF token | Readable token mirrored to request header | Rotated after successful Google sign-in or linking |
| Session response | User summary and expiry metadata | Reused shape; must not include access or refresh token values |

## Validation Rules

- Google sign-in is allowed only for buyer and seller app contexts.
- Admin accounts cannot be created or signed in through Google.
- Google account creation requires a verified Google email.
- Existing email/password account linking is attempted only when the Google verified email matches an existing local password account. Linking requires:
  - matching Google verified email;
  - explicit user consent;
  - correct existing account password;
  - active and allowed account state;
  - unexpired pending link state.
- If no same-email local password account exists, the system must not create a pending link request and must continue the normal Google sign-in/sign-up path.
- If a same-email local password account exists, the system must require linking or a safe exit to password sign-in, password reset, or another Google account; it must not create or sign in a separate same-email Google-only account.
- Wrong password attempts during linking must apply existing lockout behavior.
- Declined, failed, abandoned, or expired linking must not create a durable Google link.
- Declined, failed, abandoned, or expired linking must also not create a new HiveSpace account for the same verified email.
- New Google users must be normal user accounts and must not receive seller access until existing seller onboarding completes.

## Events

- `IdentityUserCreatedIntegrationEvent` is reused unchanged when a new Google user account is created.
- `UserEmailVerifiedIntegrationEvent` may be reused unchanged if the existing email-verification implementation publishes it whenever `EmailConfirmed` changes.
- No new integration event is planned.
