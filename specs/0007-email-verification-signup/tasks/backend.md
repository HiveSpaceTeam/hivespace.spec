# Backend Tasks

## Shared Library - `HiveSpace.Infrastructure.Messaging.Shared`

### Create

- [ ] B001 [US2] Create `IdentityUserReadyIntegrationEvent.cs`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/IdentityUserReadyIntegrationEvent.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/IdentityUserCreatedIntegrationEvent.cs`
  - Add a new integration event with `UserId`, `Email`, optional `UserName`, optional `FullName`, and `ReadyAt` fields that matches the feature data model and ADR-0006.
  - Remove `IdentityUserCreatedIntegrationEvent` from the target provisioning flow instead of keeping both created and ready events alive in parallel.
  - Do not rename or repurpose `UserEmailVerificationRequestedIntegrationEvent` or `UserEmailVerifiedIntegrationEvent`.
  - Acceptance: the shared messaging project exposes `IdentityUserReadyIntegrationEvent`, and no target implementation task still depends on `IdentityUserCreatedIntegrationEvent` for downstream profile provisioning.

## IdentityService - `src/HiveSpace.IdentityService`

### Update

- [ ] B002 [US1] Update `ApplicationUser` pending-activation lifecycle
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/DomainModels/ApplicationUser.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/DomainModels/Enum/UserStatus.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/EntityConfigurations/ApplicationUserEntityConfiguration.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Persistence/Migrations/`
  - Add or finalize the `Pending` account-state path for email/password registration, keep `Active` for usable accounts, and persist activation audit data such as `ActivatedAt` if the model does not already support it.
  - Keep `EmailConfirmed = false` for newly registered email/password users until verification succeeds.
  - Do not change Google-based creation/linking into the pending-account model.
  - Acceptance: the IdentityService model and migration set can store pending email/password accounts without issuing an authenticated session.

- [ ] B003 [US1] Update `RegisterAccount` flow
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Dtos/RegisterAccountRequest.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Dtos/RegisterAccountResponse.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RegisterAccount/RegisterAccountCommand.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RegisterAccount/RegisterAccountCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RegisterAccount/RegisterAccountCommandValidator.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`
  - Change registration to return a dedicated verification-sent response payload with masked email, app, resend timing metadata, and confirmation copy instead of `SessionResponse`.
  - Create only the pending identity account, publish the verification-request delivery event, and do not call account-session issuance or write auth/CSRF cookies in the register flow.
  - Preserve duplicate-email handling: active-account duplicates follow the standard exception envelope; pending-account duplicates either return the same verification-sent result shape or the standard exception envelope according to the documented convention.
  - Acceptance: `POST /api/v1/accounts/register` returns `201 Created` with a pending-verification result and leaves the browser unauthenticated.

- [ ] B004 [US3] Update `SignInCommandHandler.cs`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/SignIn/SignInCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/SignIn/SignInCommand.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Exceptions/IdentityDomainErrorCode.cs`
  - Add an explicit pending-account admission check before any cookie/session issuance and surface it through the existing exception response convention with a stable business error code/message.
  - Keep existing invalid-credential, inactive-account, and Google-link behaviors unchanged unless they directly conflict with pending-account blocking.
  - Do not introduce a custom login error body outside the service’s exception envelope.
  - Acceptance: a pending email/password user cannot sign in, receives the standard exception response, and no session cookies are issued.

- [ ] B005 [US3] Create `ResendEmailVerification` flow
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/Commands/ResendEmailVerification/`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Interfaces/Services/IEmailVerificationResendCooldownStore.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Services/DistributedEmailVerificationResendCooldownStore.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`
  - Add a new anonymous command/handler/validator and endpoint for `POST /api/v1/accounts/email-verification/resend`.
  - Implement cooldown persistence through `IDistributedCache` behind `IEmailVerificationResendCooldownStore`, keyed by normalized email, so repeated requests within the window do not trigger another delivery.
  - For unknown, active, and pending emails, return the same `204 No Content` success behavior when the request is accepted; only unexpected/exceptional failures should use the standard exception envelope.
  - Reuse `UserEmailVerificationRequestedIntegrationEvent` for actual resend delivery and do not create a resend-specific event.
  - Acceptance: the new resend endpoint is anonymous, returns `204 No Content`, enforces per-email cooldown, and does not leak whether the email exists.

- [ ] B006 [US2] Update `ConfirmEmailVerificationCommandHandler.cs`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/Commands/ConfirmEmailVerification/ConfirmEmailVerificationCommand.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/Commands/ConfirmEmailVerification/ConfirmEmailVerificationCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`
  - Make successful verification activate the pending account, mark the email confirmed, persist activation audit data, and return `204 No Content`.
  - Publish `IdentityUserReadyIntegrationEvent` exactly once when a pending email/password account becomes usable; keep `UserEmailVerifiedIntegrationEvent` only if the existing platform still consumes it independently.
  - Keep verification idempotent so already-used or already-active scenarios do not create duplicate downstream provisioning work.
  - Acceptance: `POST /api/v1/accounts/email-verification/verify` returns `204 No Content` on success and transitions a pending account to active exactly once.

- [ ] B007 [US2] Update `IdentityEventPublisher.cs` callers
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Interfaces/Messaging/IIdentityEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Messaging/IdentityEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RegisterAccount/RegisterAccountCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Commands/CompleteGoogleCallback/CompleteGoogleCallbackCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AdminIdentity/Commands/CreateAdmin/CreateAdminCommandHandler.cs`
  - Replace `PublishIdentityUserCreatedAsync` with a ready-event publisher API that can publish only when downstream provisioning is allowed.
  - Publish the ready event immediately for newly created Google accounts, publish it after confirmation for email/password accounts, and remove the old admin/password-account profile-provisioning behavior that depended on `IdentityUserCreatedIntegrationEvent`.
  - Do not leave any registration or Google-create path publishing both old and new provisioning events.
  - Acceptance: IdentityService compiles with a single readiness publisher abstraction, Google-created users still provision immediately, and password registrations no longer provision before verification.

## UserService - `src/HiveSpace.UserService`

### Move/Rename

- [ ] B008 [US2] Move/Rename `IdentityUserCreatedConsumer.cs`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/IdentityUserCreatedConsumer.cs` -> `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/IdentityUserReadyConsumer.cs`
  - Rename the consumer to handle `IdentityUserReadyIntegrationEvent`, update the consumed fields to the new contract, and keep profile creation idempotent by `UserId`.
  - Preserve the existing user-profile creation behavior for Google-created users and newly verified email/password users, but do not create a profile before readiness.
  - Do not introduce direct reads into IdentityService storage or any new synchronous cross-service dependency.
  - Acceptance: UserService creates one profile per ready event, skips duplicates safely, and no longer references `IdentityUserCreatedIntegrationEvent`.
