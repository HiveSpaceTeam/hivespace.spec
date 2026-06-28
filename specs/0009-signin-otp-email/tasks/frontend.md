# Frontend Tasks

## Shared Package (`@hivespace/shared`)

### Create

- [ ] F002 [US1] Create shared OTP auth request and verify types plus service client
  - File: `../hivespace.web/packages/shared/src/features/auth/otp-auth.types.ts`, `../hivespace.web/packages/shared/src/features/auth/otp-auth.service.ts`, `../hivespace.web/packages/shared/src/features/auth/index.ts`
  - Add request/response types for OTP request and verify using `challengeToken`, `expiresAt`, `canResendAt`, and `redirectUrl`.
  - Add a narrow shared service with `requestOtp` and `verifyOtp` methods only.
  - Do not create shared UI components or a generic OTP frontend framework in `@hivespace/shared`.
  - Acceptance: buyer and seller apps can import one shared auth client and type surface for the OTP endpoints.

- [ ] F003 [US2] [US3] Create shared countdown helper only if current shared utilities are insufficient
  - File: `../hivespace.web/packages/shared/src/composables/useOtpTimer.ts` or the existing shared countdown/composable file that should be extended
  - Inspect existing cooldown/countdown utilities first and extend one if it already fits the OTP flow.
  - Support countdown state for both `expiresAt` and `canResendAt`.
  - Do not duplicate an existing shared timer helper under a new name when extension is enough.
  - Acceptance: buyer and seller apps share one countdown implementation path or explicitly reuse an existing helper instead of duplicating logic.

## Buyer App

### Create

- [ ] F004 [US1] [US3] Create buyer OTP store tests for FR-001a, FR-004b, FR-004c, and FR-009
  - File: `../hivespace.web/apps/buyer/src/stores/otp-auth.store.test.ts`
  - Test: `should store challengeToken expiresAt and canResendAt after requesting OTP`
  - Test: `should block code entry when no active challenge exists`
  - Test: `should reset OTP state when returning to password sign in`
  - Test: `should replace prior challenge state after resend succeeds`
  - Mock the shared OTP auth service and router/navigation dependencies.
  - Acceptance: tests compile and fail before `F005`, `F006`, and `F008` are implemented.

- [ ] F005 [US1] Create buyer OTP store and request page
  - File: `../hivespace.web/apps/buyer/src/stores/otp-auth.store.ts`, `../hivespace.web/apps/buyer/src/pages/Auth/OtpSignInPage.vue`, and related app exports/imports
  - Store email, `challengeToken`, `expiresAt`, `canResendAt`, loading state, and request/verify error state locally to the buyer app.
  - Route OTP requests through `@hivespace/shared` service methods rather than direct HTTP calls from the page.
  - Keep all user-facing text behind i18n keys.
  - Acceptance: buyer users can open the OTP request screen and submit an email through the store from the existing sign-in experience.

- [ ] F006 [US2] Create buyer OTP code-entry page and route guard for FR-001b and FR-008
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/OtpCodeEntryPage.vue`, `../hivespace.web/apps/buyer/src/router/index.ts`, and any route-meta or guard helpers the buyer app already uses
  - Add six single-digit inputs with auto-advance and backspace behavior.
  - Guard the code-entry route when no active `challengeToken` is present in store state.
  - Redirect with backend-provided `redirectUrl` when present; otherwise fall back to the buyer home route.
  - Acceptance: direct navigation without challenge state is rejected and successful OTP verify follows the validated redirect behavior.

- [ ] F007 [US1] Create buyer sign-in page OTP entry test for FR-001
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignInPage.test.ts`
  - Test: `should render an OTP sign-in entry action next to password sign-in`
  - Test: `should navigate to the buyer OTP request flow when the OTP entry action is used`
  - Preserve existing password sign-in affordances in the rendered page assertions.
  - Acceptance: tests compile and fail before `F008` is implemented.

### Update

- [ ] F008 [US1] Update buyer sign-in page to expose OTP sign-in entry for FR-001
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignInPage.vue`
  - Add a visible OTP sign-in action near the password sign-in form.
  - Preserve current password sign-in submission flow, validation, and layout behavior.
  - Acceptance: buyer sign-in offers OTP sign-in without regressing password sign-in.

- [ ] F009 [US2] [US3] Update buyer OTP code-entry UX for countdown, resend, and expired-code state
  - File: `../hivespace.web/apps/buyer/src/stores/otp-auth.store.ts`, `../hivespace.web/apps/buyer/src/pages/Auth/OtpCodeEntryPage.vue`
  - Disable resend until `canResendAt`, show live expiry countdown from `expiresAt`, and surface expired/invalid code feedback.
  - Clear or replace stale challenge state after resend.
  - Do not add polling or extra round trips just to drive countdown behavior.
  - Acceptance: buyer OTP code entry satisfies FR-005 and FR-009 with client-side countdown state driven by the request response.

- [ ] F010 [US1] Update buyer i18n resources for OTP request and code-entry copy
  - File: `../hivespace.web/apps/buyer/src/i18n/locales/en/*.json`, `../hivespace.web/apps/buyer/src/i18n/locales/vi/*.json`, and buyer i18n registration files if new namespaces are added
  - Add keys for OTP entry point, request state, six-digit code entry, resend, expiry countdown, invalid/expired code, and back-to-password navigation.
  - Keep generic reusable text in `common` only when it is truly shared shell copy.
  - Acceptance: all new buyer OTP copy exists in both English and Vietnamese with no hardcoded strings left in the OTP flow.

## Seller App

### Create

- [ ] F011 [US1] [US3] Create seller OTP store tests for FR-001a, FR-004b, FR-004c, and FR-009
  - File: `../hivespace.web/apps/seller/src/stores/otp-auth.store.test.ts`
  - Test: `should store challengeToken expiresAt and canResendAt after requesting OTP`
  - Test: `should block code entry when no active challenge exists`
  - Test: `should reset OTP state when returning to password sign in`
  - Test: `should replace prior challenge state after resend succeeds`
  - Mirror buyer expectations while using seller routing and store conventions.
  - Acceptance: tests compile and fail before `F012`, `F013`, and `F015` are implemented.

- [ ] F012 [US1] Create seller OTP store and request page
  - File: `../hivespace.web/apps/seller/src/stores/otp-auth.store.ts`, `../hivespace.web/apps/seller/src/pages/Auth/OtpSignInPage.vue`, and related app exports/imports
  - Reuse the shared OTP service and keep seller-specific route outcomes local to the seller app.
  - Store challenge token and resend/countdown state in the seller app rather than global shared runtime state.
  - Acceptance: seller users can enter the OTP request flow from the seller sign-in surface and request a code through the store.

- [ ] F013 [US2] Create seller OTP code-entry page and route guard for FR-001b and FR-008
  - File: `../hivespace.web/apps/seller/src/pages/Auth/OtpCodeEntryPage.vue`, `../hivespace.web/apps/seller/src/router/index.ts`, and seller route helpers as needed
  - Mirror buyer code-entry behavior with seller-specific redirect fallback logic.
  - Keep async request/verify logic in the store instead of the page component.
  - Acceptance: seller OTP code entry enforces challenge-state routing and successful verify follows seller redirect rules.

- [ ] F014 [US1] Create seller sign-in page OTP entry test for FR-001
  - File: `../hivespace.web/apps/seller/src/pages/Auth/SignInPage.test.ts`
  - Test: `should render an OTP sign-in entry action next to password sign-in`
  - Test: `should navigate to the seller OTP request flow when the OTP entry action is used`
  - Preserve existing seller password sign-in affordances and onboarding-related links in the rendered page assertions.
  - Acceptance: tests compile and fail before `F016` is implemented.

### Update

- [ ] F015 [US2] [US3] Update seller OTP code-entry UX and i18n for resend/countdown flows
  - File: `../hivespace.web/apps/seller/src/stores/otp-auth.store.ts`, `../hivespace.web/apps/seller/src/pages/Auth/OtpCodeEntryPage.vue`, `../hivespace.web/apps/seller/src/i18n/locales/en/*.json`, `../hivespace.web/apps/seller/src/i18n/locales/vi/*.json`
  - Add resend cooldown, expiry countdown, invalid/expired code feedback, and translated copy in both locales.
  - Keep seller app behavior aligned with buyer UX while preserving seller-specific navigation text and redirect outcomes.
  - Acceptance: seller OTP code entry satisfies FR-005 and FR-009 with complete English and Vietnamese coverage.

- [ ] F016 [US1] Update seller sign-in page to expose OTP sign-in entry for FR-001
  - File: `../hivespace.web/apps/seller/src/pages/Auth/SignInPage.vue`
  - Add a visible OTP sign-in action near the current password sign-in form.
  - Preserve seller password sign-in behavior and any onboarding-related redirects already present.
  - Acceptance: seller sign-in exposes OTP sign-in without regressing the existing password flow.
