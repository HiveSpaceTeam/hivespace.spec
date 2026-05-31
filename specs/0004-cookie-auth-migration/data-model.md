# Data Model: Cookie Auth Migration

This feature adds session and API contract models, not new business aggregates. Identity accounts remain in IdentityService, and user profiles remain in UserService.

## BrowserSession

Represents the shared platform browser session accepted by ApiGateway for admin, seller, and buyer app requests.

| Field | Type | Owner | Notes |
|---|---|---|---|
| `sessionId` | string | IdentityService | Opaque session identifier returned in derived session state if needed for CSRF binding/log correlation. |
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

- A session is valid only when the access-token cookie is present, the token validates against IdentityService signing/issuer/audience rules, it is unexpired, and the IdentityService account remains eligible at refresh time.
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

## TokenCookies

HttpOnly token cookies issued by IdentityService. The cookies do not use application-level encryption or a Data Protection envelope; confidentiality from browser scripts comes from `HttpOnly`, and token integrity comes from the token signature/refresh-token validation model.

| Cookie / field | Type | Notes |
|---|---|---|
| `__Host-HiveSpace.AccessToken` | string | Raw signed JWT access token in an HttpOnly cookie. ApiGateway validates and forwards it as bearer auth. |
| `__Host-HiveSpace.RefreshToken` | string | Raw refresh token or opaque refresh handle in an HttpOnly cookie. IdentityService uses it for refresh/logout only. |
| `sessionId` claim/handle | string | Used for correlation and CSRF binding where available. |
| `sub` / `userId` claim | GUID string | Subject identifier. |
| `exp` / access expiry | timestamp | Gateway rejects expired access-token cookies. |
| `refreshExpiresAt` | timestamp | IdentityService rejects refresh after this time. May be stored server-side or encoded in refresh-token metadata depending on existing IdentityService pattern. |
| `securityStamp` | string or null | Used at refresh to detect credential/security changes if available. |
| `issuedAt` | timestamp | UTC issue time. |

### Validation rules

- Cookie names: `__Host-HiveSpace.AccessToken` and `__Host-HiveSpace.RefreshToken`.
- Attributes: `HttpOnly`, `Secure`, `SameSite=None`, `Path=/`, no `Domain`.
- Token cookies must not be exposed in JSON responses or script-readable storage.
- Gateway must validate the access-token cookie before forwarding and strip HiveSpace auth/CSRF cookies before downstream requests.
- Refresh token/handle is not used by ApiGateway.

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

The downstream authorization proof produced by ApiGateway from the access-token cookie.

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
