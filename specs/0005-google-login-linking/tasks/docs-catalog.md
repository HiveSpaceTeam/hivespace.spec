# Docs And Catalog Tasks

## IdentityService Docs

### Update

- [ ] D001 [US1] [US2] [US4] Update IdentityService feature docs
  - File: `services/identity-service/README.md`, `services/identity-service/api.md`, and `services/identity-service/data-and-events.md`
  - Document Google external login/linking ownership, buyer/seller-only scope, pending-link password confirmation, email verification after successful link, normal-user creation from seller app Google start, and removal of legacy `Pages/` plus `AccountCompatibilityEndpoints`.
  - Mention reused `IdentityUserCreatedIntegrationEvent` and optional unchanged `UserEmailVerifiedIntegrationEvent` behavior without introducing new event contracts.
  - Do not update UserService docs for reused profile creation unless implementation changes UserService behavior.
  - Acceptance: IdentityService docs match implemented endpoints, data ownership, events, and legacy page removal behavior.

## API Catalog

### Verify

- [ ] D002 [US1] [US2] [US4] Verify Google account API catalog rows
  - File: `shared/api-catalog.md`
  - Confirm rows exist and match implementation for `GET /api/v1/accounts/external/google/challenge`, `GET /api/v1/accounts/external/google/complete`, `POST /api/v1/accounts/external/google/link`, and `DELETE /api/v1/accounts/external/google/link`.
  - Update only if method, path, auth, owner, request/response behavior, or purpose changes during implementation.
  - Do not duplicate the same endpoint under `/identity/**` or `/Account/**`.
  - Acceptance: API catalog is synchronized with the shipped public browser-facing account contract.

## Event Catalog

### Verify

- [ ] D003 Verify event catalog remains unchanged for this feature
  - File: `shared/event-catalog.md`
  - Confirm no new cross-service command, event, saga message, failure event, timeout event, projection event, or consumer-set change was introduced.
  - Confirm `IdentityUserCreatedIntegrationEvent` is reused unchanged for new Google accounts and `UserEmailVerifiedIntegrationEvent` semantics are unchanged if reused.
  - Do not add catalog rows for in-process IdentityService handlers or frontend-only outcomes.
  - Acceptance: event catalog either remains unchanged or documents only actual implemented contract changes.

## Architecture Decisions

### Update

- [ ] D004 Update ADR-0004 after implementation choices settle
  - File: `architecture/decisions/ADR-0004-google-account-linking.md`
  - Change status from Draft to Accepted when implementation follows the decision, or revise consequences if the final implementation differs.
  - Ensure follow-up bullets reflect completed API catalog and IdentityService doc work.
  - Do not create a second ADR for the same Google account-linking decision unless service boundaries or architecture change beyond ADR-0004.
  - Acceptance: ADR-0004 accurately records final decision state and remaining risks.

## Supporting Service Docs

### Verify

- [ ] D005 Verify UserService remains reused supporting service
  - File: `services/user-service/README.md`, `services/user-service/api.md`, and `services/user-service/data-and-events.md`
  - Confirm implementation did not change UserService profile creation, store onboarding, APIs, validation, ownership, or events.
  - Do not edit UserService docs if it only consumes existing `IdentityUserCreatedIntegrationEvent` behavior unchanged.
  - Acceptance: no unnecessary UserService doc/catalog churn is introduced.

- [ ] D006 Verify ApiGateway docs remain thin routing docs
  - File: `services/api-gateway/README.md` and `services/api-gateway/api.md`
  - Confirm gateway only routes `/api/v1/accounts/**` to IdentityService and does not own Google auth behavior.
  - Update only if the route table or public route ownership changed.
  - Do not describe account-link business rules as ApiGateway responsibilities.
  - Acceptance: gateway docs accurately describe route ownership without duplicating IdentityService behavior.
