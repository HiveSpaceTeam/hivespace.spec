# API Contracts: Email OTP Sign-In (0009)

All endpoints are served through ApiGateway under the existing `/api/v1/accounts/**` route prefix, which forwards to IdentityService.

These public contracts remain sign-in-specific for v1 even though the internal IdentityService model is a reusable auth-scoped `OtpChallenge`.

## POST `/api/v1/accounts/otp/request`

**Auth**: Anonymous  
**Purpose**: Request a one-time sign-in code to be sent to an email address. Always returns a generic confirmation-style success response regardless of whether the address belongs to an active account, a locked account, an unknown address, or is in cooldown.

### Request

```json
{
  "email": "user@example.com"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `email` | `string` | Yes | Validated as a non-empty email format; normalized before lookup |

### Response - 200 OK

```json
{
  "challengeToken": "a3f8b1c2d4e5f6070809aabbccddeeff",
  "expiresAt": "2026-06-19T10:25:00Z",
  "canResendAt": "2026-06-19T10:17:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `challengeToken` | `string` | Opaque token to submit to `/otp/verify` |
| `expiresAt` | `ISO 8601 datetime` | Frontend countdown target |
| `canResendAt` | `ISO 8601 datetime` | Frontend resend cooldown target |

### Error Responses

| Status | Condition |
|---|---|
| `400 Bad Request` | Missing or invalid request body |
| `429 Too Many Requests` | Global rate limit exceeded |

## POST `/api/v1/accounts/otp/verify`

**Auth**: Anonymous  
**Purpose**: Verify a one-time sign-in code using the opaque challenge token. On success, issue a browser session identical to password sign-in.

### Request

```json
{
  "challengeToken": "a3f8b1c2d4e5f6070809aabbccddeeff",
  "code": "483921"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `challengeToken` | `string` | Yes | Token returned by `/otp/request` |
| `code` | `string` | Yes | 6-digit numeric code |

### Response - 200 OK

Browser session cookies are set on the response using the same behavior as password sign-in.

```json
{
  "redirectUrl": "/seller/dashboard"
}
```

| Field | Type | Notes |
|---|---|---|
| `redirectUrl` | `string` or `null` | Same-origin relative path or null fallback |

### Error Responses

| Status | Body | Condition |
|---|---|---|
| `400 Bad Request` | Standard validation error | Missing fields or invalid format |
| `401 Unauthorized` | `{ "error": "invalid_or_expired_code" }` | Missing challenge, expired challenge, used challenge, invalidated challenge, or incorrect code |
| `401 Unauthorized` | `{ "error": "max_attempts_exceeded" }` | Maximum incorrect attempts reached; challenge invalidated |

## Session Cookie Behavior

OTP verify and password login must produce identical cookie semantics. This remains true even though OTP persistence is modeled generically underneath.

Cookie behavior follows [ADR-0003](../../architecture/decisions/ADR-0003-gateway-mediated-cookie-browser-sessions.md).

## Open-Redirect Protection

The `redirectUrl` field is validated server-side. Absolute, protocol-relative, malformed, or cross-origin values are rejected and converted to `null`.
