# Frontend Tasks

## Source Repo Setup And Impact Analysis

### Verify

- [ ] F001 Verify frontend repo instructions and GitNexus impact scope
  - File: `../hivespace.web/AGENTS.md`, `../hivespace.web/CLAUDE.md`
  - Read both instruction files before editing frontend source.
  - Run GitNexus impact/context checks before modifying existing symbols in shared `useAuth`, `ApiService`, notification realtime helpers, app router guards, and app API singletons.
  - Record direct callers and risk in implementation notes before editing; do not ignore HIGH or CRITICAL impact.
  - Acceptance: implementation notes identify impacted auth symbols/callers and confirm repo instructions were followed.

## Shared Package `@hivespace/shared`

### Create

- [ ] F002 [US1] [US2] [US3] [US4] Create account session auth types
  - File: `../hivespace.web/packages/shared/src/types/auth-session.ts`, `../hivespace.web/packages/shared/src/types/index.ts`, `../hivespace.web/packages/shared/src/internal.ts`
  - Add `AuthApp = 'admin' | 'seller' | 'buyer'`, `SignInRequest`, `RegisterAccountRequest`, `RefreshSessionRequest`, `SignOutRequest`, `SessionUser`, `SessionResponse`, and standard API error response types matching backend `ExceptionModel`/`ErrorCodeDto`.
  - Ensure response types contain `csrfToken`, `expiresAt`, `refreshExpiresAt`, `redirectTo`, and `user`; do not include access token or refresh token fields.
  - Export types from shared public/internal entrypoints used by apps.
  - Acceptance: apps can import shared auth session types without defining local duplicates.

- [ ] F003 [US1] [US2] [US3] [US4] Create account session auth service
  - File: `../hivespace.web/packages/shared/src/features/auth/account-session.service.ts`, `../hivespace.web/packages/shared/src/features/auth/index.ts`
  - Implement calls for `POST /api/v1/accounts/login`, `POST /api/v1/accounts/register`, `POST /api/v1/accounts/session/refresh`, and `POST /api/v1/accounts/logout`.
  - Build URLs from configured gateway base URL, supporting `VITE_GATEWAY_BASE_URL=http://localhost:5000`.
  - Use `credentials: 'include'` or axios `withCredentials: true`; send `X-HiveSpace-CSRF` on refresh/logout when token exists.
  - Do not call IdentityService authority URL directly and do not use `/connect/token`.
  - Acceptance: service targets only gateway `/api/v1/accounts/**` routes and returns typed session responses.

- [ ] F004 [US1] [US2] [US3] [US4] Create shared auth image contract
  - File: `../hivespace.web/packages/shared/src/types/auth-branding.ts`
  - Define app auth image metadata shape: `imageSrc`, `altKey`, `headingKey`, `bodyKey`, and `app`.
  - Use it from app auth pages to replace demo `CommonGridShape` with app-specific image assets.
  - Do not import app-local assets into the shared package.
  - Acceptance: shared page/component contracts allow each app to provide its own auth image and translated text.

### Update

- [ ] F005 [US1] [US2] [US3] [US4] Update shared `useAuth` for cookie-session auth
  - File: `../hivespace.web/packages/shared/src/composables/useAuth.ts`
  - Replace `oidc-client-ts` redirect flow with browser-session methods: `initializeAuth({ gatewayBaseUrl, app })`, `login(request)`, `register(request)`, `refreshSession(app)`, `logout(redirectTo?)`, `getCurrentUser()`, `isAuthenticated`, and `currentUser`.
  - Make `getCurrentUser()` bootstrap by calling `POST /api/v1/accounts/session/refresh` when no in-memory session exists and CSRF/session cookies are present.
  - Store only derived `SessionUser`, expiry timestamps, and CSRF token; clean known OIDC web-storage keys on bootstrap.
  - Preserve method names needed by existing callers where possible, but change implementation to route to frontend sign-in pages instead of IdentityService redirects.
  - Acceptance: no `oidc-client-ts` import remains in `useAuth.ts`, and shared auth exposes cookie-session methods used by app routers/pages.

- [ ] F006 [US3] [US4] Update shared `ApiService` for credentials and CSRF
  - File: `../hivespace.web/packages/shared/src/utils/api.ts`
  - Set axios `withCredentials: true`.
  - Remove bearer `Authorization` injection from `currentUser.access_token`.
  - Add `X-HiveSpace-CSRF` for `POST`, `PUT`, `PATCH`, and `DELETE` when a CSRF token exists in shared auth state or `HiveSpace.Csrf` cookie.
  - On `401`, attempt one `refreshSession` before logout for protected requests; avoid retry loops.
  - Preserve `X-Correlation-ID`, `X-Request-Timestamp`, retry behavior for safe methods, and translated error callbacks.
  - Acceptance: API calls use cookies/CSRF and no script-held bearer token is read.

- [ ] F007 [US3] Update shared notification realtime auth
  - File: `../hivespace.web/packages/shared/src/composables/useNotificationHub.ts`, `../hivespace.web/packages/shared/src/features/notifications/useNotificationRealtime.ts`
  - Remove `accessTokenFactory` and watchers that depend on `currentUser.value?.access_token`.
  - Configure SignalR to send browser credentials through the gateway and reconnect after shared session refresh.
  - Do not expose access tokens to the SignalR client.
  - Acceptance: notification hub setup no longer requires script-readable token fields.

- [ ] F008 [US1] [US2] [US3] [US4] Update shared auth exports and dependencies
  - File: `../hivespace.web/packages/shared/src/internal.ts`, `../hivespace.web/packages/shared/package.json`, `../hivespace.web/pnpm-lock.yaml`
  - Export new auth service/types from shared package entrypoints.
  - Remove `oidc-client-ts` from shared runtime dependencies only if no remaining shared code imports it.
  - If dependency or lockfile changes are needed, remember backend/frontend repo guardrails: JSON files require user staging for commits.
  - Acceptance: shared package builds with new exports and no unused OIDC dependency remains when safe to remove.

### Verify

- [ ] F009 [US1] [US2] [US3] [US4] Verify no browser-readable token contract remains in shared auth
  - File: `../hivespace.web/packages/shared/src/**`
  - Search for `access_token`, `refresh_token`, `id_token`, `signinRedirect`, `signoutRedirect`, `/connect/token`, and `WebStorageStateStore`.
  - Keep only non-auth wording matches that are not part of live auth/session behavior and document any unavoidable references.
  - Acceptance: shared auth/session code no longer stores or reads access/refresh tokens in web storage.

## Demo Auth Page Reuse Source

### Verify

- [ ] F010 [US1] [US2] Verify demo auth page source before adapting
  - File: `../hivespace.web/packages/demo/src/Auth/Signin.vue`, `../hivespace.web/packages/demo/src/Auth/Signup.vue`
  - Use these two pages as the visual/layout starting point for app sign-in and sign-up pages.
  - Remove demo-only social login buttons, `/demo` links, console logging, hardcoded strings, and `CommonGridShape`.
  - Replace the right-side panel with app-provided bitmap image assets and translated app-specific copy.
  - Acceptance: app tasks reference the demo source but do not import demo components into production apps.

## Admin App

### Create

- [ ] F011 [US1] Create admin sign-in page from demo `Signin.vue`
  - File: `../hivespace.web/apps/admin/src/pages/Auth/SignInPage.vue`
  - Adapt structure from `packages/demo/src/Auth/Signin.vue` using `FullScreenLayout`, email/password form, password visibility toggle, and submit button.
  - Remove social-login buttons, `CommonGridShape`, "Back to dashboard", forgot-password link unless an existing route exists, and all hardcoded English text.
  - Use shared `useAuth().login({ email, password, app: 'admin', returnUrl, culture })` and route success to the requested admin route or admin default.
  - Show safe errors from shared auth through i18n keys and app notifications.
  - Acceptance: admin sign-in page submits to shared browser auth and contains no direct IdentityService/OIDC redirect call.

- [ ] F012 [US1] Create admin auth image asset
  - File: `../hivespace.web/apps/admin/src/assets/auth/admin-auth.jpg`
  - Add or generate a professional admin operations/dashboard-themed bitmap image suitable for the right-side auth panel.
  - Use actual image asset, not `CommonGridShape`, gradient orbs, SVG illustration, or decorative-only background.
  - Acceptance: `SignInPage.vue` renders the image with translated alt text and responsive object-cover sizing.

### Update

- [ ] F013 [US1] [US3] [US4] Update admin auth config and API singleton
  - File: `../hivespace.web/apps/admin/src/config/index.ts`, `../hivespace.web/apps/admin/src/main.ts`, `../hivespace.web/apps/admin/src/services/api.ts`
  - Change auth initialization to pass app id `admin` and gateway base URL into shared auth.
  - Keep `VITE_GATEWAY_BASE_URL` fallback support; allow local HTTP gateway `http://localhost:5000`.
  - Remove admin OIDC redirect URI/post-logout dependency from runtime auth flow, keeping only transitional config if needed for cleanup.
  - Ensure app `ApiService` singleton uses shared credentials/CSRF behavior and no refresh-token callback.
  - Acceptance: admin bootstrap uses cookie-session auth through gateway config.

- [ ] F014 [US1] [US3] [US4] Update admin router guard
  - File: `../hivespace.web/apps/admin/src/router/index.ts`
  - Add `/signin` route with `meta.allowAnonymous: true` and route component `SignInPage.vue`.
  - Replace OIDC `login()` redirect behavior with redirect to `/signin?returnUrl=...`.
  - On authenticated non-admin/non-system-admin session, call shared logout and redirect to sign-in with safe access-denied message.
  - Remove normal dependency on `/callback/login` and `/callback/logout`; keep callback routes only as transitional redirects if needed.
  - Acceptance: protected admin routes trigger frontend sign-in and valid admin/system-admin session passes.

- [ ] F015 [US1] Update admin auth i18n
  - File: `../hivespace.web/apps/admin/src/i18n/locales/en/auth.json`, `../hivespace.web/apps/admin/src/i18n/locales/vi/auth.json`, `../hivespace.web/apps/admin/src/i18n/index.ts`
  - Add `auth.signIn`, `auth.errors`, `auth.actions`, and image alt/heading/body keys for admin auth page.
  - Keep reusable generic session strings in `common` if already shared; do not overload one key as both string and namespace.
  - Acceptance: `SignInPage.vue` has no user-facing hardcoded strings.

## Seller App

### Create

- [ ] F016 [US1] Create seller sign-in page from demo `Signin.vue`
  - File: `../hivespace.web/apps/seller/src/pages/Auth/SignInPage.vue`
  - Adapt demo layout and email/password form for seller audience.
  - Remove social-login buttons, `CommonGridShape`, demo routes, console logging, and hardcoded strings.
  - Use `useAuth().login({ app: 'seller' })`, preserving seller route behavior after email verification/store role propagation.
  - Acceptance: seller sign-in page submits through shared browser auth and routes according to seller access rules.

- [ ] F017 [US2] Create seller sign-up page from demo `Signup.vue`
  - File: `../hivespace.web/apps/seller/src/pages/Auth/SignUpPage.vue`
  - Adapt demo sign-up page using full name or first/last name mapped to `fullName`, email, password, confirm password, terms checkbox where required, and shared `register({ app: 'seller' })`.
  - Remove social-login buttons, `CommonGridShape`, demo links, console logging, and hardcoded strings.
  - After registration, route to seller email verification or seller onboarding flow according to current seller router rules.
  - Acceptance: seller sign-up page calls `/api/v1/accounts/register` through shared auth and handles duplicate/validation errors safely.

- [ ] F018 [US1] [US2] Create seller auth image asset
  - File: `../hivespace.web/apps/seller/src/assets/auth/seller-auth.jpg`
  - Add or generate a merchant storefront/order management-themed bitmap image suitable for seller sign-in/sign-up right panel.
  - Use actual image asset, not `CommonGridShape`, gradient orbs, SVG illustration, or decorative-only background.
  - Acceptance: seller auth pages render the image with translated alt text and responsive object-cover sizing.

### Update

- [ ] F019 [US1] [US2] [US3] [US4] Update seller auth config and API singleton
  - File: `../hivespace.web/apps/seller/src/config/index.ts`, `../hivespace.web/apps/seller/src/main.ts`, `../hivespace.web/apps/seller/src/services/api.ts`, `../hivespace.web/apps/seller/src/services/refresh.service.ts`
  - Change auth initialization to pass app id `seller` and gateway base URL into shared auth.
  - Remove direct `/connect/token` refresh service usage from the API singleton; route refresh through shared `/api/v1/accounts/session/refresh`.
  - Preserve existing seller email verification and store registration flows by calling shared `refreshSession('seller')` after those transitions.
  - Acceptance: seller app no longer uses script-readable refresh tokens for route/API refresh.

- [ ] F020 [US1] [US2] [US3] [US4] Update seller router guard
  - File: `../hivespace.web/apps/seller/src/router/index.ts`
  - Add `/signin` and `/signup` anonymous routes.
  - Replace OIDC login redirect with frontend `/signin?returnUrl=...`.
  - Preserve seller-specific rules: admins/system-admins denied, unverified users go to `/verify-email`, verified non-sellers go to `/register-seller`, verified sellers pass.
  - Remove normal dependencies on `/callback/login` and `/callback/logout`; keep transitional redirects only if needed.
  - Acceptance: seller routing is driven by shared browser session and current role/email state.

- [ ] F021 [US2] [US3] Update seller post-verification and store registration refresh calls
  - File: `../hivespace.web/apps/seller/src/stores/account.store.ts`, `../hivespace.web/apps/seller/src/stores/store.store.ts`, `../hivespace.web/apps/seller/src/pages/VerifyEmailPage.vue`, `../hivespace.web/apps/seller/src/pages/Callback/VerifyEmailCallbackPage.vue`, `../hivespace.web/apps/seller/src/pages/RegisterStorePage.vue`
  - Replace app-local refresh-token service calls with shared `refreshSession('seller')`.
  - Keep existing success/error notifications and route outcomes.
  - Do not read `currentUser.refresh_token`.
  - Acceptance: email verification/store registration can refresh seller authorization state without browser-readable tokens.

- [ ] F022 [US1] [US2] Update seller auth i18n
  - File: `../hivespace.web/apps/seller/src/i18n/locales/en/auth.json`, `../hivespace.web/apps/seller/src/i18n/locales/vi/auth.json`, `../hivespace.web/apps/seller/src/i18n/index.ts`
  - Add sign-in/sign-up labels, validation messages, safe auth errors, image alt/heading/body, and session-expired copy.
  - Keep seller-specific copy in `auth`; keep shell/common generic strings in `common`.
  - Acceptance: seller auth pages have no user-facing hardcoded strings.

## Buyer App

### Create

- [ ] F023 [US1] Create buyer sign-in page from demo `Signin.vue`
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignInPage.vue`
  - Adapt demo layout and email/password form for storefront users.
  - Remove social-login buttons, `CommonGridShape`, demo routes, console logging, and hardcoded strings.
  - Use shared `useAuth().login({ app: 'buyer' })`, preserving return URL for protected cart/checkout/profile routes.
  - Acceptance: buyer sign-in page submits through shared browser auth and returns to requested protected route.

- [ ] F024 [US2] Create buyer sign-up page from demo `Signup.vue`
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignUpPage.vue`
  - Adapt demo sign-up page using full name or first/last name mapped to `fullName`, email, password, confirm password, terms checkbox where required, and shared `register({ app: 'buyer' })`.
  - Remove social-login buttons, `CommonGridShape`, demo links, console logging, and hardcoded strings.
  - Acceptance: buyer sign-up page calls `/api/v1/accounts/register` through shared auth and handles duplicate/validation errors safely.

- [ ] F025 [US1] [US2] Create buyer auth image asset
  - File: `../hivespace.web/apps/buyer/src/assets/auth/buyer-auth.jpg`
  - Add or generate a shopper/storefront-themed bitmap image suitable for buyer sign-in/sign-up right panel.
  - Use actual image asset, not `CommonGridShape`, gradient orbs, SVG illustration, or decorative-only background.
  - Acceptance: buyer auth pages render the image with translated alt text and responsive object-cover sizing.

### Update

- [ ] F026 [US1] [US2] [US3] [US4] Update buyer auth config and API singleton
  - File: `../hivespace.web/apps/buyer/src/config/index.ts`, `../hivespace.web/apps/buyer/src/main.ts`, `../hivespace.web/apps/buyer/src/services/api.ts`, `../hivespace.web/apps/buyer/src/services/refresh.service.ts`
  - Change auth initialization to pass app id `buyer` and gateway base URL into shared auth.
  - Remove direct `/connect/token` refresh service usage from the API singleton; route refresh through shared `/api/v1/accounts/session/refresh`.
  - Keep `VITE_GATEWAY_BASE_URL` fallback support and allow `http://localhost:5000`.
  - Acceptance: buyer app no longer uses script-readable refresh tokens for route/API refresh.

- [ ] F027 [US1] [US2] [US3] [US4] Update buyer router and header auth actions
  - File: `../hivespace.web/apps/buyer/src/router/index.ts`, `../hivespace.web/apps/buyer/src/components/layout/AppHeader.vue`
  - Add `/signin` and `/signup` anonymous routes with `meta.layout: 'none'` if needed to avoid standard storefront header/footer.
  - Replace OIDC login/register redirects with navigation to frontend pages.
  - Preserve protected route return URL behavior for cart, checkout, profile, and order routes.
  - Wire header sign-in/sign-up buttons to app routes instead of shared OIDC redirect methods.
  - Acceptance: buyer header and guards open frontend auth pages and no IdentityService UI is reached.

- [ ] F028 [US1] [US2] Update buyer auth i18n
  - File: `../hivespace.web/apps/buyer/src/i18n/locales/en/auth.json`, `../hivespace.web/apps/buyer/src/i18n/locales/vi/auth.json`, `../hivespace.web/apps/buyer/src/i18n/index.ts`
  - Add sign-in/sign-up labels, validation messages, safe auth errors, image alt/heading/body, and session-expired copy.
  - Use storefront-appropriate wording but keep module namespace stable.
  - Acceptance: buyer auth pages/header auth actions have no user-facing hardcoded strings.

## Cross-App Cleanup

### Remove

- [ ] F029 [US1] [US2] [US3] [US4] Remove obsolete app OIDC callback dependencies
  - File: `../hivespace.web/apps/admin/src/pages/Callback/**`, `../hivespace.web/apps/seller/src/pages/Callback/LoginCallbackPage.vue`, `../hivespace.web/apps/seller/src/pages/Callback/LogoutCallbackPage.vue`, `../hivespace.web/apps/buyer/src/pages/Callback/**`
  - Remove login/logout callback page imports/routes when no longer used, or convert them to transitional redirects to frontend sign-in/signed-out routes.
  - Keep seller email verification callback only if still required by email links.
  - Do not remove unrelated callback pages used by payment or verification flows.
  - Acceptance: router has no normal login/logout OIDC callback dependency and remaining callbacks are justified.

### Verify

- [ ] F030 [US1] [US2] [US3] [US4] Verify app auth pages use demo layout but not demo imports
  - File: `../hivespace.web/apps/admin/src/pages/Auth/*`, `../hivespace.web/apps/seller/src/pages/Auth/*`, `../hivespace.web/apps/buyer/src/pages/Auth/*`
  - Confirm pages are local app pages based on demo structure, not imports from `packages/demo`.
  - Confirm no app auth page imports or renders `CommonGridShape`.
  - Confirm app-specific auth image assets render in the right panel.
  - Acceptance: production apps do not depend on demo auth components and use app-specific images.
