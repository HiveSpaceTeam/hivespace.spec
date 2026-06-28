# Verification Tasks

## Backend

### Verify

- [ ] V001 [US1] [US2] [US3] Verify IdentityService build and OTP-focused tests
  - File: `../hivespace.microservice`
  - Run: `dotnet build src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj`
  - Run: targeted IdentityService OTP tests covering `RequestOtpSignInCommandHandlerTests` and `VerifyOtpSignInCommandHandlerTests`
  - Treat documented baseline warnings as non-blocking only when they predate the OTP change; fix any new OTP-related failures.
  - Acceptance: IdentityService compiles and OTP request/verify tests pass.

- [ ] V002 [US1] Verify NotificationService build and OTP consumer tests
  - File: `../hivespace.microservice`
  - Run: `dotnet build src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/HiveSpace.NotificationService.Api.csproj`
  - Run: targeted NotificationService consumer tests covering `OtpChallengeRequestedConsumerTests`
  - Acceptance: NotificationService compiles and OTP consumer tests pass with the new consumer registered.

## Frontend

### Verify

- [ ] V003 [US1] [US2] [US3] Verify shared package, buyer app, and seller app frontend checks
  - File: `../hivespace.web`
  - Run: the narrowest shared/buyer/seller checks that cover the touched workspaces, including any required shared package checks plus buyer and seller lint, type-check, store tests, and sign-in page tests
  - Keep the command set aligned with the target repo scripts; do not widen scope to unrelated admin or demo workspaces unless dependency wiring requires it.
  - Acceptance: touched frontend workspaces pass their relevant checks or any pre-existing baseline failure is clearly isolated from OTP changes.

## End-to-End and Consistency

### Verify

- [ ] V004 [US1] [US2] [US3] User-owned E2E: Verify the manual OTP sign-in flow through ApiGateway and both frontend apps
  - File: manual validation against `../hivespace.microservice` AppHost plus buyer and seller dev servers
  - Validate buyer flow: OTP entry point is visible, request returns generic success, code-entry route is guarded, valid code redirects to intended destination, invalid/expired/max-attempt states are handled
  - Validate seller flow: same coverage with seller redirects and seller sign-in entry
  - Validate resend flow: resend unlocks only after cooldown and prior code no longer works after resend
  - Acceptance: the user confirms all three stories pass independently in manual testing for both buyer and seller surfaces.

- [ ] V005 Verify final spec, docs, and catalog consistency
  - File: `specs/0009-signin-otp-email/*`, `shared/api-catalog.md`, `shared/event-catalog.md`, `services/identity-service/*`, `services/notification-service/*`
  - Search for stale event names, stale endpoint names, contradictory route/module wording, and any drift between `spec.md`, `plan.md`, ADRs, service docs, and generated tasks.
  - Confirm no task or documentation artifact requires changes in `../hivespace.config`.
  - Acceptance: planning artifacts, service docs, and shared catalogs all describe one consistent OTP sign-in design.
