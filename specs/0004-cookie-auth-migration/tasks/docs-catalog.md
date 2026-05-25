# Docs And Catalog Tasks

## Documentation Scope

Editable docs:

- `services/identity-service/README.md`
- `services/identity-service/api.md`
- `services/identity-service/data-and-events.md` only if wording must clarify unchanged event reuse
- `services/api-gateway/README.md`
- `services/api-gateway/api.md`
- `shared/api-catalog.md`
- `architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md`

Verification-only docs/services:

- `services/user-service/**` because the feature reuses existing profile creation and store role events unchanged.
- `shared/event-catalog.md` because no integration event contract, owner, semantics, or consumer set changes.

## IdentityService Docs

### Update

- [ ] D001 [US1] [US2] [US3] [US4] Update IdentityService API docs
  - File: `services/identity-service/api.md`
  - Add browser auth endpoint rows for login, register, session refresh, and logout with auth requirements from `contracts/browser-auth-api.md`.
  - Change `/Account/**` wording to legacy compatibility redirects for login/register/logout, not interactive IdentityService UI.
  - Preserve direct `/.well-known/**` and `/connect/**` protocol endpoint documentation.
  - Acceptance: IdentityService docs match `shared/api-catalog.md` and no longer describe HiveSpace login/register/logout Razor UI as current target behavior.

- [ ] D002 [US1] [US2] [US4] Update IdentityService responsibility docs
  - File: `services/identity-service/README.md`
  - Clarify that IdentityService owns credentials, account state, session issuance, refresh, logout, and compatibility redirects, but not frontend UI pages.
  - Clarify Razor auth UI is removed for this feature and frontend apps own user-facing sign-in/sign-up.
  - Acceptance: README service boundary remains aligned with constitution and new auth flow.

## ApiGateway Docs

### Update

- [ ] D003 [US3] [US4] Update ApiGateway docs for auth mediation
  - File: `services/api-gateway/README.md`, `services/api-gateway/api.md`
  - Document cookie-to-bearer forwarding, CSRF header validation, stripped browser cookies before downstream forwarding, and unchanged route prefixes.
  - Explicitly state gateway does not own account state or business authorization decisions.
  - Acceptance: ApiGateway docs explain the security mediation behavior needed by ADR-0003.

## Shared Catalogs

### Update

- [ ] D004 [US1] [US2] [US3] [US4] Verify API catalog browser auth entries
  - File: `shared/api-catalog.md`
  - Ensure entries exist for `POST /api/v1/accounts/login`, `POST /api/v1/accounts/register`, `POST /api/v1/accounts/session/refresh`, and `POST /api/v1/accounts/logout`.
  - Ensure `/Account/**` is documented as direct compatibility redirects, not gateway-routed API and not user-facing UI.
  - Do not add `/api/v1/identity/**` or duplicate account endpoints.
  - Acceptance: API catalog matches contract and plan endpoint list.

### Verify

- [ ] D005 Verify event catalog remains unchanged for reused contracts
  - File: `shared/event-catalog.md`
  - Confirm `IdentityUserCreatedIntegrationEvent`, `UserEmailVerificationRequestedIntegrationEvent`, `UserEmailVerifiedIntegrationEvent`, and `StoreCreatedIntegrationEvent` remain existing reused contracts.
  - Do not add `BrowserSessionCreatedIntegrationEvent`, duplicate registration events, or new saga messages.
  - Acceptance: event catalog has no cookie-auth-specific new events and profile creation remains on existing event flow.

- [ ] D006 Verify ADR follow-up is reflected in tasks/docs
  - File: `architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md`, `specs/0004-cookie-auth-migration/tasks/*.md`
  - Confirm tasks implement the ADR decision: IdentityService issues HttpOnly session, ApiGateway mediates cookie-to-bearer and CSRF, frontend avoids web-storage tokens.
  - Update ADR follow-up only if implementation discovers a material decision change.
  - Acceptance: ADR and tasks stay aligned.
