# Implementation Plan: Cookie Auth Migration

**Branch:** `0004-cookie-auth-migration`
**Date:** 2026-05-25
**Spec:** [spec.md](spec.md)

> Planning context read: `.specify/memory/constitution.md`, `architecture/overview.md`, `services/_inventory.md`, `shared/api-catalog.md`, `shared/event-catalog.md`, `shared/coding-conventions.md`, `shared/glossary.md`, affected service docs, and current auth/gateway/frontend source shape in the backend and frontend repos.

---

## Phase 0 - Research

### Existing context

- [x] `services/identity-service/README.md`, `api.md`, and `data-and-events.md` read for account/session ownership, existing `/api/v1/accounts/**` routes, and `IdentityUserCreatedIntegrationEvent`.
- [x] `services/api-gateway/README.md` and `api.md` read for route ownership and YARP responsibilities.
- [x] `services/user-service/README.md`, `api.md`, and `data-and-events.md` read for profile creation and seller access propagation context.
- [x] `shared/event-catalog.md` checked. Existing identity/user events are reused unchanged.
- [x] `shared/api-catalog.md` checked. `/api/v1/accounts/**` already belongs to IdentityService through ApiGateway, but login, registration, session refresh, and logout are new public endpoint contracts.
- [x] Current frontend auth source inspected under `../hivespace.web`: shared `useAuth`, shared `ApiService`, app router guards, callback pages, refresh services, and app configs currently depend on `oidc-client-ts`, redirect callbacks, and script-readable token storage.
- [x] Current backend source inspected under `../hivespace.microservice`: IdentityService still owns Razor Page login/register/logout flows, and ApiGateway is a thin YARP proxy without cookie-to-bearer or CSRF middleware.

### Technical unknowns

None unresolved. Planning decisions are captured in [research.md](research.md).

### Research notes

See [research.md](research.md). Key decisions:

- Use IdentityService-owned REST auth endpoints behind existing gateway `/api/v1/accounts/**` routing.
- Use an encrypted HttpOnly browser session cookie issued by IdentityService and readable by ApiGateway through shared ASP.NET Core Data Protection keys.
- Keep access and refresh tokens out of browser-readable storage; frontend stores only derived auth state and a server-issued CSRF token.
- Enforce CSRF in ApiGateway for cookie-authenticated state-changing browser requests before downstream forwarding.
- Keep existing cross-service profile creation and seller role propagation events unchanged.

---

## Phase 1 - Architecture & Data Model

### Service placement

IdentityService owns authentication, account lifecycle, token/session issuance, refresh, logout, and old account-page redirects. ApiGateway owns external routing plus cookie-session mediation for browser requests because downstream services must continue receiving bearer authorization context and must not read browser cookies directly. Frontend apps own the user-facing login/register/logout screens and route behavior.

| Service or app | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| IdentityService | Owning service | Adds browser auth REST endpoints, issues/refreshes/clears secure session and CSRF cookies, validates password/account status/app access, preserves account creation and email verification ownership, redirects old account UI entry points. | Update service API docs and API catalog for new `/api/v1/accounts/**` endpoints during implementation; no event catalog change. |
| ApiGateway | Changed supporting service | Adds cookie-to-bearer forwarding, CSRF validation, CORS credential behavior, and route behavior for account/session endpoints while keeping gateway business-logic-free. | Update gateway docs for auth mediation and API catalog notes; no new route prefix. |
| UserService | Reused supporting service | Continues consuming `IdentityUserCreatedIntegrationEvent` to create profiles and publishing `StoreCreatedIntegrationEvent` for seller role propagation. | Verification-only. Reused contracts unchanged; no UserService doc or catalog update required. |
| NotificationService | Reused supporting service | Continues existing email-verification delivery from identity-owned events. | Verification-only. Reused contracts unchanged; no catalog update required. |
| Admin app | Changed frontend client | Replaces OIDC redirect login with frontend-owned sign-in, session refresh, logout, route guards, and no token storage. Registration remains admin-management only, not public self-registration. | Update app auth pages/routes/i18n and shared auth integration. |
| Seller app | Changed frontend client | Replaces OIDC redirect login with frontend-owned sign-in, buyer/seller candidate registration where applicable, session refresh, logout, and seller access refresh behavior after store registration/email verification. | Update app auth pages/routes/i18n and shared auth integration. |
| Buyer app | Changed frontend client | Replaces OIDC redirect login/register with frontend-owned forms, shared session refresh, logout, and protected workflow handling. | Update app auth pages/routes/i18n and shared auth integration. |
| `@hivespace/shared` | Changed frontend support package | Replaces shared OIDC composable/token injection with cookie-session auth service, CSRF header support, credentials-enabled requests, app role helpers, and notification hub auth changes. | Update shared types/services/stores/i18n as implementation scope. |

### New aggregates

None. This feature does not add a new domain aggregate or service-owned business table. Identity accounts remain ASP.NET Identity users owned by IdentityService, and UserService profiles remain created through the existing event flow.

### New data repository interfaces

None planned. IdentityService may add application services for browser auth orchestration and protected-cookie/session issuance, but no new repository interface is required unless source implementation needs a dedicated persisted session store later.

### Session and contract model

See [data-model.md](data-model.md) for the browser session, session cookie envelope, CSRF token, frontend auth form, session summary, and authorization context models.

### Integration events

No new integration event is introduced and no existing message contract, owner, semantics, or consumer set changes.

| Event | Status | Producer | Consumer(s) | Catalog action |
| ----- | ------ | -------- | ----------- | -------------- |
| `IdentityUserCreatedIntegrationEvent` | Reused unchanged | IdentityService | UserService | No event catalog update required. |
| `UserEmailVerificationRequestedIntegrationEvent` | Reused unchanged | IdentityService | NotificationService | No event catalog update required. |
| `UserEmailVerifiedIntegrationEvent` | Reused unchanged | IdentityService | NotificationService or projections where present | No event catalog update required. |
| `StoreCreatedIntegrationEvent` | Reused unchanged | UserService | IdentityService, CatalogService, OrderService, NotificationService | No event catalog update required. |

### API endpoints

The feature adds public browser auth endpoints under the existing IdentityService-owned `/api/v1/accounts/**` gateway prefix. Detailed contracts are in [contracts/browser-auth-api.md](contracts/browser-auth-api.md).

| Method | Path | Auth | Owner | Purpose |
| ------ | ---- | ---- | ----- | ------- |
| POST | `/api/v1/accounts/login` | Anonymous | IdentityService | Validate email/password and app context, issue HttpOnly session cookie plus CSRF token, return session summary. |
| POST | `/api/v1/accounts/register` | Anonymous | IdentityService | Create identity account, publish existing profile-creation event, issue session cookie plus CSRF token, return session summary. |
| POST | `/api/v1/accounts/session/refresh` | Session cookie + CSRF | IdentityService | Bootstrap after reload or refresh/rotate the browser session before expiry, then return updated session summary. |
| POST | `/api/v1/accounts/logout` | Session cookie + CSRF | IdentityService | Revoke/clear browser session and CSRF cookie. |

Existing email verification endpoints remain unchanged and continue to appear in the API catalog. `shared/api-catalog.md` is updated in this planning change because these are new public endpoint contracts.

### Cookie and CSRF approach

- IdentityService sets `__Host-HiveSpace.Auth` as `HttpOnly`, `Secure`, `SameSite=None`, `Path=/`, with no JavaScript-readable access or refresh token.
- IdentityService sets or returns a signed `HiveSpace.Csrf` token that frontend code sends as `X-HiveSpace-CSRF` on cookie-authenticated state-changing requests.
- ApiGateway validates CSRF before forwarding state-changing requests that include the browser session cookie, except bootstrap anonymous login/register requests.
- ApiGateway strips browser auth/CSRF cookies from downstream service requests and forwards `Authorization: Bearer <access-token>` plus existing correlation headers.
- Frontend HTTP clients use `withCredentials: true` and stop adding bearer tokens from browser storage.

### Frontend gateway URL and refresh trigger

- Local ApiGateway may run at `http://localhost:5000` or `https://localhost:7001`; frontend apps must use `VITE_GATEWAY_BASE_URL` to select the active gateway origin. For HTTP local verification, set `VITE_GATEWAY_BASE_URL=http://localhost:5000`.
- The frontend does not discover `/api/v1/accounts/session/refresh` dynamically. The new shared browser auth service owns this fixed account-route contract and builds it from the configured gateway base URL.
- Existing app startup and router-guard calls to shared auth state, currently `initializeAuth(...)` plus `getCurrentUser()`, become the integration point for session bootstrap. The migrated shared auth implementation calls `POST /api/v1/accounts/session/refresh` after reload when the auth cookie and CSRF token are present, before near-expiry protected navigation, after email verification/store-role propagation, and after a protected request receives an unauthenticated response.
- If refresh fails, shared auth clears derived frontend state and each app routes to its sign-in page.

### Old IdentityService account URLs

Delete the IdentityService Razor Page UI for HiveSpace login, registration, and logout instead of keeping page handlers. Add minimal compatibility routes or middleware for `/Account/Login`, `/Account/Register`, and `/Account/Logout` that redirect to safe frontend routes using `returnUrl`, app/client hint, or configured fallback. These routes must not render user-facing IdentityService UI. Direct IdentityServer protocol endpoints such as `/.well-known/**` and `/connect/**` remain direct authority endpoints and are not routed through ApiGateway.

---

## Phase 2 - Implementation Plan

### Backend shared preparation

- [ ] Read `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md` before backend source changes.
- [ ] Confirm shared Data Protection key-ring location/configuration is available to IdentityService and ApiGateway in local and deployed environments.
- [ ] Confirm downstream JWT validation settings still accept the access tokens forwarded by ApiGateway.

### IdentityService application/core

- [ ] Add account session request/response DTOs for login, registration, and session summary; do not add a custom browser auth error DTO.
- [ ] Extend `IdentityDomainErrorCode` for session-auth failures and return errors through existing `BadRequestException`, `UnauthorizedException`, `ForbiddenException`, `ConflictException`, and `ExceptionModel` handling.
- [ ] Move existing password sign-in behavior into an application account-session service before deleting Razor UI code: email lookup, account active/locked/not allowed checks, role/app access checks, lockout-on-failure, `LastLoginAt`, and safe error mapping.
- [ ] Move existing registration orchestration into an application account-session service before deleting Razor UI code: unique email, password validation, identity account creation, `IdentityUserCreatedIntegrationEvent` publication, and initial sign-in/session issuance.
- [ ] Implement session refresh with account-status/security-stamp validation, refresh-token rotation where applicable, and updated access-token claims after seller/email verification changes.
- [ ] Implement logout with cookie clearing and refresh-token/session invalidation.
- [ ] Add CSRF token issuance/rotation tied to the protected browser session.

### IdentityService API

- [ ] Add Minimal API endpoints in `IdentityEndpoints.cs` or a feature-specific endpoint file for `/api/v1/accounts/login`, `/register`, `/session/refresh`, and `/logout`.
- [ ] Preserve existing `/api/v1/accounts/email-verification` and `/api/v1/accounts/email-verification/verify` contracts.
- [ ] Keep login/register endpoints anonymous but rate-limitable and safe-error-only.
- [ ] Require session cookie plus CSRF for refresh and logout.
- [ ] Delete IdentityService Razor Page UI files for HiveSpace login, registration, and logout after the shared behavior is moved to REST application services.
- [ ] Add minimal `/Account/Login`, `/Account/Register`, and `/Account/Logout` compatibility routes or middleware that safe-redirect to frontend routes; do not keep Razor Page handlers for these UI flows.

### ApiGateway

- [ ] Add shared Data Protection configuration matching IdentityService.
- [ ] Add session forwarding middleware/transform before `MapReverseProxy()` to decrypt/validate `__Host-HiveSpace.Auth`, extract the access token, set `Authorization: Bearer ...`, and strip browser session cookies before forwarding to downstream services.
- [ ] Add CSRF middleware before proxy forwarding for cookie-authenticated `POST`, `PUT`, `PATCH`, and `DELETE` requests, using `X-HiveSpace-CSRF`.
- [ ] Leave anonymous `POST /api/v1/accounts/login` and `POST /api/v1/accounts/register` exempt from pre-existing-session CSRF requirements.
- [ ] Preserve CORS `AllowCredentials()` with configured frontend origins and ensure all three app origins are present.
- [ ] Verify SignalR notification hub negotiation/WebSocket requests work with cookie credentials and forwarded bearer context.

### Frontend shared package

- [ ] Read `../hivespace.web/AGENTS.md` and `../hivespace.web/CLAUDE.md` before frontend source changes.
- [ ] Replace `oidc-client-ts` redirect auth behavior in shared auth code with a cookie-session auth service/store using `/api/v1/accounts/**`.
- [ ] Extend shared auth initialization so the app-provided gateway base URL drives account-route calls; local HTTP development can use `VITE_GATEWAY_BASE_URL=http://localhost:5000`.
- [ ] Rework `getCurrentUser()` or its replacement to bootstrap by calling `POST /api/v1/accounts/session/refresh` when no in-memory session exists but browser cookies are available, so existing route guards naturally trigger refresh after reload.
- [ ] Remove browser-readable access and refresh token storage and cleanup existing OIDC storage keys on migration.
- [ ] Update shared `ApiService` to use `withCredentials: true`, send `X-HiveSpace-CSRF` for state-changing requests, preserve correlation headers, and stop injecting `Authorization` from `currentUser.access_token`.
- [ ] Update shared notification hub auth to rely on cookie credentials/gateway forwarding rather than `accessTokenFactory`.
- [ ] Add shared types for login/register/session responses and the standard HiveSpace API error response shape.
- [ ] Add shared i18n keys for auth errors, session expiry, and signed-out states where reusable across apps.

### Frontend apps

- [ ] Admin: add frontend-owned sign-in route/page, route guards, session bootstrap/refresh, logout, and admin-role denial handling. Do not add public admin self-registration.
- [ ] Seller: add frontend-owned sign-in and account registration where applicable, preserve email verification and seller registration transition behavior, and refresh the session after verification/store-role propagation.
- [ ] Buyer: add frontend-owned sign-in/register routes/pages, session bootstrap/refresh, logout, and protected checkout/cart/profile handling.
- [ ] Remove normal dependencies on `/callback/login` and `/callback/logout` redirect callbacks from route guards. Keep compatibility routes only as transitional redirects if needed.
- [ ] Update English and Vietnamese i18n resources together for all new user-facing auth UI/error text.

### Documentation and catalog layer

- [ ] Update `services/identity-service/api.md` for the browser auth endpoints and old page redirects.
- [ ] Update `services/api-gateway/api.md` for cookie-session mediation and CSRF gateway behavior.
- [ ] Keep `services/user-service/*` unchanged unless implementation changes the existing profile event contract or profile behavior.
- [ ] Keep `shared/event-catalog.md` unchanged because all integration messages are reused unchanged.
- [x] Update `shared/api-catalog.md` with the planned new account endpoints.

### Verification

- [ ] Backend: run targeted IdentityService and ApiGateway builds/tests after source changes.
- [ ] Frontend: run targeted shared package and app type-check/build commands after source changes.
- [ ] Browser verification: confirm no access/refresh token appears in localStorage or sessionStorage after login/register/refresh.
- [ ] Gateway verification: confirm protected API calls include downstream `Authorization: Bearer` and no browser auth cookie reaches downstream services.
- [ ] CSRF verification: confirm cookie-authenticated state-changing requests without `X-HiveSpace-CSRF` are rejected at the gateway before downstream state changes.
- [ ] Old URL verification: confirm `/Account/Login`, `/Account/Register`, and `/Account/Logout` redirect to frontend routes through minimal compatibility routes and no Razor UI exists for those flows.

### Saga design

No saga design artifact is required. This feature changes synchronous browser authentication, gateway forwarding, and existing REST/session behavior. It does not introduce or change a MassTransit saga state machine, compensation path, timeout, or async workflow orchestration.

### Architecture decision

ADR required because the feature changes cross-repo browser auth architecture and chooses gateway cookie-to-bearer mediation over SPA-held tokens:

- [ADR-0003: Gateway-Mediated Cookie Browser Sessions](../../architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md)

---

## Phase 3 - Frontend Plan

### Surface(s)

- [x] buyer
- [x] seller
- [x] admin
- [x] `@hivespace/shared`

### Files to create or update (mandatory order: types -> service -> store -> components -> views -> routes -> i18n)

| Area | File or pattern | Notes |
| ---- | --------------- | ----- |
| Shared types | `packages/shared/src/types/auth-session.ts` | Login/register/session request and response types, standard API error response types, app context. |
| Shared service | `packages/shared/src/features/auth/account-session.service.ts` | Calls `/api/v1/accounts/login`, `/register`, `/session/refresh`, and `/logout` with credentials. |
| Shared store/composable | `packages/shared/src/composables/useAuth.ts` or new setup store | Replace OIDC redirect methods with session bootstrap, login, register, refresh, logout, role helpers, and cleanup of old token storage. |
| Shared HTTP | `packages/shared/src/utils/api.ts` | Add `withCredentials`, CSRF header injection, 401 session handling, no bearer token from web storage. |
| Shared realtime | `packages/shared/src/composables/useNotificationHub.ts`, `packages/shared/src/features/notifications/useNotificationRealtime.ts` | Remove token access factory and use gateway cookie credentials/forwarding. |
| Admin app | `apps/admin/src/pages/Auth/*`, `router/index.ts`, `services/api.ts` | Sign-in page, route guard refresh/bootstrap, no public registration. |
| Seller app | `apps/seller/src/pages/Auth/*`, `router/index.ts`, `services/api.ts` | Sign-in/register page flow, seller role refresh after store registration and email verification. |
| Buyer app | `apps/buyer/src/pages/Auth/*`, `router/index.ts`, `components/layout/AppHeader.vue`, `services/api.ts` | Sign-in/register pages and header actions wired to frontend forms. |
| i18n | `apps/*/src/i18n/locales/{en,vi}/*.json`, shared common/errors where applicable | Add auth form labels, validation, safe errors, session expiry, logout copy in both languages. |

### i18n keys to add (both English and Vietnamese)

```json
{
  "auth": {
    "signIn": {
      "title": "",
      "email": "",
      "password": "",
      "submit": ""
    },
    "register": {
      "title": "",
      "fullName": "",
      "email": "",
      "password": "",
      "confirmPassword": "",
      "submit": ""
    },
    "errors": {
      "invalidCredentials": "",
      "accountLocked": "",
      "accountInactive": "",
      "appAccessDenied": "",
      "duplicateEmail": "",
      "validationFailed": "",
      "sessionExpired": ""
    },
    "actions": {
      "signOut": "",
      "createAccount": "",
      "backToSignIn": ""
    }
  }
}
```

---

## Constitution Compliance Check

- [x] No hard-deletes planned.
- [x] No new money values planned.
- [x] No new persisted domain IDs planned.
- [x] Existing integration events continue through the transactional outbox; no new integration messages are planned.
- [x] No NuGet dependency versioning in `.csproj` is planned. Any needed package version belongs in `Directory.Packages.props`.
- [x] Both English and Vietnamese frontend i18n resources will be updated.
- [x] Frontend text will use translation keys only.
- [x] Public endpoint additions are recorded in `shared/api-catalog.md`.
- [x] Reused common contracts are explicitly unchanged and require no event catalog update.
- [x] Service boundaries remain intact: IdentityService owns credentials/session issuance, ApiGateway owns forwarding/CSRF mediation, downstream services own authorization decisions from forwarded claims, and UserService owns profile/store data.

Post-design check: no constitution violations remain. The primary trade-off is that ApiGateway now performs security mediation in addition to route forwarding, but it remains boundary-safe because it does not own account state or business decisions and only translates browser session proof into the existing downstream bearer authorization context.
