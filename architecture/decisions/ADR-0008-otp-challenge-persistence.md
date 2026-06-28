# ADR-0008: OTP Challenge Persistence

- **Status**: Draft
- **Date**: 2026-06-21
- **Feature**: `0009-signin-otp-email`
- **Deciders**: Project maintainers

## Context

Feature 0009 introduces email OTP sign-in for buyers and sellers. The implementation also needs to avoid painting IdentityService into a sign-in-only model if nearby auth flows later adopt OTP.

Each OTP challenge produces:
- A 6-digit code sent to the user
- An opaque challenge token returned to the caller
- Metadata such as expiry, cooldown, attempt count, purpose, and invalidation state

The record must be queryable by challenge token and by normalized email plus purpose, and it must survive service restarts until expiry or invalidation.

For this feature, the purpose is modeled as an enum in the domain, stored as an `int` in the database, and exposed as the enum name string in integration events.

## Decision

Store OTP state as an EF Core entity named `OtpChallenge` in IdentityService's existing SQL Server database.

Use a background hosted service to remove expired rows after a configurable grace period.

## Consequences

### Positive

- No new infrastructure dependency.
- Durable state across deploys and restarts.
- Strong consistency for attempt-count and invalidation updates.
- Supports future IdentityService auth flows without replacing persistence.

### Negative

- Requires a cleanup job.
- Slightly more latency than in-memory or cache-backed storage.
- Keeps OTP state in the primary service database rather than an ephemeral store.

## Alternatives Considered

| Option | Why rejected |
|---|---|
| Redis | Not ideal for service-owned durable auth state and vulnerable to eviction/restart loss |
| ASP.NET Identity `UserToken` store | Poor fit for challenge-token-first lookup and purpose-based challenge semantics |
| In-memory cache | Loses unexpired auth state on restart |

## Follow-Up

- Add `otp_challenges` table and indexes.
- Register `OtpChallengeCleanupService`.
- Update IdentityService and NotificationService docs to reference `OtpChallenge` and `UserOtpChallengeRequestedIntegrationEvent`.
