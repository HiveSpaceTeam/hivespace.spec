# Backend Tasks

## Source Repo Setup And Impact Analysis

### Verify

- [ ] B001 Verify backend repo instructions and GitNexus impact scope
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.microservice/CLAUDE.md`
  - Read both instruction files before editing backend source.
  - Run GitNexus impact/context checks before modifying existing symbols in IdentityService and ApiGateway, including `MapIdentityEndpoints`, `ConfigurePipelineAsync`, `AddAppIdentity`, `AddAppAuthentication`, and ApiGateway `Program.cs`.
  - Record direct callers and risk in the implementation notes before editing; do not ignore HIGH or CRITICAL impact.
  - Acceptance: backend implementation notes identify impacted symbols/callers and confirm repo instructions were followed.

## IdentityService

### Create

- [ ] B002 [US1] [US2] Create account session DTOs
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Dtos/AccountSessionDtos.cs`
  - Include request records: `SignInRequest(string Email, string Password, string App, string? ReturnUrl, string? Culture)`, `RegisterAccountRequest(string Email, string Password, string ConfirmPassword, string? FullName, string App, string? ReturnUrl, string? Culture)`, `RefreshSessionRequest(string App)`, `SignOutRequest(string? RedirectTo)`.
  - Include response records: `SessionResponse(SessionUser User, DateTimeOffset ExpiresAt, DateTimeOffset RefreshExpiresAt, string CsrfToken, string? RedirectTo)` and `SessionUser(Guid UserId, string Email, string? DisplayName, IReadOnlyCollection<string> Roles, bool EmailVerified, string AccountStatus)`.
  - Do not include access token, refresh token, Duende persisted grant, password hash, profile/address/store fields, or UserService-owned data.
  - Acceptance: DTOs compile and match `contracts/browser-auth-api.md` request/response fields.

- [ ] B003 [US1] [US2] Create account session commands and validators
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/*`
  - Create CQRS command records and validators for `SignInCommand`, `RegisterAccountCommand`, `RefreshSessionCommand`, and `SignOutCommand`.
  - Validate email format, non-empty password, matching confirmation password, `app` limited to `admin`, `seller`, `buyer`, and safe local/configured frontend `returnUrl`.
  - Reject public admin self-registration in the register validator or handler with `IdentityDomainErrorCode.AccountNotAllowed`.
  - Acceptance: command/validator files compile and validation failures use the existing exception pipeline without leaking account existence.

- [ ] B004 [US1] [US2] [US3] [US4] Create protected session cookie service
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Services/IProtectedSessionCookieService.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Services/ProtectedSessionCookieService.cs`
  - Implement issuance, refresh, and clearing of `__Host-HiveSpace.Auth` with `HttpOnly`, `Secure`, `SameSite=None`, `Path=/`, no `Domain`.
  - Store protected envelope fields from `data-model.md`: `sessionId`, `userId`, access token, refresh handle/token, access expiry, refresh expiry, security stamp when available, and issued time.
  - Use ASP.NET Core Data Protection purpose names shared with ApiGateway; do not expose protected envelope contents in JSON responses.
  - Acceptance: successful login/register/refresh can set cookies and JSON response contains no access or refresh token fields.

- [ ] B005 [US1] [US2] [US3] [US4] Create CSRF token service
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Services/ICsrfTokenService.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Services/CsrfTokenService.cs`
  - Issue signed CSRF token bound to `sessionId`, `issuedAt`, and `expiresAt`.
  - Set readable `HiveSpace.Csrf` cookie with `Secure`, `SameSite=None`, `Path=/`; return the same token in session response for frontend state.
  - Clear the CSRF cookie on logout.
  - Do not use bearer/access token value as the CSRF token.
  - Acceptance: refresh/logout responses can rotate or clear CSRF token and token value is independent of bearer token material.

- [ ] B006 [US1] [US2] [US3] [US4] Extend IdentityService domain error codes for session auth
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Exceptions/IdentityDomainErrorCode.cs`
  - Add `InvalidCredentials` (`IDN6006`), `AccountInactive` (`IDN6007`), `AccountLocked` (`IDN6008`), `AccountNotAllowed` (`IDN6009`), `DuplicateEmail` (`IDN6010`), `InvalidSession` (`IDN6011`), `SessionExpired` (`IDN6012`), and `InvalidReturnUrl` (`IDN6013`) after the existing `IDN6005` values.
  - Map Identity/SignInManager failures to `UnauthorizedException`, `ForbiddenException`, `ConflictException`, or `BadRequestException` using `IdentityDomainErrorCode`; do not add a custom auth error DTO or parallel error response family.
  - Acceptance: endpoint handlers return the existing `ExceptionModel` shape with `errors[].code`, `errors[].messageCode`, and `errors[].source`.

### Update

- [ ] B007 [US1] Update password login behavior into CQRS handler
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/SignInCommandHandler.cs`
  - Move behavior currently in `HiveSpace.IdentityService.Api/Pages/Account/Login/Index.cshtml.cs`: find user by email, account status check, admin/seller/buyer app role checks, lockout-on-failure, `LastLoginAt`, Duende login success/failure events where still applicable, and safe error mapping.
  - On success, issue protected browser session and CSRF token through B004/B005.
  - Do not render Razor pages, redirect to IdentityService UI, or store tokens in script-readable response fields.
  - Acceptance: handler returns `SessionResponse` for valid credentials and standard `IdentityDomainErrorCode` errors for invalid, locked, inactive, or wrong-app accounts.

- [ ] B008 [US2] Update registration behavior into CQRS handler
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RegisterAccountCommandHandler.cs`
  - Move behavior currently in `HiveSpace.IdentityService.Api/Pages/Account/Register/Index.cshtml.cs`: unique email check, `ApplicationUser` creation, password validation, `CreatedAt`/`UpdatedAt`, `IdentityUserCreatedIntegrationEvent` publication via `IIdentityEventPublisher`, and initial session issuance.
  - Use existing outbox-backed publisher policy; publish event before save where existing IdentityService publisher requires it.
  - Do not call UserService synchronously and do not introduce a new registration integration event.
  - Acceptance: one identity account is created, existing profile-creation event is published, and successful response sets session/CSRF cookies.

- [ ] B009 [US3] Update session refresh behavior
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/RefreshSessionCommandHandler.cs`
  - Read and validate protected session cookie, account status, lockout, security stamp when available, refresh expiry, and app role eligibility.
  - Rotate access token/session envelope and CSRF token; update claims so email verification and seller-role propagation can be observed after refresh.
  - Return `SessionExpired`, `AccountInactive`, or `AccountNotAllowed` safely on failure through the standard exception pipeline.
  - Acceptance: reload/near-expiry refresh returns updated `SessionResponse` or safe signed-out error without exposing tokens.

- [ ] B010 [US4] Update session logout behavior
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AccountSessions/Commands/SignOutCommandHandler.cs`
  - Clear `__Host-HiveSpace.Auth` and `HiveSpace.Csrf` cookies and invalidate refresh handle/token where applicable.
  - Keep logout idempotent when the session cookie is missing or already invalid.
  - Do not require a redirect through IdentityService UI.
  - Acceptance: logout returns `204 No Content` and subsequent protected requests are unauthenticated.

- [ ] B011 [US1] [US2] [US3] [US4] Update IdentityService endpoint mapping
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/IdentityEndpoints.cs`
  - Add Minimal API endpoints: `POST /api/v1/accounts/login`, `POST /api/v1/accounts/register`, `POST /api/v1/accounts/session/refresh`, and `POST /api/v1/accounts/logout`.
  - Keep existing email verification endpoints unchanged.
  - Set auth metadata: login/register anonymous; refresh/logout require valid browser session cookie and CSRF validation path.
  - Return contract status codes through the existing `ExceptionModel` error response from `UseHiveSpaceExceptionHandler`.
  - Acceptance: route list contains the four new endpoints and existing email verification routes remain functional.

- [ ] B012 [US1] [US2] [US4] Update old Account URL compatibility
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Endpoints/AccountCompatibilityEndpoints.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/HostingExtensions.cs`
  - Add minimal redirects for `/Account/Login`, `/Account/Register`, and `/Account/Logout`.
  - Resolve target frontend route from safe `returnUrl`, app/client hint, or configured fallback origins for admin/seller/buyer.
  - Reject or ignore unsafe external return URLs and fall back to configured default.
  - Do not depend on Razor Page handlers for these routes.
  - Acceptance: old URLs return redirects to frontend routes and no IdentityService auth UI is rendered.

- [ ] B013 [US1] [US2] [US4] Remove IdentityService Razor auth UI files
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Login/**`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Register/**`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Logout/**`, auth-only CSS under `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/wwwroot/css/`
  - Delete login, registration, and logout Razor Page UI files after B007-B012 are implemented.
  - Remove imports/usings/routes/static assets made unused by deletion.
  - Preserve non-auth protocol pages only if still required by IdentityServer and not user-facing HiveSpace login/register/logout UI.
  - Acceptance: repository search finds no `Pages/Account/Login`, `Pages/Account/Register`, or `Pages/Account/Logout` UI handlers, and app builds without stale Razor references.

### Verify

- [ ] B014 [US1] [US2] [US3] [US4] Verify IdentityService auth contract shape
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/**`
  - Search JSON response DTOs and endpoint handlers for forbidden `access_token`, `refresh_token`, `id_token`, `token`, and `refreshToken` response fields.
  - Confirm `IdentityUserCreatedIntegrationEvent` is still the only profile-creation event used by registration.
  - Acceptance: no account session JSON response exposes access/refresh tokens and event reuse remains unchanged.

## ApiGateway

### Create

- [ ] B015 [US3] Create gateway session forwarding middleware
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/Middleware/SessionForwardingMiddleware.cs`
  - Read `__Host-HiveSpace.Auth`, decrypt/validate protected envelope with shared Data Protection purpose, reject expired/invalid session for protected routes, and attach `Authorization: Bearer <access-token>` before YARP forwarding.
  - Strip `__Host-HiveSpace.Auth` and `HiveSpace.Csrf` from forwarded downstream requests.
  - Leave anonymous login/register requests pass-through without requiring a pre-existing cookie.
  - Acceptance: downstream services receive bearer authorization context while session/CSRF cookies are not forwarded.

- [ ] B016 [US3] [US4] Create gateway CSRF middleware
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/Middleware/CsrfValidationMiddleware.cs`
  - For cookie-authenticated browser requests, require `X-HiveSpace-CSRF` on `POST`, `PUT`, `PATCH`, and `DELETE`.
  - Exempt anonymous `POST /api/v1/accounts/login`, anonymous `POST /api/v1/accounts/register`, and safe methods `GET`, `HEAD`, `OPTIONS`.
  - Return the standard HiveSpace `ExceptionModel` error response before proxy forwarding when missing or invalid; add `CommonErrorCode.CsrfValidationFailed` (`APP0018`) if no existing common code expresses this gateway-level failure.
  - Acceptance: state-changing request without CSRF is rejected at gateway and downstream service receives no request.

### Update

- [ ] B017 [US3] Update ApiGateway startup pipeline
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/Program.cs`
  - Register Data Protection, `SessionForwardingMiddleware`, and `CsrfValidationMiddleware` before `app.MapReverseProxy()`.
  - Preserve `UseCors()`, `UseWebSockets()`, health checks, and existing YARP route loading.
  - Do not add business validation or account ownership logic to the gateway.
  - Acceptance: ApiGateway builds and middleware runs before YARP forwarding.

- [ ] B018 [US3] Update ApiGateway auth/session options
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.json`, `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings.Development.json`
  - Add shared Data Protection key-ring path/config section, cookie names, CSRF header name `X-HiveSpace-CSRF`, and frontend origins.
  - Ensure local HTTP origin `http://localhost:5000` and existing HTTPS gateway origin are documented/configurable.
  - Do not remove existing route prefixes or notification hub route.
  - Acceptance: local gateway can load config and existing routes remain unchanged.

### Verify

- [ ] B019 [US3] Verify gateway route and header behavior
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/**`
  - Confirm `/api/v1/accounts/**` still routes to `identity-cluster`.
  - Confirm middleware does not handle direct `/.well-known/**`, `/connect/**`, or `/Account/**` authority endpoints because they are not gateway routes.
  - Acceptance: route table remains compatible with `shared/api-catalog.md`.
