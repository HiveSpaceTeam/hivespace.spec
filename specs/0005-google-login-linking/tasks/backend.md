# Backend Tasks

## HiveSpace.IdentityService

### Verify

- [ ] B001 Verify `../hivespace.microservice` implementation context
  - File: `../hivespace.microservice/AGENTS.md`
  - Read source repo instructions, then inspect current IdentityService source paths before editing: `HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`, `HiveSpace.IdentityService.Api/Extensions/HostingExtensions.cs`, `HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`, `HiveSpace.IdentityService.Core/Features/AccountSessions/`, and `HiveSpace.IdentityService.Core/DomainModels/ApplicationUser.cs`.
  - Run required GitNexus impact analysis before editing existing symbols such as `MapIdentityEndpoints`, `ConfigurePipelineAsync`, `AddAppRazorUi`, `AddAppAuthentication`, `SignInCommandHandler`, and session/token helpers.
  - Do not change unrelated services or shared contracts during source inspection.
  - Acceptance: implementation owner can name the exact existing symbols/files to change and has recorded impact-analysis blast radius before code edits.

### Create

- [ ] B002 [US1] Create Google auth configuration model
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Configs/GoogleExternalAuthOptions.cs`
  - Include keys for `ClientId`, `ClientSecret`, `CallbackPath`, allowed buyer/seller frontend origins, pending-link lifetime, and provider display name `Google`.
  - Add validation for missing client ID, missing secret, invalid callback path, and empty frontend origin list during startup or provider challenge setup.
  - Do not allow `admin` as an app context or frontend origin for this feature.
  - Acceptance: IdentityService can bind and validate Google options without exposing secrets in logs or responses.

- [ ] B003 [US1] [US2] Create Google external auth request and response DTOs
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Dtos/GoogleExternalAuthDtos.cs`
  - Include `GoogleChallengeRequest` fields `App`, nullable `ReturnUrl`, nullable `Culture`; `ConfirmGoogleLinkRequest` fields `ConsentAccepted`, `Password`, `App`, nullable `ReturnUrl`, nullable `Culture`; and a safe error/result representation for callback redirects.
  - Match `contracts/browser-auth-api.md`: challenge uses `buyer` or `seller`; confirm link returns the existing `SessionResponse` shape plus `redirectTo` when appropriate.
  - Do not include provider access tokens, refresh tokens, or Google provider user IDs in public DTOs.
  - Acceptance: DTOs compile and represent all contract inputs without browser-readable provider secrets.

- [ ] B004 [US2] Create pending Google link state service
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Services/IPendingGoogleLinkStore.cs` and implementation file in the same feature area or API service layer if cookie-backed
  - Store provider name, provider key, verified email, target account ID, app, safe return URL, culture, expiry, and a CSRF/link token in server-side or HttpOnly state.
  - Include methods to create, read/validate, clear, and expire pending-link state.
  - Do not store provider keys or tokens in readable JSON, query strings, local storage, or session storage.
  - Acceptance: expired, missing, cancelled, or mismatched pending-link state cannot produce a durable Google login.

- [ ] B005 [US1] Create Google challenge command and handler
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Commands/StartGoogleChallenge/StartGoogleChallengeCommand.cs` and handler/validator files in the same folder
  - Validate `App` is `buyer` or `seller`, normalize culture, sanitize return URL against allowed frontend origins, and initiate ASP.NET Authentication Google challenge with callback state.
  - Reuse existing safe-return behavior where available; otherwise keep validation local to IdentityService auth flow.
  - Do not accept `admin`, arbitrary absolute URLs, or old `/Account/**` return targets.
  - Acceptance: invalid app or unsafe return URL returns a safe `400`; valid buyer/seller requests redirect to Google.

- [ ] B006 [US1] [US2] [US3] Create Google callback command and handler
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Commands/CompleteGoogleCallback/CompleteGoogleCallbackCommand.cs` and handler/validator files in the same folder
  - Use `SignInManager`/`UserManager` external-login APIs to read Google login info, require Google verified email for new account creation or link matching, and durable-link by provider key.
  - Implement outcomes from `contracts/browser-auth-api.md`: already linked signs in, no matching local password account follows normal Google sign-in/sign-up, matching unlinked password account creates pending-link state and redirects to frontend confirmation.
  - Enforce account status, lockout, deleted/disabled restrictions, and buyer/seller app context before issuing a session.
  - Do not auto-link an existing password account based only on email match, and do not create or sign in a separate same-email Google-only account while a matching unlinked local password account exists.
  - Acceptance: callback covers linked, new-user, pending-link, duplicate-same-email prevention, missing/unverified email, blocked account, and unsafe return outcomes.

- [ ] B007 [US2] Create confirm Google link command and handler
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Commands/ConfirmGoogleLink/ConfirmGoogleLinkCommand.cs` and handler/validator files in the same folder
  - Validate pending-link state, explicit `ConsentAccepted == true`, app, safe return URL, account status, existing password, lockout behavior, and provider-key uniqueness before adding the Google login.
  - Reject missing or false consent with `400` before durable linking; do not treat password entry alone as consent.
  - On success, call ASP.NET Identity external login APIs, mark `EmailConfirmed = true` when Google verified the same email, clear pending state, issue standard browser session cookies and CSRF token, and return `SessionResponse`.
  - Do not bypass existing failed-password/lockout handling, create a link when consent/password validation fails, or fall through to duplicate same-email Google account creation.
  - Acceptance: explicit consent plus correct password links and signs in; missing/false consent, wrong password, blocked account, expired state, or provider conflict does not link or create a duplicate account.

- [ ] B008 [US2] Create cancel Google link command and handler
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/ExternalLogins/Commands/CancelGoogleLink/CancelGoogleLinkCommand.cs` and handler/validator files in the same folder
  - Validate temporary pending-link state and CSRF/link token, then clear state.
  - Return no content and do not issue session cookies.
  - Do not remove an already durable Google login, mutate the matched password account, issue a session, or create a duplicate same-email Google-only account.
  - Acceptance: cancelling leaves the existing account unchanged, creates no duplicate account, and future link attempts require a fresh Google callback.

### Update

- [ ] B016 [US1] [US2] Update backend IdentityService Google OAuth settings
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/appsettings.json`, local-only unpushed `appsettings.Development.json`, or .NET user-secrets as appropriate.
  - Add the committed non-secret shape for Google provider config: client ID placeholder, client secret placeholder or secret-provider reference, callback path `/api/v1/accounts/external/google/complete`, pending-link lifetime, and allowed buyer/seller frontend origins.
  - Allow real Google OAuth client ID/client secret values only in local developer configuration that will not be pushed to remote, such as unpushed local appsettings changes, `appsettings.Development.json`, or .NET user-secrets.
  - Keep local development values compatible with `http://localhost:5174` seller and `http://localhost:5175` buyer unless the source repo config uses different ports.
  - Do not commit real Google client secrets, provider tokens, or frontend files containing OAuth secrets.
  - Keep runtime settings work inside backend source/local developer configuration.
  - Acceptance: IdentityService can bind expected Google config keys locally with secrets supplied through safe local configuration.

- [ ] B009 [US1] [US2] [US3] Create Google external auth endpoint mapping
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/GoogleExternalAuthEndpoints.cs`
  - Create a new Minimal API endpoint file with `MapGoogleExternalAuthEndpoints(this IEndpointRouteBuilder app)`.
  - Map `GET /api/v1/accounts/external/google/challenge`, `GET /api/v1/accounts/external/google/complete`, `POST /api/v1/accounts/external/google/link`, and `DELETE /api/v1/accounts/external/google/link`.
  - Keep `IdentityEndpoints.cs` focused on existing password login, registration, session refresh, logout, and email verification endpoints unless shared request records must move.
  - Use `AllowAnonymous()` for challenge/callback; require temporary pending-link state plus CSRF/link token for link confirm/cancel.
  - Set endpoint names, summaries, tags, and response behavior aligned with `contracts/browser-auth-api.md`.
  - Do not map any Google endpoint under `/Account/**` or `/identity/**`.
  - Acceptance: OpenAPI/route inspection shows the four `/api/v1/accounts/external/google/**` routes are mapped from the new endpoint file and no provider secrets appear in responses.

- [ ] B010 [US1] [US2] [US3] Update IdentityService endpoint registration for Google routes
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/HostingExtensions.cs`
  - Add `app.MapGoogleExternalAuthEndpoints()` alongside `app.MapIdentityEndpoints()` and `app.MapAdminIdentityEndpoints()`.
  - Preserve the existing account endpoint mapper for non-Google account/session/email routes.
  - Do not add a catch-all `/Account/**`, `/identity/**`, or compatibility redirect route while registering the Google endpoint mapper.
  - Acceptance: IdentityService startup maps both existing account endpoints and the new Google external-auth endpoint file without changing the planned public paths.

- [ ] B011 [US1] [US3] Update IdentityService service registration for Google auth
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`
  - Register Google external authentication with ASP.NET Identity using centrally bound config, callback path, claims mapping needed for provider key/email/email_verified, and pending-link/session services.
  - Preserve existing JWT bearer auth, IdentityServer, token cookie service, CSRF token service, MassTransit, and localization registration.
  - Do not add NuGet versions to `.csproj`; if a package is required, put the version in `Directory.Packages.props`.
  - Acceptance: IdentityService builds and Google challenge can resolve its authentication scheme.

- [ ] B012 [US1] [US2] [US3] Update session issuance reuse for Google flows
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/AccountSessionHandlerBase.cs` and related session service files under `Features/AccountSessions/`
  - Reuse the same browser session cookie, refresh cookie, CSRF token, `SessionUser`, expiry, and `redirectTo` behavior used by password login/register.
  - Ensure new Google accounts publish/reuse the existing `IdentityUserCreatedIntegrationEvent` path and already linked accounts preserve roles/store claims.
  - Do not return access or refresh token values in JSON.
  - Acceptance: password login/register/session refresh behavior remains unchanged and Google success responses match `SessionResponse` semantics.

### Remove

- [ ] B013 [US4] Remove legacy IdentityService Razor Pages and static web asset folders
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/` and `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/wwwroot/`
  - Delete the full `Pages/` tree including Account, ExternalLogin, Consent, Device, Ciba, Diagnostics, Grants, Redirect, shared partials, page models, suppressions, and layout files.
  - Delete the full `wwwroot/` tree including legacy CSS, JS, images, Bootstrap/jQuery libraries, favicon/logo assets, and `wwwroot/localization`.
  - Preserve required non-page IdentityServer protocol behavior under `/.well-known/**` and `/connect/**`.
  - Before deleting `wwwroot/localization`, confirm no non-page API endpoint depends on `LocalizationService`; remove or replace that service in `B015` if it is only supporting deleted hosted pages.
  - Do not add compatibility redirects for removed page URLs.
  - Acceptance: source tree no longer contains `HiveSpace.IdentityService.Api/Pages/` or `HiveSpace.IdentityService.Api/wwwroot/`, and old page URLs cannot render user-facing Razor pages or legacy static assets.

- [ ] B014 [US4] Remove account compatibility endpoint surface
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/AccountCompatibilityEndpoints.cs`
  - Delete the file and remove `MapAccountCompatibilityEndpoints()` from `Extensions/HostingExtensions.cs`.
  - Remove `/Account/Login`, `/Account/Register`, and `/Account/Logout` frontend redirect behavior.
  - Do not replace this with new compatibility routes during development.
  - Acceptance: project builds with no `AccountCompatibilityEndpoints` references and `/Account/**` is unsupported legacy URL space.

- [ ] B015 [US4] Remove Razor Pages startup wiring
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/HostingExtensions.cs` and `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`
  - Remove `AddAppRazorUi()` or narrow it to only required non-page session dependencies; remove `AddRazorPages()`, `MapRazorPages().RequireAuthorization()`, `app.UseStaticFiles()`, and unused static file/MVC/page-only service registrations.
  - Remove `LocalizationService`, `ILocalizationService`, `AddLocalizationServices()`, and `wwwroot/localization` resource loading if no non-page endpoint uses them after deleting hosted pages; keep only culture middleware/options that are still needed by API behavior.
  - Keep distributed cache/session only if the Google pending-link implementation or existing cookies still require it.
  - Do not break `UseIdentityServer()`, `UseAuthentication()`, `UseAuthorization()`, health checks, Swagger/Scalar, or account/admin minimal endpoints.
  - Acceptance: IdentityService starts/builds without Razor Pages, static-file middleware, or deleted `wwwroot` dependencies, and protocol endpoints remain available.

## HiveSpace.ApiGateway

### Verify

- [ ] B017 [US1] [US4] Verify existing account route covers Google endpoints
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`
  - Confirm `/api/v1/accounts/{**catch-all}` or equivalent forwards to IdentityService and that `/.well-known/**`, `/connect/**`, and `/Account/**` remain direct/unsupported as documented.
  - Update only if the existing route does not forward `/api/v1/accounts/external/google/**`.
  - Keep any required gateway route change inside the backend source repo.
  - Do not add business logic or account-link decisions to ApiGateway.
  - Acceptance: gateway route table forwards all four Google account endpoints to IdentityService and does not route `/Account/**` as a supported public API.
