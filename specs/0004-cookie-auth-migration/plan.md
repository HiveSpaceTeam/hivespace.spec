# Implementation Plan: Cookie Auth Migration

**Branch:** `master`
**Date:** 2026-05-27
**Spec:** [spec.md](spec.md)

> Planning update requested: backend stores token material in HttpOnly cookies without application-level encryption; frontend introduces a shared `AuthLayout` based on the demo sign-in/sign-up pages so pages only provide the left-side form while the right side shares the `CommonGridShape` background and varies center copy/image.

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
- [x] `services/api-gateway/README.md`
- [x] `services/user-service/README.md`
- [x] `../hivespace.microservice/AGENTS.md` and `CLAUDE.md`
- [x] `../hivespace.web/AGENTS.md` and `CLAUDE.md`
- [x] Demo sources: `../hivespace.web/packages/demo/src/Auth/Signin.vue` and `Signup.vue`
- [x] Shared frontend source: `FullScreenLayout`, `CommonGridShape`, shared auth and API conventions

### Technical unknowns

No unresolved `NEEDS CLARIFICATION` items remain for this planning pass.

### Research notes

Research is captured in [research.md](research.md). The main update from the earlier plan is that the browser session is no longer an encrypted Data Protection envelope. IdentityService will issue raw token cookies using normal cookie protections (`HttpOnly`, `Secure`, `SameSite=None`) and the token's own signing/expiry; ApiGateway will read the access-token cookie directly and forward it as a bearer token.

---

## Phase 1 - Architecture & Data Model

### Service placement

IdentityService owns account/session issuance. ApiGateway mediates browser cookies into downstream bearer authorization and CSRF enforcement. UserService remains a reused supporting service for existing profile creation and seller role propagation events. Frontend apps and `@hivespace/shared` own the user-facing sign-in/sign-up UI.

| Service / area | Classification | Reason | Documentation/catalog action |
| --- | --- | --- | --- |
| IdentityService | Owning service | Owns credentials, login/register handlers, token issuance, refresh, logout, old `/Account/**` compatibility redirects, and Identity-owned account/session rules. | Update service docs during catalog/doc task; existing `/api/v1/accounts/**` catalog rows already cover endpoint paths. |
| ApiGateway | Changed supporting service | Adds cookie-to-bearer mediation and gateway CSRF rejection before YARP forwarding. | Update ApiGateway docs during catalog/doc task; no new public route prefix. |
| UserService | Reused supporting service | Existing `IdentityUserCreatedIntegrationEvent` profile creation and `StoreCreatedIntegrationEvent` seller role propagation remain unchanged. | Verification only; do not rewrite UserService docs or event catalog rows for unchanged contracts. |
| NotificationService | Reused supporting service | Existing email verification delivery remains unchanged. | Verification only; no catalog update. |
| Admin app | Changed frontend client | Adds frontend-owned sign-in page and cookie-session route guard. | Update app routes/i18n only. |
| Seller app | Changed frontend client | Adds frontend-owned sign-in/sign-up pages and refresh after email/store role changes. | Update app routes/i18n only. |
| Buyer app | Changed frontend client | Adds frontend-owned sign-in/sign-up pages and storefront auth entry points. | Update app routes/i18n only. |
| `@hivespace/shared` | Changed frontend shared package | Adds cookie-session auth contracts/service/composable behavior, credentials/CSRF HTTP behavior, and shared `AuthLayout`. | Update shared exports and i18n where needed. |
| Config repo | Changed supporting config | Mirrors cookie names, CSRF header, CORS/credentials, gateway URL, and frontend fallback origins. | Sync config after implementation if appsettings change. |

### Data model

See [data-model.md](data-model.md).

This feature introduces no new domain aggregate, repository, EF Core table, or migration. Identity accounts remain IdentityService-owned, User profiles remain UserService-owned, and session state is represented by token cookies plus derived frontend session state.

### Cookie/session model

| Item | Owner | Notes |
| --- | --- | --- |
| `__Host-HiveSpace.AccessToken` | IdentityService issues, ApiGateway reads | HttpOnly JWT access token cookie. No application-level encryption or Data Protection envelope. Gateway copies the cookie value into `Authorization: Bearer <token>` after expiry/signature validation. |
| `__Host-HiveSpace.RefreshToken` | IdentityService | HttpOnly refresh token or refresh handle cookie. Used only by refresh/logout account endpoints. No JSON response exposes it. |
| `HiveSpace.Csrf` | IdentityService issues, frontend reads, ApiGateway validates | Readable signed CSRF token cookie mirrored in session response. Required in `X-HiveSpace-CSRF` on cookie-authenticated state-changing requests. |
| `SessionResponse` | IdentityService API contract | Contains user summary, expiry timestamps, `csrfToken`, and redirect target only. It must not contain access or refresh token fields. |

### Integration events

No new integration events are introduced.

| Existing contract | Producer | Consumer(s) | Reuse decision |
| --- | --- | --- | --- |
| `IdentityUserCreatedIntegrationEvent` | IdentityService | UserService | Reused unchanged for profile creation after registration; no event catalog update needed. |
| `StoreCreatedIntegrationEvent` | UserService | IdentityService and projections | Reused unchanged for seller role/claim propagation; no event catalog update needed. |
| Email verification events | IdentityService | NotificationService | Reused unchanged; no event catalog update needed. |

### API endpoints

The feature uses the existing account route group in [shared/api-catalog.md](../../shared/api-catalog.md).

| Method | Path | Auth | Handler / behavior |
| --- | --- | --- | --- |
| POST | `/api/v1/accounts/login` | Anonymous | Validate password login, set token cookies and CSRF token, return `SessionResponse`. |
| POST | `/api/v1/accounts/register` | Anonymous | Create Identity account, publish existing profile event, set token cookies and CSRF token, return `SessionResponse`. |
| POST | `/api/v1/accounts/session/refresh` | Access/refresh cookie + CSRF | Validate account/session, rotate token cookies and CSRF token, return `SessionResponse`. |
| POST | `/api/v1/accounts/logout` | Cookie + CSRF when present | Clear token and CSRF cookies, invalidate refresh handle where applicable, accept no redirect target, return `204`. |

No new public endpoint path is added beyond the cataloged `/api/v1/accounts/**` rows.

### Contracts

See [contracts/browser-auth-api.md](contracts/browser-auth-api.md).

### Saga design

No saga required. This feature is request/response browser authentication plus gateway mediation. It does not add a MassTransit state machine, compensation path, timeout workflow, or long-running multi-service orchestration.

### Architecture decision

Use [ADR-0003: Gateway-Mediated Cookie Browser Sessions](../../architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md). This ADR captures the cross-repo decision to use HttpOnly token cookies, gateway bearer forwarding, and shared frontend-owned auth UI.

---

## Phase 2 - Implementation Plan

### Backend layer order

IdentityService is a Lite service, but new behavior still follows CQRS-style command handlers plus Minimal API endpoints. ApiGateway remains thin infrastructure and must not own business authorization.

**IdentityService**

- [ ] Add account session DTOs and commands for login, register, refresh, and logout; logout accepts no redirect target because frontend apps own post-logout navigation.
- [ ] Implement a token cookie service that writes raw access and refresh token cookies with `HttpOnly`, `Secure`, `SameSite=None`, `Path=/`, and no `Domain`.
- [ ] Implement CSRF token issuance/rotation separate from bearer/refresh token values.
- [ ] Move password login and registration behavior from Razor UI handlers into CQRS handlers.
- [ ] Add Minimal API routes under `/api/v1/accounts/**`.
- [ ] Replace old `/Account/Login`, `/Account/Register`, and `/Account/Logout` UI with compatibility redirects to configured frontend routes.
- [ ] Remove obsolete IdentityService auth Razor UI once handlers and redirects are in place.

**ApiGateway**

- [ ] Add middleware to read the access-token cookie directly, validate token expiry/signature using existing JWT validation configuration, attach `Authorization: Bearer <token>` before forwarding, and strip browser cookies from downstream requests.
- [ ] Add middleware to reject cookie-authenticated `POST`, `PUT`, `PATCH`, and `DELETE` requests without a valid `X-HiveSpace-CSRF` token.
- [ ] Preserve existing YARP route prefixes, CORS, WebSockets, health checks, and downstream authorization expectations.

**Config**

- [ ] Configure cookie names, CSRF header name, frontend origins, credentials/CORS behavior, and old account URL fallback origins.
- [ ] Do not configure shared Data Protection for auth-cookie decryption because the cookie stores token values without an encrypted application envelope.

### Frontend plan

Build order remains types -> service -> store/composable -> components -> pages -> routes -> i18n.

**Shared package**

- [ ] Add account session types and browser auth service for `/api/v1/accounts/**`.
- [ ] Update `useAuth` to store only derived `SessionUser`, expiry timestamps, and CSRF token, and to clean old OIDC web-storage keys.
- [ ] Update `ApiService` to send cookies with credentials, add `X-HiveSpace-CSRF` on state-changing requests, and stop injecting script-held bearer tokens.
- [ ] Update SignalR notification auth to rely on gateway cookies instead of `accessTokenFactory`.
- [ ] Create `packages/shared/src/components/layout/AuthLayout.vue` based on the demo `Signin.vue`/`Signup.vue` layout:
  - wraps `FullScreenLayout`;
  - exposes a left slot for the page form;
  - renders a shared right panel with `CommonGridShape` background;
  - accepts translated heading/body keys plus a center image source/alt key;
  - does not import app-local assets or production-import demo package files.

**Apps**

- [ ] Admin creates only a sign-in page that uses shared `AuthLayout` and provides the admin form in the left slot.
- [ ] Seller creates sign-in and sign-up pages using shared `AuthLayout` with seller-specific right-panel text and image.
- [ ] Buyer creates sign-in and sign-up pages using shared `AuthLayout` with buyer-specific right-panel text and image.
- [ ] Each app updates route guards to route to frontend auth pages instead of IdentityService/OIDC UI redirects.
- [ ] Each app updates English and Vietnamese auth i18n together.

### Verification

- Backend: targeted IdentityService and ApiGateway builds after implementation; search responses for forbidden token fields.
- Frontend: targeted shared/app builds or type checks; verify no script-readable access/refresh token storage remains.
- Browser/manual: use [quickstart.md](quickstart.md) for login, registration, refresh, logout, CSRF, cross-app session, and old IdentityService URL redirect checks.
- Docs/catalogs: account endpoint catalog rows already exist; event catalog remains unchanged because no new integration message is introduced.

---

## Constitution Compliance Check

- [x] No hard-deletes planned for business entities; only obsolete UI files may be removed.
- [x] No persisted money values involved.
- [x] No new aggregate IDs or EF migrations involved.
- [x] Existing integration events are reused unchanged; no new MassTransit event/outbox contract is introduced.
- [x] No new NuGet or npm dependency version plan requires direct version attributes in `.csproj`.
- [x] Frontend text must use i18n keys, with English and Vietnamese resources updated together.
- [x] Apps import shared runtime code only from `@hivespace/shared`; production apps do not import demo package components.
- [x] Reused common contracts are unchanged and require no event catalog update.

## Post-Design Constitution Check

The design remains compliant after the cookie/no-encryption and shared `AuthLayout` update. The main security tradeoff is explicit: tokens are not application-encrypted inside the cookie, so controls rely on HttpOnly/Secure/SameSite cookie attributes, token signing/expiry validation, CSRF checks, short access-token lifetime, and refresh rotation/invalidation in IdentityService.
