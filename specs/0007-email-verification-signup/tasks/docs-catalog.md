# Documentation And Catalog Tasks

## Documentation And Catalog Scope

- Editable docs for this feature:
  - `services/identity-service/README.md`
  - `services/identity-service/api.md`
  - `services/identity-service/data-and-events.md`
  - `services/user-service/README.md`
  - `services/user-service/data-and-events.md`
  - `shared/api-catalog.md`
  - `shared/event-catalog.md`
- Verification-only context for this feature:
  - `services/notification-service/**`
  - `services/api-gateway/**`
  - Existing shared catalog rows that are reused unchanged for `UserEmailVerificationRequestedIntegrationEvent` and authenticated `POST /api/v1/accounts/email-verification`

## IdentityService Docs And API Catalog

### Update

- [ ] D001 [US1][US3] Update `shared/api-catalog.md`
  - File: `shared/api-catalog.md`, `services/identity-service/api.md`, `services/identity-service/README.md`
  - Update the register, login, resend, and verify entries so they document pending registration, no registration session issuance, anonymous resend, `204 No Content` success for resend/verify, and exception-response conventions for pending-account failures.
  - Add the new `/api/v1/accounts/email-verification/resend` public endpoint and remove any wording that still says registration creates a browser session immediately.
  - Do not rewrite Google-auth or authenticated `POST /api/v1/accounts/email-verification` rows beyond context needed to keep this feature accurate.
  - Acceptance: IdentityService docs and the shared API catalog describe the shipped public account contract without contradicting the feature plan or quickstart.

## UserService And Event Catalog

### Update

- [ ] D002 [US2] Update `shared/event-catalog.md`
  - File: `shared/event-catalog.md`, `services/identity-service/data-and-events.md`, `services/identity-service/README.md`, `services/user-service/data-and-events.md`, `services/user-service/README.md`
  - Add `IdentityUserReadyIntegrationEvent`, remove `IdentityUserCreatedIntegrationEvent` from the downstream provisioning story, and document that UserService now provisions profiles from readiness instead of raw account creation.
  - Keep `UserEmailVerificationRequestedIntegrationEvent` described as the reused delivery contract and only mention `UserEmailVerifiedIntegrationEvent` where it remains a separate identity-owned fact.
  - Do not add NotificationService doc edits for reused delivery behavior that did not change.
  - Acceptance: service docs and the shared event catalog show one provisioning trigger, clear producer/consumer ownership, and no stale references to provisioning on raw account creation.
