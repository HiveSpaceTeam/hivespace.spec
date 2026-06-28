# Backend Tasks

## IdentityService

### Create

- [ ] B001 [US1] Create `RequestOtpSignInCommandHandlerTests` for FR-002, FR-003, FR-004a, FR-004b, FR-004c, FR-010, and FR-011
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Tests/Features/AccountSessions/Commands/RequestOtpSignIn/RequestOtpSignInCommandHandlerTests.cs`
  - Test: `Handle_EligibleBuyerOrSeller_PersistsChallengeAndPublishesEvent`
  - Test: `Handle_UnknownLockedInactiveOrAdminAccount_ReturnsGenericResponseWithoutUsableChallenge`
  - Test: `Handle_RequestDuringCooldown_ReturnsExistingResendTimingWithoutIssuingNewCode`
  - Use in-memory or existing repo test fixtures only; do not require a live SQL Server or RabbitMQ instance.
  - Acceptance: tests compile and fail before `B004`, `B006`, and `B007` are implemented.

- [ ] B002 [US2] Create `VerifyOtpSignInCommandHandlerTests` for FR-005, FR-006, FR-007, and FR-008
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Tests/Features/AccountSessions/Commands/VerifyOtpSignIn/VerifyOtpSignInCommandHandlerTests.cs`
  - Test: `Handle_ValidChallengeTokenAndCode_MarksChallengeUsedAndIssuesSession`
  - Test: `Handle_ExpiredUsedInvalidatedOrMissingChallenge_ReturnsInvalidOrExpiredCode`
  - Test: `Handle_MaxAttemptsReached_InvalidatesChallengeAndReturnsMaxAttemptsExceeded`
  - Test: `Handle_NonSignInPurpose_RejectsVerification`
  - Keep verification lookup token-based only; do not pass email into verify tests.
  - Acceptance: tests compile and fail before `B005` and `B009` are implemented.

- [ ] B003 [US1] Create `OtpChallenge` domain model and repository contract
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/DomainModels/OtpChallenge.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/DomainModels/OtpChallengeId.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/DomainModels/Enums/OtpChallengePurpose.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Interfaces/IOtpChallengeRepository.cs`
  - Add create, increment-attempt, mark-used, and invalidate behavior matching `data-model.md`.
  - Add repository methods for active lookup by challenge token and latest active lookup by normalized email plus purpose.
  - Do not introduce a cross-service OTP abstraction or email-based verify API.
  - Acceptance: the entity and repository surface match the plan, ADR-0008, and ADR-0009.

- [ ] B004 [US1] Create request-side OTP command, validator, handler, and DTOs
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RequestOtpSignIn/RequestOtpSignInCommand.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RequestOtpSignIn/RequestOtpSignInCommandValidator.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RequestOtpSignIn/RequestOtpSignInCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Dtos/OtpSignInResponseDto.cs`
  - Normalize email, enforce buyer/seller-only eligibility, and return the same public success shape for valid, unknown, locked, inactive, or admin accounts.
  - Return `challengeToken`, `expiresAt`, and `canResendAt` on every successful public response path.
  - Publish `UserOtpChallengeRequestedIntegrationEvent` only for valid buyer/seller accounts after persistence.
  - Acceptance: request handling satisfies FR-002 through FR-004c, FR-010, and FR-011 without exposing account existence.

- [ ] B005 [US2] Create verify-side OTP command, validator, handler, and DTOs
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/VerifyOtpSignIn/VerifyOtpSignInCommand.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/VerifyOtpSignIn/VerifyOtpSignInCommandValidator.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/VerifyOtpSignIn/VerifyOtpSignInCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Dtos/VerifyOtpSignInResponseDto.cs`
  - Enforce challenge-token-only lookup, `Purpose = SignIn`, expiry, single-use, invalidation, and maximum-attempt rules.
  - Return only a validated same-origin `redirectUrl`; treat invalid or cross-origin values as null.
  - Reuse existing password sign-in session issuance services rather than duplicating cookie/CSRF behavior.
  - Acceptance: verify handling satisfies FR-005 through FR-008 and contract semantics in `contracts/otp-api.md`.

- [ ] B006 [US1] Create OTP persistence configuration, repository, and migration
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/EntityConfigurations/OtpChallengeConfiguration.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Repositories/OtpChallengeRepository.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/*AddOtpChallenge*.cs`
  - Map table `otp_challenges` with unique `challenge_token` and composite `(email_normalized, purpose)` indexes.
  - Persist `purpose` as `int`, `code` as 6-digit string, and timestamps/flags from `data-model.md`.
  - Keep handler data access behind `IOtpChallengeRepository`; do not query DbContext directly from handlers.
  - Acceptance: schema and repository behavior align with `data-model.md` and ADR-0008.

- [ ] B007 [US1] Create shared messaging contract `UserOtpChallengeRequestedIntegrationEvent`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/UserOtpChallengeRequestedIntegrationEvent.cs`
  - Include only `RecipientEmail`, `OtpCode`, `ExpiresAt`, and string `Purpose`.
  - Derive from the repo's existing `IntegrationEvent` base type and keep the wire contract sign-in-purpose aware.
  - Do not expose `ChallengeToken`, attempt counters, or internal invalidation flags.
  - Acceptance: the shared event matches `data-model.md`, `research.md`, and the event-catalog entry.

- [ ] B008 [US1] Create `OtpSignInEndpoints` and request/verify API request models
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/OtpSignInEndpoints.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Contracts/RequestOtpSignInRequest.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Contracts/VerifyOtpSignInRequest.cs`
  - Map `POST /api/v1/accounts/otp/request` and `POST /api/v1/accounts/otp/verify` under the existing accounts route group.
  - Keep endpoint modules thin and command-driven; do not add business rules inline in the endpoint methods.
  - Preserve existing `/api/v1/accounts/**` ownership and route shape.
  - Acceptance: IdentityService exposes the documented public endpoints with the request/response shapes from `contracts/otp-api.md`.

### Update

- [ ] B009 [US2] Update IdentityService session issuance services for OTP verify reuse
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Services/*` and any password sign-in command/handler files that currently own cookie issuance
  - Extract or reuse the existing browser-session issuance path so password sign-in and OTP verify set identical cookies, refresh-token state, and CSRF behavior.
  - Keep password sign-in behavior unchanged.
  - Acceptance: OTP verify reuses existing session logic and does not create a parallel cookie-writing implementation.

- [ ] B010 [US1] Update IdentityService startup wiring and OTP options binding
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Program.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/*.cs`, and IdentityService appsettings files if the service keeps feature settings there
  - Register validators, repository, endpoint module, and OTP options for expiry minutes, cooldown seconds, max attempts, cleanup interval, and cleanup grace.
  - Keep runtime settings in the backend repo; do not add feature tasks against `../hivespace.config`.
  - Acceptance: IdentityService boots with OTP dependencies configured from service-local settings only.

- [ ] B011 [US3] Update IdentityService background cleanup for expired OTP records
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Infrastructure/OtpChallengeCleanupService.cs` and related DI registration files
  - Delete expired `otp_challenges` rows older than the configured grace period on a recurring cadence.
  - Keep cleanup purpose-agnostic inside IdentityService so future auth-only purposes can reuse it.
  - Acceptance: cleanup leaves active challenges untouched and prevents unbounded growth of expired OTP rows.

## NotificationService

### Create

- [ ] B012 [US1] Create `OtpChallengeRequestedConsumerTests` for OTP email delivery
  - File: `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Tests/Consumers/OtpChallengeRequestedConsumerTests.cs`
  - Test: `Consume_SignInPurpose_SendsOtpEmail`
  - Test: `Consume_DuplicateMessage_RemainsIdempotent`
  - Test: `Consume_UnknownPurpose_FailsObservably`
  - Use repo-standard consumer test doubles; do not require live email delivery infrastructure.
  - Acceptance: tests compile and fail before `B013` and `B014` are implemented.

- [ ] B013 [US1] Create `OtpChallengeRequestedConsumer` and OTP email template
  - File: `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Consumers/OtpChallengeRequestedConsumer.cs` and the existing notification email template directory used for account emails
  - Consume `UserOtpChallengeRequestedIntegrationEvent` and send the OTP email payload for `Purpose = SignIn`.
  - Render the OTP code and expiry in user-facing content without leaking internal persistence terms such as challenge-token storage.
  - Fail observably for unsupported future purposes instead of silently ignoring them.
  - Acceptance: NotificationService can deliver the OTP email through existing delivery infrastructure using the shared contract from `B007`.

### Update

- [ ] B014 [US1] Update NotificationService messaging registration for OTP delivery
  - File: `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Program.cs` and existing MassTransit/consumer registration extensions
  - Register `OtpChallengeRequestedConsumer` in the current messaging pipeline.
  - Preserve retry and dead-letter visibility; do not swallow missing-template or malformed-payload failures.
  - Acceptance: NotificationService startup includes OTP consumer wiring with no unrelated API or hub changes.

## Backend Verification Prep

### Verify

- [ ] B015 [US3] Verify resend behavior invalidates the previous active sign-in challenge
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RequestOtpSignIn/*`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Repositories/OtpChallengeRepository.cs`
  - Confirm resend after cooldown invalidates the prior `OtpChallenge` and issues a new code and challenge token.
  - Confirm dummy responses for ineligible accounts do not create a usable sign-in challenge record.
  - Acceptance: code path satisfies FR-004a, FR-009, FR-010, and FR-011.

- [ ] B016 [US2] Verify endpoint error mapping for OTP verification failures
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/OtpSignInEndpoints.cs` and any shared exception/HTTP mapping used by IdentityService
  - Map missing, expired, used, invalidated, or wrong-code outcomes to `401` with `invalid_or_expired_code`.
  - Map attempt exhaustion to `401` with `max_attempts_exceeded`.
  - Acceptance: HTTP behavior matches `contracts/otp-api.md` for failed verification paths.

- [ ] B017 Verify ApiGateway remains verification-only for this feature
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/`
  - Inspect current gateway route forwarding for `/api/v1/accounts/**` and confirm it already covers the OTP routes.
  - Do not add ApiGateway implementation changes unless source inspection proves the route group is missing or incompatible.
  - Acceptance: OTP feature implementation stays inside IdentityService, NotificationService, and shared messaging unless a real gateway gap is found.
