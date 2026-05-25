# Contract: Browser Auth API

All endpoints are browser-facing through ApiGateway:

```text
{VITE_GATEWAY_BASE_URL}/api/v1/accounts/**
```

Local gateway origins may be `http://localhost:5000` or `https://localhost:7001`; frontend apps select the active one through `VITE_GATEWAY_BASE_URL`.

The frontend must call these endpoints with credentials enabled. Responses must not include access tokens or refresh tokens.

Error responses use the existing HiveSpace exception response shape produced by `UseHiveSpaceExceptionHandler` / `ExceptionResponseFactory`; do not introduce a separate browser-auth error body.

```json
{
  "errors": [
    {
      "code": "IDN6006",
      "messageCode": "InvalidCredentials",
      "source": "email"
    }
  ],
  "status": "401",
  "timestamp": "2026-05-25T15:00:00+07:00",
  "traceId": "8f8d6d89-7f6c-4777-a7aa-2ac65009db0a",
  "version": "1.0"
}
```

Frontend clients should map `errors[].messageCode` and `errors[].code` to localized copy.

## Common Headers

### Request

| Header | Required | Notes |
|---|---:|---|
| `Content-Type: application/json` | For JSON body requests | Login/register/refresh/logout use JSON or empty JSON body. |
| `X-Correlation-ID` | Recommended | Existing frontend `ApiService` generates this. |
| `X-HiveSpace-CSRF` | Required for cookie-authenticated state-changing requests | Required for refresh, logout, and downstream `POST`/`PUT`/`PATCH`/`DELETE` calls. |

### Response Cookies

| Cookie | Attributes | Notes |
|---|---|---|
| `__Host-HiveSpace.Auth` | `HttpOnly; Secure; SameSite=None; Path=/` | Protected browser session cookie. |
| `HiveSpace.Csrf` | `Secure; SameSite=None; Path=/` | Readable signed CSRF token or token mirror used for `X-HiveSpace-CSRF`. |

## POST `/api/v1/accounts/login`

Validates a password login and establishes a browser session.

### Auth

Anonymous. If a stale session cookie is present, the endpoint may replace it on successful login.

### Request

```json
{
  "email": "buyer@example.com",
  "password": "P@ssw0rd!",
  "app": "buyer",
  "returnUrl": "/checkout",
  "culture": "en"
}
```

| Field | Required | Validation |
|---|---:|---|
| `email` | Yes | Valid email format. |
| `password` | Yes | Non-empty. |
| `app` | Yes | `admin`, `seller`, or `buyer`. |
| `returnUrl` | No | Must be a safe frontend route or configured frontend origin. |
| `culture` | No | Supported culture code. |

### Success Response: `200 OK`

Sets `__Host-HiveSpace.Auth` and `HiveSpace.Csrf`.

```json
{
  "user": {
    "userId": "7f3989b3-2bf3-4a1b-8e85-255a79ecce77",
    "email": "buyer@example.com",
    "displayName": "Buyer Example",
    "roles": ["User"],
    "emailVerified": true,
    "accountStatus": "Active"
  },
  "expiresAt": "2026-05-25T15:30:00Z",
  "refreshExpiresAt": "2026-06-24T15:00:00Z",
  "csrfToken": "signed-csrf-token",
  "redirectTo": "/checkout"
}
```

### Error Responses

| Status | Error | Notes |
|---:|---|---|
| `400` | Validation errors or `IDN6013` / `InvalidReturnUrl` | Invalid request shape, missing fields, or unsafe return URL. |
| `401` | `IDN6006` / `InvalidCredentials` | Invalid email/password, safe generic message. |
| `403` | `IDN6008` / `AccountLocked` | Account is locked. |
| `403` | `IDN6007` / `AccountInactive` | Account cannot sign in due to inactive/deleted status. |
| `403` | `IDN6009` / `AccountNotAllowed` | Valid account but not allowed for requested app. |

## POST `/api/v1/accounts/register`

Creates an identity-owned account, publishes the existing profile-creation event, and establishes a browser session.

### Auth

Anonymous.

### Request

```json
{
  "email": "newbuyer@example.com",
  "password": "P@ssw0rd!",
  "confirmPassword": "P@ssw0rd!",
  "fullName": "New Buyer",
  "app": "buyer",
  "returnUrl": "/",
  "culture": "en"
}
```

| Field | Required | Validation |
|---|---:|---|
| `email` | Yes | Unique valid email. |
| `password` | Yes | IdentityService password policy. |
| `confirmPassword` | Yes | Must match `password`. |
| `fullName` | No | Trimmed, safe length. |
| `app` | Yes | `seller` or `buyer` for public self-registration. Admin public self-registration is not allowed. |
| `returnUrl` | No | Must be safe. |
| `culture` | No | Supported culture code. |

### Success Response: `201 Created`

Sets `__Host-HiveSpace.Auth` and `HiveSpace.Csrf`.

```json
{
  "user": {
    "userId": "018fb1cf-5f9a-4bf5-9f76-177cdf9c3481",
    "email": "newbuyer@example.com",
    "displayName": "New Buyer",
    "roles": ["User"],
    "emailVerified": false,
    "accountStatus": "Active"
  },
  "expiresAt": "2026-05-25T15:30:00Z",
  "refreshExpiresAt": "2026-06-24T15:00:00Z",
  "csrfToken": "signed-csrf-token",
  "redirectTo": "/"
}
```

### Side Effects

- IdentityService creates the identity account.
- IdentityService publishes existing `IdentityUserCreatedIntegrationEvent`.
- UserService profile creation remains eventual and event-driven.

### Error Responses

| Status | Error | Notes |
|---:|---|---|
| `400` | Validation errors or `IDN6013` / `InvalidReturnUrl` | Password/confirm/email/full name validation failed, or return URL is unsafe. |
| `403` | `IDN6009` / `AccountNotAllowed` | Public registration is not allowed for the requested app. |
| `409` | `IDN6010` / `DuplicateEmail` | Email is already registered. |

## POST `/api/v1/accounts/session/refresh`

Bootstraps session state after reload or refreshes the browser session before expiry. It rotates token material without exposing tokens to scripts.

Frontend trigger points:

- Shared auth bootstrap after app load when no in-memory session exists.
- Existing app route guards that ask shared auth for the current user.
- Near-expiry renewal before protected API calls or protected navigation.
- Explicit refresh after email verification or seller-role propagation.
- One retry path after a protected call returns unauthenticated.

### Auth

Requires valid session cookie and `X-HiveSpace-CSRF`.

### Request

```json
{
  "app": "seller"
}
```

### Success Response: `200 OK`

Sets rotated `__Host-HiveSpace.Auth` and `HiveSpace.Csrf`.

```json
{
  "user": {
    "userId": "7f3989b3-2bf3-4a1b-8e85-255a79ecce77",
    "email": "seller@example.com",
    "displayName": "Seller Example",
    "roles": ["User", "StoreOwner"],
    "emailVerified": true,
    "accountStatus": "Active"
  },
  "expiresAt": "2026-05-25T16:00:00Z",
  "refreshExpiresAt": "2026-06-24T15:00:00Z",
  "csrfToken": "rotated-signed-csrf-token"
}
```

### Error Responses

| Status | Error | Notes |
|---:|---|---|
| `400` | `APP0018` / `CsrfValidationFailed` | Missing or invalid CSRF header rejected by ApiGateway before forwarding. |
| `401` | `IDN6011` / `InvalidSession` | Session cookie cannot be read or validated. |
| `401` | `IDN6012` / `SessionExpired` | Session cannot be refreshed. |
| `403` | `IDN6007` / `AccountInactive` | Account status changed after session creation. |
| `403` | `IDN6008` / `AccountLocked` | Account became locked after session creation. |
| `403` | `IDN6009` / `AccountNotAllowed` | Account no longer satisfies requested app role rules. |

## POST `/api/v1/accounts/logout`

Clears the browser session.

### Auth

Requires valid session cookie and `X-HiveSpace-CSRF`. The endpoint should be idempotent and clear cookies even when the session is already invalid.

### Request

```json
{
  "redirectTo": "/"
}
```

### Success Response: `204 No Content`

Clears `__Host-HiveSpace.Auth` and `HiveSpace.Csrf`.

### Error Responses

| Status | Error | Notes |
|---:|---|---|
| `400` | `APP0018` / `CsrfValidationFailed` | Missing or invalid CSRF header for an active cookie-authenticated request. |

## Gateway CSRF Contract For Downstream APIs

For browser requests that include `__Host-HiveSpace.Auth`, ApiGateway must require `X-HiveSpace-CSRF` for:

- `POST`
- `PUT`
- `PATCH`
- `DELETE`

Exemptions:

- Anonymous `POST /api/v1/accounts/login`
- Anonymous `POST /api/v1/accounts/register`
- Safe methods: `GET`, `HEAD`, `OPTIONS`

Failure response before downstream forwarding uses the same HiveSpace exception response shape:

```json
{
  "errors": [
    {
      "code": "APP0018",
      "messageCode": "CsrfValidationFailed",
      "source": "xHiveSpaceCsrf"
    }
  ],
  "status": "400",
  "timestamp": "2026-05-25T15:00:00+07:00",
  "traceId": "8f8d6d89-7f6c-4777-a7aa-2ac65009db0a",
  "version": "1.0"
}
```
