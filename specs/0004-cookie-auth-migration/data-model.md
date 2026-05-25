# Data Model: Cookie Auth Migration

This feature adds session and API contract models, not new business aggregates. Identity accounts remain in IdentityService, and user profiles remain in UserService.

## BrowserSession

Represents the shared platform browser session accepted by ApiGateway for admin, seller, and buyer app requests.

| Field | Type | Owner | Notes |
|---|---|---|---|
| `sessionId` | string | IdentityService | Opaque session identifier inside the protected cookie envelope. |
| `userId` | GUID string | IdentityService | Public user ID shared with UserService profile creation. |
| `email` | string | IdentityService | Returned in session summary when available. |
| `displayName` | string or null | IdentityService/UserService projection in frontend | IdentityService can return token display claims; apps may enrich from `/api/v1/users/me`. |
| `roles` | string[] | IdentityService | Used by app route guards and downstream authorization policies. |
| `claims` | key-value collection | IdentityService | Minimal claims required by app and service policies. |
| `emailVerified` | boolean | IdentityService | Used by seller flow and account prompts. |
| `accountStatus` | string | IdentityService | Active, locked, inactive, deleted, or equivalent safe status. |
| `issuedAt` | timestamp | IdentityService | UTC issue time. |
| `expiresAt` | timestamp | IdentityService | Browser session/access-token expiry visible to frontend. |
| `refreshExpiresAt` | timestamp | IdentityService | Latest allowed refresh time, not a browser-readable refresh token. |
| `csrfToken` | string | IdentityService | Server-issued token sent as `X-HiveSpace-CSRF`; may also be stored as a readable secure cookie. |

### Validation rules

- A session is valid only when the protected cookie decrypts successfully, is unexpired, and the IdentityService account remains eligible at refresh time.
- The frontend must not receive access token or refresh token fields.
- The same session cookie is accepted for all three frontend apps through the gateway; each app still enforces role and account-state rules.
- A failed refresh clears local frontend session state and routes to signed-out state.

### State transitions

| From | Trigger | To |
|---|---|---|
| None | Successful login or registration | Active |
| Active | Refresh before expiry | Active with rotated expiry/token material |
| Active | Logout | Cleared |
| Active | Expiry or failed refresh | Expired |
| Active | Account locked/inactive/deleted detected on refresh | Invalid |

## ProtectedSessionCookieEnvelope

Opaque cookie payload protected by ASP.NET Core Data Protection. It is not browser-readable.

| Field | Type | Notes |
|---|---|---|
| `sessionId` | string | Used for correlation and CSRF binding. |
| `userId` | GUID string | Subject identifier. |
| `accessToken` | string | Bearer token used only by ApiGateway for downstream forwarding. |
| `refreshToken` or `refreshHandle` | string | Used only by IdentityService refresh/logout paths, never exposed to scripts. |
| `accessTokenExpiresAt` | timestamp | Gateway rejects or omits bearer forwarding when expired. |
| `refreshExpiresAt` | timestamp | IdentityService rejects refresh after this time. |
| `securityStamp` | string or null | Used at refresh to detect credential/security changes if available. |
| `issuedAt` | timestamp | UTC issue time. |

### Validation rules

- Cookie name: `__Host-HiveSpace.Auth`.
- Attributes: `HttpOnly`, `Secure`, `SameSite=None`, `Path=/`, no `Domain`.
- Payload must be integrity-protected and encrypted.
- Gateway must strip this cookie before forwarding to downstream services.

## CsrfToken

Server-issued anti-forgery proof tied to the browser session.

| Field | Type | Notes |
|---|---|---|
| `token` | string | Sent by frontend in `X-HiveSpace-CSRF`. |
| `sessionId` | string | Bound to the active session in signed token data. |
| `issuedAt` | timestamp | Used for rotation/expiry checks. |
| `expiresAt` | timestamp | Should not outlive the browser session. |

### Validation rules

- Required for cookie-authenticated `POST`, `PUT`, `PATCH`, and `DELETE` requests.
- Anonymous login and registration are exempt before a session exists.
- Refresh and logout require a valid CSRF token when a session cookie is present.
- Missing or invalid CSRF must be rejected at ApiGateway before downstream forwarding.

## FrontendAuthForm

App-owned login or registration form state.

| Field | Type | Applies to | Notes |
|---|---|---|---|
| `email` | string | Login, registration | Required, email format. |
| `password` | string | Login, registration | Required. Server applies Identity password policy. |
| `confirmPassword` | string | Registration | Must match password on client and server. |
| `fullName` | string or null | Registration | Passed to existing profile creation event when provided. |
| `app` | `admin`/`seller`/`buyer` | Login, registration | Used for app access checks and safe redirect defaults. |
| `returnUrl` | string or null | Login, registration | Must be validated as safe frontend route or origin. |
| `culture` | string or null | Login, registration | Used for localized safe error messages where supported. |

## AuthorizationContext

The downstream authorization proof produced by ApiGateway from the protected cookie.

| Field | Type | Notes |
|---|---|---|
| `Authorization` | HTTP header | `Bearer <access-token>` set by ApiGateway. |
| `X-Correlation-ID` | HTTP header | Preserved or generated by shared frontend/gateway flow. |
| User claims | JWT claims | Existing downstream services continue validating JWT claims and policies. |

### Validation rules

- Downstream services must not parse browser session cookies.
- Existing role and policy outcomes must remain consistent with current bearer-auth behavior.
- Gateway must not make business authorization decisions beyond session/CSRF validation and token forwarding.

## Existing Models Reused

| Model or contract | Owner | Reuse |
|---|---|---|
| Identity Account | IdentityService | Credentials, account status, roles, claims, lockout, email verification remain unchanged. |
| User Profile | UserService | Created from `IdentityUserCreatedIntegrationEvent`; no direct registration-time UserService write. |
| `IdentityUserCreatedIntegrationEvent` | IdentityService | Published after registration exactly as today. |
| `StoreCreatedIntegrationEvent` | UserService | Continues seller role propagation to IdentityService. |
