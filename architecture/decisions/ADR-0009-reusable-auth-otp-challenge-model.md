# ADR-0009: Reusable Auth OTP Challenge Model

- **Status**: Draft
- **Date**: 2026-06-21
- **Feature**: `0009-signin-otp-email`
- **Deciders**: Project maintainers

## Context

The current feature only exposes OTP sign-in. However, IdentityService is the natural owner for nearby OTP-based auth flows such as account recovery, password reset confirmation, or step-up verification.

The main decision is whether to:
1. Keep a sign-in-specific OTP entity and rename/refactor later.
2. Introduce a reusable auth-scoped OTP challenge model now.
3. Build a cross-service OTP platform now.

## Decision

Introduce a reusable IdentityService-owned `OtpChallenge` model with a required `Purpose` discriminator.

V1 supports only `Purpose = SignIn`. Public routes remain:
- `POST /api/v1/accounts/otp/request`
- `POST /api/v1/accounts/otp/verify`

Notification delivery also uses a generic event name, `UserOtpChallengeRequestedIntegrationEvent`, while the current business behavior remains sign-in-specific.

For v1, the only allowed purpose is `SignIn`. Integration events expose the purpose as the enum name string. Any new purpose value requires a new feature/spec review and shared catalog review before it becomes supported behavior.

## Consequences

### Positive

- Avoids a later table/entity rename when the next IdentityService OTP auth flow is added.
- Keeps service ownership and auth behavior clear.
- Limits reuse to a justified scope instead of over-designing a platform.

### Negative

- Slightly more abstraction than a one-off sign-in entity.
- Requires purpose-aware repository queries and event wording from the start.

## Alternatives Considered

| Option | Why rejected |
|---|---|
| Sign-in-specific `OtpSignInRequest` only | Fastest now, but likely forces a follow-up schema and naming refactor |
| Cross-service shared OTP platform | Too broad for current evidence and weakens clear service ownership |

## Guardrails

- Only IdentityService auth flows may reuse this model without a new ADR.
- Browser session issuance remains sign-in-specific in this feature.
- Future purposes must not silently piggyback on sign-in endpoints; new public behavior requires new spec and catalog review.
