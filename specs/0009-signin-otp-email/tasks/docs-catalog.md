# Docs and Catalog Tasks

## Documentation and Catalog Scope

- Editable service docs: `services/identity-service/*`, `services/notification-service/*`
- Editable shared catalogs: `shared/api-catalog.md`, `shared/event-catalog.md`
- Verification-only supporting service context: ApiGateway existing `/api/v1/accounts/**` route coverage
- ADRs are verification-first for this feature because ADR-0008 and ADR-0009 already exist; update them only if implementation materially departs from the documented decisions

## Service Docs

### Update

- [ ] D001 [US1] Update IdentityService docs for OTP sign-in ownership, routes, and reusable challenge model
  - File: `services/identity-service/README.md`, `services/identity-service/api.md`, `services/identity-service/data-and-events.md`, `services/identity-service/domain.md`
  - Document `OtpChallenge`, `Purpose = SignIn` for v1, `OtpSignInEndpoints`, and the public request/verify routes.
  - Keep the docs scoped to IdentityService-owned auth behavior; do not describe a cross-service OTP platform.
  - Acceptance: IdentityService docs match `plan.md`, `data-model.md`, `contracts/otp-api.md`, and ADR-0009.

- [ ] D002 [US1] Update NotificationService docs for OTP challenge email delivery
  - File: `services/notification-service/README.md`, `services/notification-service/data-and-events.md`
  - Document consumption of `UserOtpChallengeRequestedIntegrationEvent` and the v1 `Purpose = SignIn` guardrail.
  - Do not rewrite unrelated notification workflows or common delivery rules.
  - Acceptance: NotificationService docs reflect only the changed supporting-service behavior introduced by OTP sign-in.

## Shared Catalogs

### Update

- [ ] D003 [US1] Update shared API catalog for final OTP request and verify contracts
  - File: `shared/api-catalog.md`
  - Verify the two OTP endpoint rows match final path, method, auth, response shape, cookie/session semantics, and same-origin redirect behavior.
  - Update only the rows for `/api/v1/accounts/otp/request` and `/api/v1/accounts/otp/verify`; do not rewrite unrelated account endpoints.
  - Acceptance: the API catalog matches the implemented endpoint contract and `contracts/otp-api.md`.

- [ ] D004 [US1] Update shared event catalog for the canonical OTP challenge delivery event
  - File: `shared/event-catalog.md`
  - Verify there is exactly one canonical row for `UserOtpChallengeRequestedIntegrationEvent` with owner `IdentityService`, consumer `NotificationService`, and field notes `RecipientEmail`, `OtpCode`, `ExpiresAt`, `Purpose`.
  - Remove or avoid any duplicate sign-in-only event naming if source implementation never uses it.
  - Acceptance: the event catalog has one correct OTP event entry aligned with the shared messaging contract.

## Architecture Decisions

### Verify

- [ ] D005 [US1] Verify ADR-0008 and ADR-0009 still describe the shipped implementation
  - File: `architecture/decisions/ADR-0008-otp-challenge-persistence.md`, `architecture/decisions/ADR-0009-reusable-auth-otp-challenge-model.md`
  - Confirm the final code and docs still use SQL-backed `OtpChallenge`, IdentityService-only reuse, and sign-in-specific public routes.
  - Update ADR status or follow-up bullets only if implementation meaningfully departs from the current decision record.
  - Acceptance: the ADRs remain accurate explanations of the final OTP architecture choices.
