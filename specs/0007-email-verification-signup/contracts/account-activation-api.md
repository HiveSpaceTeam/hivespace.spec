# Account Activation API Contract

## Scope

This feature changes the browser-facing account registration and verification workflow under the existing IdentityService-owned `/api/v1/accounts/**` route group.

Catalog updates are required after task generation for changed and new public endpoints listed here.

## Endpoints

### `POST /api/v1/accounts/register`

- **Auth:** Anonymous
- **Purpose:** Create a pending buyer or seller email/password account, send the first verification email, and return a non-authenticated confirmation result.
- **Behavior:**
  - Does not issue access, refresh, or CSRF cookies.
  - Creates only the pending identity account.
  - Publishes verification-request delivery.
  - Does not create the UserService profile yet.

Suggested success result:

```json
{
  "email": "masked@example.com",
  "app": "buyer",
  "canResendAt": "2026-06-08T10:05:00Z"
}
```

The register success body is data-only. Buyer and seller apps must render their own local confirmation copy instead of depending on a server-supplied `message` field.

Suggested recoverable error cases:

| Case | Contract expectation |
| --- | --- |
| Email already belongs to an active account | If handled through an exception path, return the standard exception response for the service; do not create or replace an account. |
| Email already belongs to a pending password account | Either return the same normal verification-sent result shape that routes the user into resend recovery, or if implemented through an exception path, return the standard exception response; do not create a duplicate account. |

### `POST /api/v1/accounts/login`

- **Auth:** Anonymous
- **Purpose:** Existing password login endpoint, now with explicit pending-account blocking.

Suggested pending-account failure behavior:

If pending-account blocking is implemented through an exception path, it must return the standard exception response shape already used by the service. The business error code/message inside that conventional error response must remain stable enough for buyer and seller apps to route into recovery.

### `POST /api/v1/accounts/email-verification/resend`

- **Auth:** Anonymous
- **Purpose:** Public resend flow for pending accounts.
- **Success response:** `204 No Content`
- **Requirements:**
  - Enforce per-email cooldown.
  - Return the same generic success response for unknown, active, and pending emails.
  - Never create a new account.
  - Reuse existing verification delivery event contract.

On success this endpoint returns no body. Generic success semantics still apply for unknown, active, and pending emails so callers cannot infer account existence from the response.

### `POST /api/v1/accounts/email-verification/verify`

- **Auth:** Anonymous
- **Purpose:** Complete verification, make the pending account usable for normal sign-in, and trigger downstream provisioning.
- **Success response:** `204 No Content`
- **Behavior:**
  - Marks the account verified and active.
  - Publishes `IdentityUserReadyIntegrationEvent`.
  - Publishes `UserEmailVerifiedIntegrationEvent` only if that separate verification-state fact is intentionally retained.
  - Returns no body on success; the frontend should treat `204` as completion and route the user to sign-in with the local success message/copy.

Suggested failure cases:

| Case | Contract expectation |
| --- | --- |
| Invalid token | If handled through an exception path, return the standard exception response and include enough stable business information for the frontend to show resend recovery. |
| Expired token | If handled through an exception path, return the standard exception response and include enough stable business information for the frontend to show resend recovery. |
| Already-used token / already-active account | Return an idempotent or recoverable outcome that must not duplicate downstream ready-event work; if exception-driven, use the standard exception response convention. |

### `POST /api/v1/accounts/email-verification`

- **Auth:** `RequireAdminOrUser`
- **Purpose:** Existing authenticated verification-email send flow.
- **Decision:** Reused unchanged by this feature.

## Integration contract impact

### New

| Contract | Producer | Consumer | Purpose |
| --- | --- | --- | --- |
| `IdentityUserReadyIntegrationEvent` | IdentityService | UserService | Start downstream profile creation and related registration side effects only when the account is usable, carrying the public user ID, email, optional username/full name, and readiness timestamp needed for bootstrap. |

### Reused / retained

| Contract | Producer | Consumer | Purpose |
| --- | --- | --- | --- |
| `UserEmailVerificationRequestedIntegrationEvent` | IdentityService | NotificationService | Deliver initial and resend verification email. |
| `UserEmailVerifiedIntegrationEvent` | IdentityService | Existing verification-state consumers | Preserve identity-owned verified-email fact only if consumers still need it separate from provisioning. |

### Removed by this feature

| Contract | Reason |
| --- | --- |
| `IdentityUserCreatedIntegrationEvent` | Raw identity-row creation is not the downstream boundary. Google-created and email-verified accounts both use `IdentityUserReadyIntegrationEvent` when they become usable, so the old event should be deleted entirely. |
