# Verification Tasks

## Backend Verification

### Verify

- [ ] V001 [US1] [US2] [US3] [US4] Verify IdentityService build
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj`
  - Run targeted build from backend repo after IdentityService tasks complete.
  - Confirm no Razor Page deletion errors, stale usings, missing DI registrations, or endpoint mapping failures.
  - Acceptance: IdentityService builds; only documented baseline warnings remain.

- [ ] V002 [US3] [US4] Verify ApiGateway build
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj`
  - Run targeted build from backend repo after gateway middleware/config tasks complete.
  - Confirm middleware registrations compile and YARP route config still loads.
  - Acceptance: ApiGateway builds and can start on `http://localhost:5000`.

- [ ] V003 [US1] [US2] [US3] [US4] Verify backend forbidden token exposure
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/**`, `../hivespace.microservice/src/HiveSpace.ApiGateway/**`
  - Search browser auth endpoint DTOs/responses for `access_token`, `refresh_token`, `id_token`, and token-returning JSON fields.
  - Confirm access/refresh token values only appear in HttpOnly cookie-setting code, token validation paths, or gateway forwarding header code.
  - Acceptance: no browser JSON response exposes access or refresh token material.

- [ ] V004 [US1] [US2] [US4] Verify old IdentityService UI removal
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/**`
  - Search for login/register/logout Razor Page files, handlers, route links, and auth-only CSS references.
  - Confirm `/Account/Login`, `/Account/Register`, and `/Account/Logout` are handled only by minimal compatibility redirects.
  - Acceptance: old URLs redirect to frontend routes and no IdentityService auth UI remains.

## Frontend Verification

### Verify

- [ ] V005 [US1] [US2] [US3] [US4] Verify shared package build/type safety
  - File: `../hivespace.web/packages/shared`
  - Run shared package build/type-check command available in the frontend repo after shared auth/API changes.
  - Confirm no introduced TypeScript errors from auth session types, `useAuth`, `ApiService`, or notification realtime changes.
  - Acceptance: shared package checks pass or only known baseline issues unrelated to touched files remain.

- [ ] V006 [US1] [US2] [US3] [US4] Verify admin app checks
  - File: `../hivespace.web/apps/admin`
  - Run `pnpm lint` and `pnpm type-check` from admin app after admin auth page/router/config changes.
  - Known baseline failures may exist; record only introduced errors from touched auth files and fix them.
  - Acceptance: no new admin errors in auth page, router, config, API singleton, or i18n wiring.

- [ ] V007 [US1] [US2] [US3] [US4] Verify seller app checks
  - File: `../hivespace.web/apps/seller`
  - Run `pnpm lint` and `pnpm type-check` from seller app after seller auth page/router/config changes.
  - Known baseline failures may exist; record only introduced errors from touched auth files and fix them.
  - Acceptance: no new seller errors in auth pages, router, config, API singleton, stores, or i18n wiring.

- [ ] V008 [US1] [US2] [US3] [US4] Verify buyer app checks
  - File: `../hivespace.web/apps/buyer`
  - Run targeted buyer build/type-check command available in the frontend repo after buyer auth page/router/config changes.
  - Confirm buyer auth pages compile with `meta.layout: 'none'` route behavior where used.
  - Acceptance: no new buyer errors in auth pages, header, router, config, API singleton, or i18n wiring.

- [ ] V009 [US1] [US2] [US3] Verify frontend forbidden token storage
  - File: `../hivespace.web/packages/shared/src`, `../hivespace.web/apps/admin/src`, `../hivespace.web/apps/seller/src`, `../hivespace.web/apps/buyer/src`
  - Search for `localStorage`, `sessionStorage`, `access_token`, `refresh_token`, `id_token`, `WebStorageStateStore`, `/connect/token`, and `oidc-client-ts`.
  - Document any remaining non-auth or cleanup-only references; remove live token storage and refresh grant usage.
  - Acceptance: browser auth flow stores no access/refresh token in script-readable storage.

- [ ] V010 [US1] [US2] Verify auth pages use shared AuthLayout and app images
  - File: `../hivespace.web/apps/admin/src/pages/Auth/*`, `../hivespace.web/apps/seller/src/pages/Auth/*`, `../hivespace.web/apps/buyer/src/pages/Auth/*`
  - Search app auth pages for direct `CommonGridShape` imports/usages, `common-grid-shape`, social login buttons, `/demo`, and hardcoded English user-facing strings.
  - Confirm pages render through shared `AuthLayout`, which owns the `CommonGridShape` right-side background.
  - Confirm each app passes its own SVG image asset into `AuthLayout`: `admin-auth.svg`, `seller-auth.svg`, `buyer-auth.svg`.
  - Acceptance: app auth pages are production app pages using shared layout, app images, and i18n.

## Manual Quickstart Verification

### Verify

- [ ] V011 [US1] Verify sign-in flow through ApiGateway
  - File: `specs/0004-cookie-auth-migration/quickstart.md`
  - Set frontend `VITE_GATEWAY_BASE_URL=http://localhost:5000`, start ApiGateway and IdentityService, then open admin/seller/buyer apps.
  - Sign in with valid and invalid credentials in each app.
  - Confirm network requests target `POST /api/v1/accounts/login` through gateway and no IdentityService UI page renders.
  - Measure the time from sign-in form submit to post-login route render for at least one valid admin, seller, and buyer login under local development with ApiGateway and IdentityService already running.
  - Confirm at least 95% of repeated successful attempts reach the expected destination within 5 seconds, or document any baseline local infrastructure delay.
  - Acceptance: valid sign-in reaches correct app destination within the SC-002 timing target and invalid sign-in shows safe error.

- [ ] V012 [US2] Verify registration flow through ApiGateway
  - File: `specs/0004-cookie-auth-migration/quickstart.md`
  - Register a new buyer and seller candidate from frontend-owned pages.
  - Confirm `POST /api/v1/accounts/register` creates one IdentityService account and publishes existing `IdentityUserCreatedIntegrationEvent`.
  - Confirm duplicate email and validation errors are safe and no duplicate account is created.
  - Acceptance: registration meets SC-003 and uses no direct IdentityService authority URL.

- [ ] V013 [US3] Verify protected API, refresh, storage, and cross-app session
  - File: `specs/0004-cookie-auth-migration/quickstart.md`
  - After sign-in, call protected profile/cart/product/order/admin paths through frontend apps.
  - Reload the app and confirm `POST /api/v1/accounts/session/refresh` runs through gateway.
  - Inspect localStorage/sessionStorage for absence of access/refresh tokens.
  - Open another HiveSpace frontend app and confirm shared session is accepted while role rules still apply.
  - Acceptance: protected workflows preserve existing authorization outcomes and no script-readable tokens are present.

- [ ] V014 [US3] Verify CSRF rejection at gateway
  - File: `specs/0004-cookie-auth-migration/quickstart.md`
  - Replay a cookie-authenticated `POST`, `PUT`, `PATCH`, or `DELETE` request without `X-HiveSpace-CSRF`.
  - Confirm ApiGateway returns the standard HiveSpace exception response with `CommonErrorCode.CsrfValidationFailed` (`APP0018`) before downstream state changes.
  - Replay with valid CSRF header and confirm normal authorization behavior.
  - Acceptance: 100% of tested missing-CSRF state-changing requests are rejected at gateway.

- [ ] V015 [US4] Verify logout and old URL redirects
  - File: `specs/0004-cookie-auth-migration/quickstart.md`
  - Sign out from each app and confirm session/CSRF cookies are cleared and protected routes require sign-in after refresh.
  - Open `http://localhost:5001/Account/Login`, `/Account/Register`, and `/Account/Logout` directly.
  - Confirm each old URL redirects to frontend route and no Razor UI is rendered.
  - Acceptance: logout and old URL compatibility match SC-006 and SC-009.

## Final Scope Checks

### Verify

- [ ] V016 Verify GitNexus change scope before commit
  - File: `../hivespace.microservice`, `../hivespace.web`
  - Run GitNexus detect-changes in each source repo after implementation and before commit.
  - Confirm changed symbols/files match expected backend auth/gateway and frontend auth/app page scopes.
  - Acceptance: no unexpected high-risk unrelated changes are reported.

- [ ] V017 Verify docs/catalog consistency after implementation
  - File: `shared/api-catalog.md`, `shared/event-catalog.md`, `services/identity-service/**`, `services/api-gateway/**`
  - Confirm docs/catalog entries match implemented endpoint paths, auth requirements, cookie/CSRF behavior, and event no-change policy.
  - Acceptance: durable docs match source implementation and no stale IdentityService UI wording remains.
