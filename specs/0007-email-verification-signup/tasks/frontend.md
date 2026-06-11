# Frontend Tasks

## Shared Package - `packages/shared`

### Update

- [ ] F001 [US1] Update `auth-session.ts`
  - File: `../hivespace.web/packages/shared/src/types/auth-session.ts`, `../hivespace.web/packages/shared/src/features/auth/account-session.service.ts`, `../hivespace.web/packages/shared/src/composables/useAuth.ts`, `../hivespace.web/packages/shared/src/features/auth/index.ts`
  - Introduce a dedicated registration result type for `POST /api/v1/accounts/register` instead of reusing `SessionResponse`, and model it without a server-supplied `message` field.
  - Keep `SessionResponse` unchanged for login, refresh, and Google-link confirmation; only the register flow should stop applying a browser session in `useAuth.register(...)`.
  - Ensure the shared service already tolerates `204 No Content` for resend/verify and does not try to parse bodies for those success paths.
  - Acceptance: shared auth code distinguishes registration results from session results, and `useAuth.register(...)` no longer mutates authenticated session state on success.

## Buyer App - `apps/buyer`

### Update

- [ ] F002 [US1] Update `SignUpPage.vue`
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignUpPage.vue`
  - Change the buyer registration page to consume the new register result shape and route to the buyer verification-sent flow instead of using `session.redirectTo`.
  - Render buyer confirmation text from local i18n/resources rather than any register-response `message`.
  - Preserve Google sign-up behavior and existing validation UX.
  - Do not assume registration authenticates the user or that a CSRF cookie is available after registration.
  - Acceptance: buyer sign-up succeeds without logging the user in and routes to buyer verification guidance.

### Create

- [ ] F003 [US1][US2][US3] Create buyer verify-email flow
  - File: `../hivespace.web/apps/buyer/src/pages/VerifyEmailPage.vue`, `../hivespace.web/apps/buyer/src/pages/VerifyEmailCallbackPage.vue`, `../hivespace.web/apps/buyer/src/router/index.ts`, `../hivespace.web/apps/buyer/src/i18n/index.ts`, `../hivespace.web/apps/buyer/src/i18n/locales/en/verifyEmail.json`, `../hivespace.web/apps/buyer/src/i18n/locales/vi/verifyEmail.json`, `../hivespace.web/apps/buyer/src/i18n/locales/en/verifyEmailCallback.json`, `../hivespace.web/apps/buyer/src/i18n/locales/vi/verifyEmailCallback.json`
  - Add an anonymous buyer verification-sent page that can resend by email through the new resend endpoint and a callback page that submits the verification token, treats `204 No Content` as success, and routes the user to sign-in with local success copy.
  - Wire `/verify-email` and `/verify-email-callback` routes with `allowAnonymous` and buyer-appropriate return-url defaults.
  - Keep copy in dedicated module-owned i18n files; do not overload `common.json` with feature-specific verification states.
  - Acceptance: buyer users can register, resend verification anonymously, complete the callback flow, and reach sign-in without any authenticated-session dependency.

## Seller App - `apps/seller`

### Update

- [ ] F004 [US1] Update `SignUpPage.vue`
  - File: `../hivespace.web/apps/seller/src/pages/Auth/SignUpPage.vue`
  - Change seller registration to consume the new register result shape and route to `/verify-email` without assuming a browser session exists.
  - Render seller confirmation text from local i18n/resources rather than any register-response `message`.
  - Preserve Google sign-up and return-url normalization behavior where it does not depend on immediate session issuance.
  - Do not call `session.redirectTo` from the register response anymore.
  - Acceptance: seller sign-up lands on the seller verification screen after a successful pending registration instead of an authenticated route.

- [ ] F005 [US2][US3] Create dedicated seller verify-email pages while preserving `EmailAction*`
  - File: `../hivespace.web/apps/seller/src/services/account.service.ts`, `../hivespace.web/apps/seller/src/stores/account.store.ts`, `../hivespace.web/apps/seller/src/types/account.types.ts`, `../hivespace.web/apps/seller/src/pages/VerifyEmailPage.vue`, `../hivespace.web/apps/seller/src/pages/VerifyEmailCallbackPage.vue`, `../hivespace.web/apps/seller/src/router/index.ts`, `../hivespace.web/apps/seller/src/i18n/index.ts`, `../hivespace.web/apps/seller/src/i18n/locales/en/verifyEmail.json`, `../hivespace.web/apps/seller/src/i18n/locales/vi/verifyEmail.json`, `../hivespace.web/apps/seller/src/i18n/locales/en/verifyEmailCallback.json`, `../hivespace.web/apps/seller/src/i18n/locales/vi/verifyEmailCallback.json`
  - Add seller `VerifyEmailPage.vue` and `VerifyEmailCallbackPage.vue` that follow the buyer verification flow, while leaving `EmailActionPage.vue` and `EmailActionCallbackPage.vue` unchanged.
  - Route seller `/verify-email` and `/verify-email-callback` to the new dedicated verify pages, use anonymous auth-style route metadata, and keep seller-appropriate return-url defaults.
  - Replace the current authenticated resend flow with the anonymous resend contract by email, treat resend success as `204 No Content`, and remove any dependency on `currentUser.profile.email` or an authenticated session to trigger resend.
  - Update callback verification to treat `204 No Content` as success, stop calling `refreshSession('seller')`, and route users to sign-in with local success messaging after verification completes.
  - Keep seller post-verification onboarding intact by redirecting from the next sign-in/session refresh into `/register-seller`, not by authenticating during verification.
  - Acceptance: seller verify pages work for unauthenticated pending users, the public seller routes remain compatible, `EmailAction*` files stay unchanged, and successful verification no longer tries to refresh a non-existent session.

- [ ] F006 [US3] Update pending-login recovery handling
  - File: `../hivespace.web/apps/buyer/src/pages/Auth/SignInPage.vue`, `../hivespace.web/apps/seller/src/pages/Auth/SignInPage.vue`, `../hivespace.web/apps/buyer/src/i18n/locales/en/auth.json`, `../hivespace.web/apps/buyer/src/i18n/locales/vi/auth.json`, `../hivespace.web/apps/seller/src/i18n/locales/en/auth.json`, `../hivespace.web/apps/seller/src/i18n/locales/vi/auth.json`
  - Map the stable pending-verification exception code/message to a frontend recovery path that sends users to the appropriate verify-email route with the submitted email preserved where needed.
  - Add/adjust auth translation keys for pending-verification, resend guidance, expired-link recovery, and duplicate pending-account registration messaging.
  - Do not change Google-link or standard invalid-credential handling outside the new pending-account branch.
  - Acceptance: buyer and seller sign-in pages can route pending users into verification recovery while all other auth failures still follow the existing error mapping.
