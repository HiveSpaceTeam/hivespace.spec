# Research Notes: Email OTP Sign-In (0009)

## 1. OTP Challenge Storage Strategy

**Decision**: Store reusable `OtpChallenge` records as an EF Core entity in IdentityService's existing SQL Server database.

**Rationale**:
- IdentityService already depends on SQL Server; no additional infrastructure is required.
- OTP challenge records contain audit-relevant state such as purpose, attempt counts, used/invalidated flags, and timestamps that benefit from ACID guarantees.
- Durable storage survives IdentityService restarts and supports future auth flows without introducing a separate OTP platform.
- The same persistence model can support future IdentityService auth scenarios while keeping ownership inside the service boundary.

**Alternatives considered**:
- **Redis**: Faster lookup but requires cache invalidation logic, could silently evict unexpired records under memory pressure, and adds a second primary persistence mechanism for IdentityService-owned auth state.
- **ASP.NET Identity `UserToken` store**: Not suitable because OTP verification must look up by challenge token before user identity is fully resolved, and a reusable purpose-based challenge model does not fit the user-token table shape cleanly.

See [ADR-0008](../../architecture/decisions/ADR-0008-otp-challenge-persistence.md) and [ADR-0009](../../architecture/decisions/ADR-0009-reusable-auth-otp-challenge-model.md).

## 2. Generic Auth-Scoped OTP Model

**Decision**: Model OTP state as a generic `OtpChallenge` with a required `Purpose` discriminator. V1 only uses `Purpose = SignIn`.

**Rationale**:
- The current feature remains email OTP sign-in only, but nearby IdentityService auth flows such as password reset, account recovery, or step-up verification can reuse the same persistence and lifecycle model later.
- A purpose discriminator avoids a future schema replacement from a sign-in-specific table to a generic OTP table.
- Keeping reuse limited to IdentityService auth flows avoids over-designing a cross-service OTP platform before there is a real need.

**Alternatives considered**:
- **Keep `OtpSignInRequest` sign-in-specific**: simplest today but forces a later rename/schema refactor when the next OTP-based auth flow arrives.
- **Build a cross-service OTP platform now**: too broad for the current scope and would blur service boundaries.

## 3. OTP Code Hashing

**Decision**: Store the code as plain text in `otp_challenges`.

**Rationale**:
- The code is already transmitted as plain text in the notification event used for email delivery.
- OTP codes are short-lived, single-use, and protected by expiry plus a maximum-attempt limit.
- Hashing adds verification complexity without materially improving security under the current delivery design.

**Consideration**: Revisit this if a future design removes plain-text OTP codes from notification delivery.

## 4. Anti-Enumeration and Timing Normalization

**Decision**: `RequestOtpSignInCommandHandler` always returns `200 OK` with an `OtpSignInResponseDto` containing a challenge token, `expiresAt`, and `canResendAt`, regardless of account existence, lock status, or cooldown state.

**Implementation approach**:
- For unknown, locked, inactive, or admin accounts: generate a random dummy challenge token, do not persist a usable sign-in challenge, and return synthetic expiry/cooldown timestamps.
- For active cooldown: return the existing `CanResendAt` from the live sign-in challenge but do not send a new email.
- For valid buyer or seller accounts: generate a real sign-in challenge, persist it, and publish the notification event.

The public response shape is identical in all paths. Internal processing differs, but externally observable behavior remains constant.

## 5. Challenge Token Design

**Decision**: The challenge token is `Guid.NewGuid().ToString("N")`.

**Rationale**:
- 128-bit random opaque token.
- No ordering or timestamp leakage.
- Simple to generate, store, and transmit.

**Lookup**: Unique index on `challenge_token`.

## 6. Cooldown Mechanism

**Decision**: `CanResendAt` is stored on `OtpChallenge`. Cooldown checks are purpose-aware and keyed by normalized email plus `Purpose = SignIn` for this feature.

**Rationale**:
- Future auth flows can use the same entity while keeping cooldown behavior isolated by purpose.
- V1 sign-in requests reuse the same resend semantics already defined in the feature spec.

**Resend flow**: When cooldown has passed, the current active sign-in challenge is invalidated and replaced by a newly issued sign-in challenge with a new code and challenge token.

## 7. Session Issuance on Verification

**Decision**: Reuse the same session-issuance code path as password sign-in.

**Rationale**:
- FR-008 requires a browser session equivalent to successful password sign-in.
- The generic OTP challenge model should not generalize session issuance; that remains purpose-specific application behavior.
- In this feature, only `Purpose = SignIn` may issue browser auth cookies.

## 8. Frontend Shared Package Scope

**Decision**: Keep frontend reuse narrow. Place OTP sign-in API types and service methods in `@hivespace/shared`; keep pages, stores, routes, and i18n app-local.

**Rationale**:
- Buyer and seller apps need the same API contract.
- This feature does not justify a broader frontend OTP framework for unrelated auth flows.

## 9. Event Naming

**Decision**: Rename the planned notification event to `UserOtpChallengeRequestedIntegrationEvent`.

**Rationale**:
- The event remains IdentityService-owned and currently used only for sign-in email delivery.
- The generic name aligns with a purpose-based OTP challenge model and avoids a later contract rename when another auth flow uses OTP delivery.

**Contract shape**:
- `RecipientEmail`
- `OtpCode`
- `ExpiresAt`
- `Purpose` as the enum name string; v1 value is `SignIn`

## 10. Gateway Route Impact

**Conclusion**: No gateway configuration changes are required. `/api/v1/accounts/**` already routes to IdentityService.

**Catalog note**: The existing OTP sign-in endpoints remain the public contract for this feature.

## 11. Expired Record Cleanup

**Decision**: Add a background hosted service in IdentityService that periodically deletes expired `otp_challenges` rows older than a configurable grace period.

**Rationale**:
- Prevents unbounded growth.
- Applies equally well to future auth-scoped OTP purposes without requiring a new cleanup mechanism.
