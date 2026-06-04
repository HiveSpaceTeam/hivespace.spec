# Browser Auth API Contract: Google Login and Account Linking

All browser-facing account endpoints are routed through ApiGateway under:

```text
{VITE_GATEWAY_BASE_URL}/api/v1/accounts
```

Successful session responses must never include access or refresh token values. Token material is stored only in secure HttpOnly cookies.

## Start Google Authentication

```http
GET /api/v1/accounts/external/google/challenge?app=buyer&returnUrl=%2Fprofile&culture=en
```

Auth: Anonymous

Query:

| Name | Required | Notes |
| --- | --- | --- |
| `app` | Yes | `buyer` or `seller`; `admin` is rejected |
| `returnUrl` | No | Safe frontend path or allowed loopback development URL |
| `culture` | No | User-facing culture hint |

Response:

- `302` to Google provider authorization URL.
- `400` for invalid app or unsafe return URL.

Buyer and seller sign-in and sign-up pages both use this same endpoint. The callback result decides whether to sign in an already linked account, create a new normal user account, or start the account-link confirmation flow for a matching unlinked local password account. A matching local password account blocks separate same-email Google-only account creation.

## Complete Google Callback

```http
GET /api/v1/accounts/external/google/complete
```

Auth: Anonymous

Response outcomes:

| Outcome | Response |
| --- | --- |
| Google identity is already linked | Issue session cookies and redirect to sanitized return URL |
| Google verified email has no matching local password account | Follow normal Google path: sign in if the Google identity is linked, otherwise create a normal user account, publish existing account-created event, issue session cookies, redirect to sanitized return URL |
| Google verified email matches unlinked local password account | Set temporary pending-link state and redirect to frontend account-link page; do not create or sign in a separate Google-only account for the same email |
| Google email is missing or unverified | Redirect to frontend sign-in page with safe error code |
| Account is inactive, locked, disabled, deleted, or not allowed for app | Redirect to frontend sign-in page with safe error code |

No response may expose provider access tokens, provider refresh tokens, or provider user IDs to browser scripts.

## Confirm Google Link

```http
POST /api/v1/accounts/external/google/link
Content-Type: application/json
X-HiveSpace-CSRF: <csrf-or-link-csrf-token>
```

Auth: Temporary pending Google link state

Request:

```json
{
  "consentAccepted": true,
  "password": "existing-account-password",
  "app": "buyer",
  "returnUrl": "/profile",
  "culture": "en"
}
```

Success response:

```json
{
  "user": {
    "userId": "guid",
    "email": "user@example.com",
    "displayName": "User Name",
    "roles": ["Buyer"],
    "emailVerified": true,
    "accountStatus": "Active"
  },
  "expiresAt": "2026-05-31T12:30:00Z",
  "refreshExpiresAt": "2026-06-07T12:00:00Z",
  "csrfToken": "new-csrf-token",
  "redirectTo": "/profile"
}
```

Errors:

| Status | Condition |
| --- | --- |
| `400` | Missing/expired pending link state, missing or false `consentAccepted`, invalid app, unsafe return URL |
| `401` | Wrong password |
| `403` | Account inactive, locked, admin-only, or otherwise not allowed |
| `409` | Google identity already linked to another account |

On success, the endpoint must:

- require `consentAccepted = true` before password validation or durable linking;
- add the Google external login to the existing account;
- mark the matching email verified when Google reported it verified;
- clear temporary pending-link state;
- issue normal browser session cookies and a fresh CSRF token.

## Cancel Google Link

```http
DELETE /api/v1/accounts/external/google/link
X-HiveSpace-CSRF: <csrf-or-link-csrf-token>
```

Auth: Temporary pending Google link state

Response:

- `204 No Content`

Behavior:

- Clear temporary pending-link state.
- Do not create a durable Google link.
- Do not issue a signed-in browser session.
- Do not create a duplicate Google-only HiveSpace account for the same verified email.

## Reused Session Endpoints

These existing endpoints keep their current contracts:

- `POST /api/v1/accounts/login`
- `POST /api/v1/accounts/register`
- `POST /api/v1/accounts/session/refresh`
- `POST /api/v1/accounts/logout`
