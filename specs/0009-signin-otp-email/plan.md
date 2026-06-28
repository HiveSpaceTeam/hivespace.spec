# Implementation Plan: Email OTP Sign-In

**Branch:** `0009-signin-otp-email`  
**Date:** `2026-06-21`  
**Spec:** [spec.md](spec.md)

> Before writing anything in this file, read `.specify/memory/constitution.md`

## Phase 0 - Research

### Existing context

- [x] `services/identity-service/` - identity-owned auth, session, verification, and account status flows
- [x] `services/notification-service/` - delivery mechanics and notification consumers
- [x] `shared/event-catalog.md` - no conflicting reusable auth OTP event contract
- [x] `shared/api-catalog.md` - no conflicting `/api/v1/accounts/otp/**` routes
- [x] Existing saga usage - not applicable; this feature does not introduce or change a saga state machine

### Key decisions

1. Persist OTP state as a generic `OtpChallenge` entity in IdentityService's SQL database.
2. Scope reuse to future IdentityService auth flows only; do not build a cross-service OTP platform.
3. Keep public v1 routes sign-in-specific while making internal persistence and event naming purpose-based.
4. Use `Purpose = SignIn` as the only supported OTP challenge purpose in this feature.
5. Rename the planned notification event to `UserOtpChallengeRequestedIntegrationEvent`.

## Phase 1 - Architecture and Data Model

### Service placement

OTP sign-in remains an IdentityService-owned auth feature. The reusable part is the internal challenge model, not the service boundary.

| Service | Classification | Reason | Documentation/catalog action |
|---|---|---|---|
| IdentityService | Owning service | Owns authentication, browser session issuance, account eligibility, and reusable auth-scoped OTP challenge state | Update service docs, API/event catalogs, data model, and ADRs |
| NotificationService | Changed supporting service | Consumes OTP challenge notification event and sends OTP email | Update service event docs only |
| ApiGateway | Reused supporting service | Existing `/api/v1/accounts/**` routing already covers the feature | No route changes |
| `@hivespace/shared` | Changed supporting package | Shared frontend OTP request/verify types and service client | No catalog changes |
| Buyer app | Owning frontend surface | OTP sign-in UI flow | App-local work only |
| Seller app | Owning frontend surface | OTP sign-in UI flow | App-local work only |

### New entity: OtpChallenge

See [data-model.md](data-model.md) for the full model.

```csharp
public class OtpChallenge : Entity<OtpChallengeId>
{
    public string EmailNormalized { get; private set; }
    public OtpChallengePurpose Purpose { get; private set; }
    public string ChallengeToken { get; private set; }
    public string Code { get; private set; }
    public DateTimeOffset ExpiresAt { get; private set; }
    public DateTimeOffset CanResendAt { get; private set; }
    public int AttemptCount { get; private set; }
    public bool IsUsed { get; private set; }
    public bool IsInvalidated { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; }
}
```

EF Core table: `otp_challenges`  
Migration name: `AddOtpChallenge`

### Repository interface

```csharp
public interface IOtpChallengeRepository
{
    Task<OtpChallenge?> GetActiveByChallengeTokenAsync(string challengeToken, CancellationToken ct);
    Task<OtpChallenge?> GetLatestActiveByEmailAndPurposeAsync(string emailNormalized, OtpChallengePurpose purpose, CancellationToken ct);
    Task AddAsync(OtpChallenge challenge, CancellationToken ct);
    Task SaveChangesAsync(CancellationToken ct);
}
```

### Integration event

| Event | Producer | Consumer(s) | Catalog action |
|---|---|---|---|
| `UserOtpChallengeRequestedIntegrationEvent` | IdentityService | NotificationService sends OTP email | Add/update event catalog entry |

Event fields: `RecipientEmail`, `OtpCode`, `ExpiresAt`, `Purpose`

Representation decision:
- domain model: `OtpChallengePurpose` enum
- database storage: `int`
- integration event payload: string enum name
- v1 allowed value: `SignIn`

### API endpoints

Public API stays unchanged for v1:

| Method | Path | Auth | Handler | Catalog action |
|---|---|---|---|---|
| POST | `/api/v1/accounts/otp/request` | Anonymous | `RequestOtpSignInHandler` | Existing new endpoint entry remains valid |
| POST | `/api/v1/accounts/otp/verify` | Anonymous | `VerifyOtpSignInHandler` | Existing new endpoint entry remains valid |

## Phase 2 - Implementation Plan

### No saga required

This feature does not introduce a MassTransit saga. The flow is two anonymous REST endpoints plus one notification event published through the transactional outbox.

### Architecture decisions

- [ADR-0008: OTP challenge persistence](../../architecture/decisions/ADR-0008-otp-challenge-persistence.md)
- [ADR-0009: Reusable auth OTP challenge model](../../architecture/decisions/ADR-0009-reusable-auth-otp-challenge-model.md)

### Layer order

#### IdentityService - Domain layer

- [ ] `OtpChallengeId.cs` - strongly typed ID with ULID-backed GUID
- [ ] `OtpChallengePurpose.cs` - enum with `SignIn` as the only v1 value
- [ ] `OtpChallenge.cs` - generic auth-scoped entity with create, increment-attempt, mark-used, invalidate behavior
- [ ] `IOtpChallengeRepository.cs` - purpose-aware repository abstraction

#### IdentityService - Application layer

- [ ] `RequestOtpSignInCommand.cs` - request sign-in OTP by email
- [ ] `RequestOtpSignInCommandHandler.cs`
  - Normalize email
  - Query existing active challenge by normalized email and `Purpose = SignIn`
  - Return generic cooldown response if resend is not allowed yet
  - Resolve account eligibility; buyers and sellers only
  - For valid accounts, create `OtpChallenge` with `Purpose = SignIn`
  - Publish `UserOtpChallengeRequestedIntegrationEvent` with `Purpose = SignIn`
  - Always return `OtpSignInResponseDto`
- [ ] `VerifyOtpSignInCommand.cs` - verify `ChallengeToken + Code`
- [ ] `VerifyOtpSignInCommandHandler.cs`
  - Look up by challenge token only
  - Require `Purpose = SignIn`
  - Enforce expiry, single-use, invalidation, and max-attempt rules
  - Reuse the existing session-issuance path from password sign-in
  - Return `VerifyOtpSignInResponseDto`
- [ ] DTOs stay sign-in-specific for the public API

#### IdentityService - Infrastructure layer

- [ ] `OtpChallengeConfiguration.cs`
  - Table `otp_challenges`
  - Unique index on `challenge_token`
  - Composite lookup index on `email_normalized, purpose`
- [ ] `OtpChallengeRepository.cs`
- [ ] Migration `AddOtpChallenge`
- [ ] `UserOtpChallengeRequestedIntegrationEvent.cs` in shared messaging contracts
- [ ] `OtpChallengeCleanupService.cs`

#### IdentityService - API layer

- [ ] `OtpSignInEndpoints.cs`
- [ ] `RequestOtpSignInRequest.cs`
- [ ] `VerifyOtpSignInRequest.cs`
- [ ] Register endpoints under the existing accounts route group

#### NotificationService - Infrastructure layer

- [ ] `OtpChallengeRequestedConsumer.cs`
  - Consume `UserOtpChallengeRequestedIntegrationEvent`
  - Send OTP email for `Purpose = SignIn`
  - Keep delivery logic idempotent
- [ ] OTP sign-in email template
- [ ] Consumer registration

### Security requirements

| Requirement | Implementation |
|---|---|
| Anti-enumeration | Request endpoint returns the same public success shape regardless of account state |
| Challenge-token-only lookup | Verify endpoint never accepts email |
| Purpose isolation | Only `SignIn` challenges are valid for the sign-in endpoints |
| Session equivalence | Successful OTP verify reuses the password sign-in cookie issuance path |
| Admin exclusion | Admin and system-admin accounts receive the same generic response as unknown accounts |
| Attempt limiting | Max-attempt exhaustion invalidates the challenge |
| Redirect safety | Only same-origin relative return URLs are honored |

## Phase 3 - Frontend Plan

### Surfaces

- [x] buyer
- [x] seller
- [ ] admin

### Files to create or update

| File | Location | Notes |
|---|---|---|
| `types/auth/otp.ts` | `@hivespace/shared` | Shared request/response types |
| `services/otp-auth.service.ts` | `@hivespace/shared` | `requestOtp` and `verifyOtp` only |
| `composables/useOtpTimer.ts` | `@hivespace/shared` | Only if existing countdown utilities are insufficient |
| `stores/otp-auth.store.ts` | buyer and seller apps | Sign-in-only OTP UI state |
| Sign-in page updates | buyer and seller apps | Add OTP sign-in entry point |
| `OtpSignInView.vue` | buyer and seller apps | Request step |
| `OtpCodeEntryView.vue` | buyer and seller apps | Verify step |
| Router entries | buyer and seller apps | `/auth/otp` and `/auth/otp/code` |
| `auth.json` i18n updates | buyer and seller apps | English and Vietnamese together |

### UX requirements

- Six per-digit inputs with auto-advance and backspace behavior
- Live expiry countdown from `expiresAt`
- Resend disabled until `canResendAt`
- Back link to password sign-in
- Route guard when no active `challengeToken` is present in store state

## Constitution Compliance Check

- [x] IdentityService remains the owning service
- [x] No saga introduced
- [x] No changes required in `../hivespace.config`
- [x] Events remain outbox-published and consumer-idempotent
- [x] Public OTP endpoints remain in `shared/api-catalog.md`
- [x] English and Vietnamese frontend text must be updated together
