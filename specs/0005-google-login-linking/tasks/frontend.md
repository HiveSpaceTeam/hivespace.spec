# Frontend Tasks

## Shared Package: `@hivespace/shared`

### Create

- [ ] F001 [US1] [US2] Create Google auth shared types
  - File: `../hivespace.web/packages/shared/src/types/auth-session.ts`
  - Add `GoogleAuthApp` or reuse existing `AuthApp` constrained to `buyer | seller` for Google flows; add `StartGoogleAuthRequest`, `ConfirmGoogleLinkRequest` with required `consentAccepted: true`, and any safe error-code type needed by buyer/seller pages.
  - Export the new types through `../hivespace.web/packages/shared/src/types/index.ts`.
  - Do not include provider access tokens, refresh tokens, or provider user IDs in frontend types.
  - Acceptance: shared package type exports compile and represent the browser-auth contract inputs.

- [ ] F002 [US1] [US2] Create shared Google account-session service helpers
  - File: `../hivespace.web/packages/shared/src/features/auth/account-session.service.ts`
  - Add `startGoogleAuth(request)` to build `/api/v1/accounts/external/google/challenge?app=...&returnUrl=...&culture=...` from the configured gateway base URL and navigate the browser to it.
  - Add `confirmGoogleLink(request)` and `cancelGoogleLink()` using `POST`/`DELETE /external/google/link`, `credentials: 'include'`, and the existing CSRF header helper.
  - Reuse `normalizeGatewayBaseUrl`, `buildAccountUrl`, and `requestJson` patterns already in the file.
  - Do not create app-local duplicate fetch helpers for the same account endpoints.
  - Acceptance: buyer and seller can call the same shared service methods for Google challenge, link confirm, and cancel.

### Update

- [ ] F003 [US1] [US2] [US3] Update shared `useAuth` Google actions
  - File: `../hivespace.web/packages/shared/src/composables/useAuth.ts`
  - Expose `startGoogleAuth`, `confirmGoogleLink`, and `cancelGoogleLink`; default app/culture from initialized auth config and i18n locale; normalize frontend return URLs with `normalizeFrontendRedirect`.
  - Apply returned `SessionResponse` from successful link through existing `applySession`.
  - Preserve password `login`, `register`, `refreshSession`, `logout`, `getCurrentUser`, and old OIDC storage cleanup behavior.
  - Do not allow `admin` to start Google auth from shared API.
  - Acceptance: buyer/seller pages can use `useAuth()` for both password and Google flows without duplicating session state.

- [ ] F004 [US1] [US2] Update shared auth exports and Google button primitive
  - File: `../hivespace.web/packages/shared/src/features/auth/index.ts`, `../hivespace.web/packages/shared/src/internal.ts`, and `../hivespace.web/packages/shared/src/components/common/GoogleAuthButton.vue` if no suitable shared button exists
  - Reuse existing `GoogleIcon` from `packages/shared/src/icons/GoogleIcon.vue` and existing `Button` styling for a provider button that supports loading/disabled state and slot/label text.
  - Export any new shared component through the existing component barrels.
  - Do not create buyer-only and seller-only copies of the same Google provider button.
  - Acceptance: one shared Google button or equivalent shared pattern is available to both apps and imports from `@hivespace/shared`.

## Buyer App

### Create

- [ ] F005 [US2] Create buyer Google account-link confirmation page
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/GoogleLinkPage.vue`
  - Render pending-link consent copy, an explicit consent checkbox or switch, password input, confirm action, cancel action, error banner, and loading state using shared components and `useAuth().confirmGoogleLink/cancelGoogleLink`.
  - Send `consentAccepted: true` only after the user explicitly checks the consent control; keep confirm disabled or return a localized validation error until consent is checked.
  - On successful link, redirect to sanitized `returnUrl` or `/`; on cancel or declined consent, send user to `/signin` with a safe localized outcome and visible options for password sign-in, password reset, or choosing another Google account.
  - Do not display provider key, provider email if not included in safe frontend state, token values, or raw backend exception payloads.
  - Acceptance: existing password account can complete Google linking only after explicit consent plus password, can cancel linking, and cancel/decline never presents "continue without linking" as an option.

### Update

- [ ] F006 [US2] Update buyer router for Google link route
  - File: `../hivespace.web/apps/buyer/src/router/index.ts`
  - Add an anonymous route for the Google link confirmation page, for example `/auth/google/link`, preserving `returnUrl`, `culture`, and safe error query handling.
  - Ensure authenticated users follow current buyer guard behavior after a successful link/session refresh.
  - Do not mark admin or seller-only flows as buyer routes.
  - Acceptance: Google callback redirects can land on a buyer link page without triggering protected-route redirects.

- [ ] F007 [US1] Update buyer sign-in page for Google
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignInPage.vue`
  - Add a Google provider button near the password sign-in form using shared `GoogleIcon`/button behavior and `useAuth().startGoogleAuth({ app: 'buyer', returnUrl, culture })`.
  - Preserve existing password login validation, error mapping, `returnUrl`, and sign-up link behavior.
  - Do not call account service methods directly from the page for Google; use shared auth composable/service.
  - Acceptance: buyer sign-in starts `/api/v1/accounts/external/google/challenge` with `app=buyer`.

- [ ] F008 [US1] Update buyer sign-up page for Google
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignUpPage.vue`
  - Add the same Google provider flow as sign-in, preserving current password registration form and terms validation.
  - Reuse `returnUrl` and culture handling from the page.
  - Do not create a separate Google sign-up backend endpoint.
  - Acceptance: buyer sign-up starts the same Google challenge endpoint as buyer sign-in.

- [ ] F009 [US1] [US2] Update buyer auth i18n
  - File: `../hivespace.web/apps/buyer/src/i18n/locales/en/auth.json` and `../hivespace.web/apps/buyer/src/i18n/locales/vi/auth.json`
  - Add keys for Continue with Google, link confirmation title/body, password confirmation label, confirm/cancel actions, declined link, password sign-in, password reset, use another Google account, wrong password, Google cancellation/failure, blocked account, missing/unverified Google email, and generic link retry.
  - Keep key structure identical in English and Vietnamese.
  - Do not put feature-specific Google auth copy in `common.json` unless it is reused outside auth.
  - Acceptance: all new buyer user-facing text uses translation keys in both locales.

## Seller App

### Create

- [ ] F010 [US2] Create seller Google account-link confirmation page
  - File: `../hivespace.web/apps/seller/src/pages/Auth/GoogleLinkPage.vue`
  - Mirror buyer link-confirmation behavior with seller app context, explicit consent checkbox or switch, and seller-safe redirects, including safe exits for password sign-in, password reset, or choosing another Google account after declined/failed linking.
  - Send `consentAccepted: true` only after the user explicitly checks the consent control; keep confirm disabled or return a localized validation error until consent is checked.
  - After successful link, rely on existing seller guards to route normal users through email verification or store registration when they lack seller access.
  - Do not grant seller role, store ID, or seller access in frontend state.
  - Acceptance: seller app can confirm Google linking only after explicit consent plus password, can cancel linking, and does not bypass seller onboarding or offer same-email Google continuation without linking.

### Update

- [ ] F011 [US2] Update seller router for Google link route
  - File: `../hivespace.web/apps/seller/src/router/index.ts`
  - Add an anonymous route for the seller Google link confirmation page and preserve current seller guard order for authenticated users.
  - Keep admins/system-admins logged out per existing seller rules.
  - Do not route Google callback users directly to seller-only pages unless existing guard allows it.
  - Acceptance: Google callback redirects can land on seller link confirmation and then continue through existing seller auth/onboarding guards.

- [ ] F012 [US1] Update seller sign-in and sign-up pages for Google
  - File: `../hivespace.web/apps/seller/src/pages/Auth/SignInPage.vue` and `../hivespace.web/apps/seller/src/pages/Auth/SignUpPage.vue`
  - Add shared Google provider button to both pages using `useAuth().startGoogleAuth({ app: 'seller', returnUrl, culture })`.
  - Preserve existing password sign-in/sign-up forms, error mapping, and return URL behavior.
  - Do not imply Google sign-up creates seller access; new Google users remain normal users until existing onboarding completes.
  - Acceptance: seller sign-in and sign-up start the same Google challenge endpoint with `app=seller`.

- [ ] F013 [US1] [US2] Update seller auth i18n
  - File: `../hivespace.web/apps/seller/src/i18n/locales/en/auth.json` and `../hivespace.web/apps/seller/src/i18n/locales/vi/auth.json`
  - Add seller auth keys for Google provider action, link confirmation, password confirmation, cancellation, password sign-in, password reset, use another Google account, wrong password, blocked account, missing/unverified Google email, and normal-user seller onboarding expectation.
  - Keep English and Vietnamese key shape synchronized.
  - Do not place seller-specific onboarding copy in shared auth package.
  - Acceptance: seller Google auth and linking UI has complete localized copy in both locales.

## Admin App

### Verify

- [ ] F014 [US4] Verify admin app excludes Google sign-in
  - File: `../hivespace.web/apps/admin/src/pages/` and `../hivespace.web/apps/admin/src/router/index.ts`
  - Confirm admin sign-in pages/routes do not import `GoogleIcon`, Google auth button, or call `startGoogleAuth`.
  - Confirm admin redirects still use the supported admin sign-in flow and do not start `app=admin` challenges.
  - Do not add admin Google copy or Google routes.
  - Acceptance: searching admin source for Google auth terms finds no user-facing Google sign-in entry point.

## Frontend Environment

### Verify

- [ ] F015 [US1] Verify frontend gateway base URL and allowed origins
  - File: `../hivespace.web/apps/buyer/.env*`, `../hivespace.web/apps/seller/.env*`, and frontend source config files if present
  - Confirm buyer/seller use a `VITE_GATEWAY_BASE_URL` that can reach ApiGateway and that the same frontend origins are allowed in IdentityService source config.
  - Keep admin environment unchanged for Google auth.
  - Do not put Google OAuth client secrets in frontend environment files; browser apps should only know the gateway base URL and non-secret routing context.
  - Do not commit environment files containing secrets.
  - Keep frontend environment work inside frontend source/local developer configuration.
  - Acceptance: buyer and seller can start Google challenge through the gateway in local development.
