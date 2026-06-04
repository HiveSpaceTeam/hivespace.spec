# Verification Tasks

## Backend

### Verify

- [ ] V001 [US1] Verify IdentityService build
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj`
  - Run from `../hivespace.microservice`: `dotnet build src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/HiveSpace.IdentityService.Api.csproj`.
  - Confirm Razor Pages removal, `wwwroot` deletion, static-file/localization cleanup, Google auth registration, endpoint mapping, and session helpers compile together.
  - Acceptance: build succeeds with no new errors; known baseline warnings are documented if present.

- [ ] V002 [US4] Verify ApiGateway build if route config changed
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj`
  - Run from `../hivespace.microservice`: `dotnet build src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/HiveSpace.YarpApiGateway.csproj` if gateway source/config changed.
  - Confirm `/api/v1/accounts/**` still forwards to IdentityService and `/Account/**` is not added as a supported gateway route.
  - Acceptance: build succeeds when gateway changes are present, or task records that no gateway build was needed because only existing catch-all routing was verified.

## Frontend

### Verify

- [ ] V003 [US1] Verify shared frontend package
  - File: `../hivespace.web/packages/shared/`
  - Run the narrowest available shared package check, preferably `pnpm --filter @hivespace/shared build` from `../hivespace.web`.
  - Confirm `useAuth`, shared auth service, exported types, and Google button/component exports compile.
  - Acceptance: shared package build/check passes or only known pre-existing baseline issues are reported with no introduced Google-auth errors.

- [ ] V004 [US1] Verify buyer app auth flow build checks
  - File: `../hivespace.web/apps/buyer/`
  - Run buyer `pnpm lint` and `pnpm type-check` from the app directory or equivalent workspace filter available in the repo.
  - Confirm sign-in, sign-up, Google link page, router, and i18n changes compile and lint.
  - Acceptance: buyer checks pass or introduced issues are fixed; no hardcoded user-facing Google auth text remains.

- [ ] V005 [US1] Verify seller app auth flow build checks
  - File: `../hivespace.web/apps/seller/`
  - Run seller `pnpm lint` and `pnpm type-check` from the app directory or equivalent workspace filter available in the repo.
  - Account for documented seller/admin baseline lint/type-check failures; fix only errors introduced by this feature.
  - Acceptance: seller Google auth changes introduce no new lint/type-check failures beyond documented baseline.

- [ ] V006 [US4] Verify admin app has no Google entry point
  - File: `../hivespace.web/apps/admin/src/`
  - Search admin source for `Google`, `startGoogleAuth`, `external/google`, and Google auth i18n keys.
  - Manually inspect admin sign-in route/page if search results appear for shared exports only.
  - Acceptance: admin app does not render Google sign-in and cannot start an `app=admin` challenge.

## Manual Quickstart

### Verify

- [ ] V007 [US1] [US3] Verify new and already-linked Google sign-in scenarios
  - File: `specs/0005-google-login-linking/quickstart.md`
  - With local infrastructure, IdentityService, ApiGateway, buyer app, and seller app running, complete Google sign-in from buyer sign-in, buyer sign-up, seller sign-in, and seller sign-up.
  - Verify no matching local password account follows normal Google path; already linked Google identity signs in without link prompt; new seller-started Google user is normal user and follows seller onboarding guards.
  - Confirm successful responses set HttpOnly cookies and no access/refresh token values are readable in JSON.
  - Acceptance: US1 and US3 independent tests pass for buyer and seller.

- [ ] V008 [US2] Verify existing password account linking scenarios
  - File: `specs/0005-google-login-linking/quickstart.md`
  - Create/use an existing local password buyer or seller account with no Google link, then start Google with the same verified email.
  - Verify pending link prompt appears; missing or unchecked explicit consent cannot submit or returns a safe validation error; declined consent leaves account unchanged, creates no duplicate same-email Google-only account, and offers password sign-in, password reset, or another Google account; wrong password leaves account unchanged, creates no duplicate account, and applies lockout behavior; explicit consent plus correct password links Google, marks email verified, signs in, and future Google sign-in skips prompt.
  - Confirm expired or abandoned pending-link state cannot be confirmed later.
  - Acceptance: US2 independent test passes and no account link or duplicate same-email account occurs without explicit consent plus correct password.

- [ ] V009 [US4] Verify legacy IdentityService page URL removal
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/`
  - Visit or request old user-facing URLs including `/Account/Login`, `/Account/Register`, `/Account/Logout`, old external-login pages, consent, grants, device, diagnostics, and CIBA page URLs.
  - Confirm they do not render legacy HiveSpace user-facing Razor pages or serve legacy `wwwroot` assets, and compatibility redirects are gone.
  - Confirm `/.well-known/**` and `/connect/**` required protocol endpoints still work.
  - Acceptance: US4 independent test passes without breaking IdentityServer protocol discovery/authorization endpoints.

## Final Audit

### Verify

- [ ] V010 Verify final source and catalog consistency
  - File: `../hivespace.microservice/`, `../hivespace.web/`, `shared/api-catalog.md`, `shared/event-catalog.md`, and `services/identity-service/`
  - Search for stale references to deleted IdentityService `Pages/`, `wwwroot/`, `UseStaticFiles`, `LocalizationService`, `wwwroot/localization`, `AccountCompatibilityEndpoints`, old `/Account/**` compatibility redirects, browser-readable token returns, `app=admin` Google challenge usage, and duplicated Google endpoint paths.
  - Run GitNexus detect-changes in source repos before committing code changes, per source repo instructions.
  - Do not stage JSON files automatically in backend/frontend repos; follow each repo's commit guardrails.
  - Acceptance: final diff matches the feature scope and no stale legacy account-page, static asset, localization-resource, or unsafe Google-token behavior remains.
