# Implementation Plan: Google Login and Account Linking

**Branch:** `0005-google-login-linking`  
**Date:** 2026-06-01  
**Spec:** [spec.md](spec.md)

> Planning source of truth: `.specify/memory/constitution.md`

---

## Phase 0 - Research

### Existing context read before planning

- [x] `.specify/memory/constitution.md`
- [x] `architecture/overview.md`
- [x] `services/_inventory.md`
- [x] `shared/api-catalog.md`
- [x] `shared/event-catalog.md`
- [x] `shared/coding-conventions.md`
- [x] `shared/glossary.md`
- [x] `services/identity-service/README.md`
- [x] `services/user-service/README.md`
- [x] `../hivespace.microservice/AGENTS.md`
- [x] `../hivespace.web/AGENTS.md`
- [x] Existing IdentityService source shape for `Pages/`, `AccountCompatibilityEndpoints`, Razor registration, and account endpoint mapping
- [x] Existing buyer/seller auth page source shape for sign-in, sign-up, shared auth composables, and i18n

### Technical unknowns

- None unresolved. Research decisions are captured in [research.md](research.md).

### Research decisions

- Google browser authentication belongs under IdentityService `/api/v1/accounts/external/google/**` endpoints.
- Durable Google links use ASP.NET Identity external login storage, not a custom table.
- Existing local email/password accounts require explicit consent plus password confirmation before Google is linked. If a verified Google email matches an existing unlinked local password account, the user must link or use password sign-in, password reset, or another Google account; the system must not create a duplicate Google-only account for that email. If no same-email local password account exists, the callback follows the normal Google sign-in/sign-up path instead of showing a link prompt.
- New Google-authenticated users are normal user accounts, even when started from the seller app.
- Google must be available from buyer and seller sign-in and sign-up pages, because both pages are entry points to the same external authentication challenge.
- IdentityService legacy hosted user-facing pages are removed from the implementation target. In source this means deleting `HiveSpace.IdentityService.Api/Pages/`, removing Razor Pages registration/mapping, and deleting `Endpoints/AccountCompatibilityEndpoints.cs`.
- No MassTransit saga is required.
- ADR-0004 records the security-sensitive account-linking decision.

---

## Phase 1 - Architecture & Data Model

### Service placement

IdentityService owns the feature because it owns accounts, credentials, external logins, account status, lockout, email verification, IdentityServer/OIDC state, token cookie issuance, and browser auth endpoints. Frontend apps consume IdentityService account endpoints through ApiGateway. UserService is reused only through the existing account-created event and existing seller onboarding flow.

| Service / App | Classification | Reason | Documentation/catalog action |
| --- | --- | --- | --- |
| IdentityService | Owning service | Adds Google challenge/callback/link endpoints, stores Google external login links, creates normal user accounts from Google only when no same-email local password account exists, blocks duplicate same-email Google-only identities, marks email verified after successful linking, enforces account status/lockout, and removes legacy hosted account UI. | Update `services/identity-service/*` and `shared/api-catalog.md`; no new event catalog row. |
| ApiGateway | Changed supporting service | Browser-facing Google endpoints must route under existing `/api/v1/accounts/**`; no business logic. | Verify existing account route covers new endpoints or update gateway config if needed. |
| UserService | Reused supporting service | Existing `IdentityUserCreatedIntegrationEvent` creates profiles for new Google users; existing store onboarding grants seller access. No UserService behavior change. | Read for context only; no UserService docs/catalog update beyond already stated reuse. |
| NotificationService | Reused supporting service | Existing email verification notification/event behavior may be reused if linking publishes `UserEmailVerifiedIntegrationEvent`; no new delivery contract. | No docs/catalog update unless implementation changes the existing event semantics. |
| Buyer app | Changed frontend client | Buyer sign-in and sign-up pages start Google challenge and handle link/cancel/error return paths. | Update buyer auth pages, auth service/store/composable usage, routes if link-confirmation page is needed, and `en`/`vi` auth copy. |
| Seller app | Changed frontend client | Seller sign-in and sign-up pages start Google challenge; new Google users remain normal users and continue through existing seller guards/onboarding. | Update seller auth pages, auth service/store/composable usage, routes if link-confirmation page is needed, and `en`/`vi` auth copy. |
| Admin app | Reused frontend client | Admin sign-in remains supported without Google. | Verification only: no Google button and no admin Google app context. |

### Data model

See [data-model.md](data-model.md). No new business aggregate is introduced.

Existing IdentityService data reused:

- `ApplicationUser` for account email, status, lockout, roles, and `EmailConfirmed`.
- ASP.NET Identity external login storage (`AspNetUserLogins`) for `LoginProvider = Google` and Google provider key.
- Short-lived pending Google link state stored server-side or in HttpOnly/link-state cookies so provider keys and tokens are never exposed to browser scripts.
- Existing browser session cookies and CSRF token state.

No EF migration is planned unless the implementation lacks required Identity external login tables or needs storage for temporary server-side link state.

### Integration events

| Event | Producer | Consumer(s) | Plan |
| --- | --- | --- | --- |
| `IdentityUserCreatedIntegrationEvent` | IdentityService | UserService | Reused unchanged when Google creates a new normal user account. No event catalog update required. |
| `UserEmailVerifiedIntegrationEvent` | IdentityService | Existing consumers | Reused unchanged only if the current email-verification path publishes it when linking marks `EmailConfirmed = true`. No new event catalog row. |

No new cross-service command, event, saga message, failure event, timeout event, or projection event is planned.

### API endpoints

See [contracts/browser-auth-api.md](contracts/browser-auth-api.md).

| Method | Path | Auth | Handler / behavior |
| --- | --- | --- | --- |
| GET | `/api/v1/accounts/external/google/challenge` | Anonymous | Start Google for `app=buyer` or `app=seller`, preserving safe return URL and culture. Used by both sign-in and sign-up pages. |
| GET | `/api/v1/accounts/external/google/complete` | Anonymous | Complete Google callback; sign in linked users, create normal users only when no same-email local password account exists, or redirect to frontend account-link confirmation for same-email unlinked local password accounts. Same-email duplicate Google-only account creation is forbidden. |
| POST | `/api/v1/accounts/external/google/link` | Temporary Google link state + CSRF/link token | Confirm consent and password, add Google login, mark verified matching email, issue normal session cookies. |
| DELETE | `/api/v1/accounts/external/google/link` | Temporary Google link state + CSRF/link token | Cancel pending Google link and clear temporary state. |

Existing session endpoints are reused unchanged:

- `POST /api/v1/accounts/login`
- `POST /api/v1/accounts/register`
- `POST /api/v1/accounts/session/refresh`
- `POST /api/v1/accounts/logout`

Shared API catalog entries for the Google endpoints already exist in `shared/api-catalog.md`; keep them unchanged unless implementation changes path, method, auth, owner, request/response contract, or purpose.

---

## Phase 2 - Backend Implementation Plan

Layer order remains domain/application/infrastructure/api where applicable. IdentityService is a LiteService using CQRS-style handlers plus ASP.NET Identity infrastructure.

### IdentityService application/infrastructure

- [ ] Add Google provider configuration validation for client ID, client secret, callback URL, and allowed buyer/seller frontend origins.
- [ ] Add Google external-auth commands/handlers or cohesive account feature handlers for challenge, callback completion, link confirmation, and link cancellation.
- [ ] Reuse ASP.NET Identity `UserManager`/`SignInManager` external-login APIs for durable Google links.
- [ ] Enforce buyer/seller-only app context. Reject `admin` for challenge, callback, new account creation, and linking.
- [ ] On verified Google email with no same-email local password account, continue the normal Google path: sign in an already linked Google account or create a normal user account and reuse the existing account-created publication path.
- [ ] On existing same-email unlinked local password account match, create short-lived pending-link state and redirect to the frontend link-confirmation route. Do not create or sign in a separate Google-only HiveSpace account for that email.
- [ ] On link confirmation, require consent in the frontend flow, validate password with existing lockout/status behavior, add the Google login, mark matching email verified, clear pending state, and issue standard browser session cookies plus CSRF token.
- [ ] Ensure provider access tokens, refresh tokens, and provider user IDs are never returned in JSON or readable browser state.

### IdentityService API cleanup

- [ ] Delete `src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/` entirely.
- [ ] Delete `src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/AccountCompatibilityEndpoints.cs`.
- [ ] Remove `MapAccountCompatibilityEndpoints()` from `Extensions/HostingExtensions.cs`.
- [ ] Remove `MapRazorPages().RequireAuthorization()` from `Extensions/HostingExtensions.cs`.
- [ ] Remove `AddRazorPages()` from `Extensions/ServiceCollectionExtensions.cs`.
- [ ] Remove Razor Pages-only static file, MVC, suppressions, namespace, or package references if they become unused after deleting `Pages/`.
- [ ] Preserve required IdentityServer protocol endpoints under `/.well-known/**` and `/connect/**`.
- [ ] Known old `/Account/Login`, `/Account/Register`, `/Account/Logout`, consent, grants, device, CIBA, diagnostics, and external-login page URLs must not render legacy HiveSpace user-facing pages. Compatibility redirects are intentionally removed during development.

### ApiGateway / source runtime config

- [ ] Verify the existing `/api/v1/accounts/**` route forwards Google endpoints to IdentityService.
- [ ] Add committed Google OAuth config shape only in backend source appsettings/runtime config, and keep real Google client secrets in local-only unpushed backend appsettings or .NET user-secrets.
- [ ] Add frontend redirect/origin settings only in backend/frontend source repo runtime config that is part of implementation scope; frontend environment files must not contain OAuth client secrets.

### Backend verification

- [ ] Build IdentityService:

  ```bash
  dotnet build src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj
  ```

- [ ] Build ApiGateway if route/config code changes:

  ```bash
  dotnet build src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj
  ```

- [ ] Manually verify new Google account creation, existing account linking, already-linked sign-in, blocked-account rejection, missing/unverified Google email rejection, and old page URL removal.

### Saga design

No saga required. The feature is an IdentityService-local browser authentication/linking flow. Cross-service profile creation reuses `IdentityUserCreatedIntegrationEvent` and does not add a new MassTransit state machine, compensation, or timeout workflow.

### Architecture decision

ADR required and already generated: [ADR-0004: Google Account Linking](../../architecture/decisions/ADR-0004-google-account-linking.md).

---

## Phase 3 - Frontend Plan

### Surfaces

- [x] buyer
- [x] seller
- [x] admin verification only

### Shared-first implementation order

Before app-local changes, inspect `../hivespace.web/packages/shared/src/features/auth`, `../hivespace.web/packages/shared/src/composables`, `../hivespace.web/packages/shared/src/icons`, and existing `useAuth` exports. Reuse the existing `GoogleIcon`.

| Order | Area | Files / notes |
| --- | --- | --- |
| 1 | Types | Extend shared/app auth types with Google challenge params and link request/response if not already present. |
| 2 | Service | Add shared or app auth service helpers to build/start `/api/v1/accounts/external/google/challenge?app=...&returnUrl=...&culture=...`. |
| 3 | Store/composable | Extend `useAuth` or account stores with `startGoogleAuth(app, returnUrl, culture)` and link/cancel actions. |
| 4 | Components | Create or reuse a Google auth button component using `GoogleIcon`; keep provider button behavior shared if buyer and seller need the same UI. |
| 5 | Pages | Add Google button to `apps/buyer/src/pages/Auth/SignInPage.vue`, `apps/buyer/src/pages/Auth/SignUpPage.vue`, `apps/seller/src/pages/Auth/SignInPage.vue`, and `apps/seller/src/pages/Auth/SignUpPage.vue`. Add account-link confirmation page/component if not already present. |
| 6 | Routes | Register buyer/seller account-link confirmation routes for Google pending-link redirects. Admin routes remain unchanged. |
| 7 | i18n | Update buyer and seller `auth.json` in both `en` and `vi`; add clear copy for Google sign-in, Google sign-up, link confirmation, password confirmation, cancellation, and safe error outcomes. |

### Frontend behavior

- Both sign-in and sign-up pages in buyer and seller apps start the same Google challenge endpoint. The page copy may say "Continue with Google" or equivalent, but the backend behavior is selected by Google callback result, not by a separate sign-up endpoint.
- Preserve `returnUrl` from the current route query using existing safe redirect helpers.
- Redirect Google failures, cancellation, declined link, wrong password, and blocked account states to localized user-facing outcomes. Declined or failed same-email linking must offer password sign-in, password reset, or choosing another Google account, not continuing as a separate same-email Google account.
- Seller app must not grant seller access merely because Google started from seller sign-up. Existing seller guards route normal users through email verification/store onboarding as currently designed.
- Admin app must not render Google sign-in or start a Google challenge.

### Frontend verification

- [ ] Buyer: lint/type-check or narrow build per repo instructions; verify Google button on sign-in and sign-up.
- [ ] Seller: lint/type-check or narrow build per repo instructions; verify Google button on sign-in and sign-up.
- [ ] Admin: verify no Google button and supported admin sign-in still works.
- [ ] Verify English and Vietnamese auth copy are updated together.

---

## Constitution Compliance Check

### Pre-design

- [x] Service ownership stays inside IdentityService for credentials, external logins, status, lockout, email verification, and token issuance.
- [x] UserService is reused through existing events/onboarding only; no cross-service database reads.
- [x] Browser-facing endpoints are cataloged under `/api/v1/accounts/**`.
- [x] New browser responses do not expose access/refresh tokens to scripts.
- [x] No new money values.
- [x] No hard-delete of business entities.
- [x] No new saga where a simple service-local flow is enough.
- [x] Frontend plan uses shared-first auth/icon/composable review and updates `en`/`vi` together.

### Post-design

- [x] `research.md`, `data-model.md`, `contracts/browser-auth-api.md`, and `quickstart.md` exist.
- [x] ADR-0004 exists and is linked from this plan.
- [x] No `saga-design.md` is generated because no MassTransit saga is introduced or changed.
- [x] Reused common contracts are explicitly unchanged: `IdentityUserCreatedIntegrationEvent`, optional existing `UserEmailVerifiedIntegrationEvent`, and existing session endpoints.
- [x] Shared event catalog requires no update for this plan because no new event/command/consumer-set change is introduced.
- [x] Shared API catalog already contains the Google endpoint rows; update only if implementation changes the contract.
- [x] Same verified email plus existing unlinked local password account cannot create a duplicate Google-only account; linking or account recovery is required.
