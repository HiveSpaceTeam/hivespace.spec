# IdentityService API

## Direct IdentityServer Authority Endpoints

These endpoints are served directly by IdentityService at `http://localhost:5001`. They are not forwarded by ApiGateway and are not versioned under `/api/v1`.

| Entry point | Auth | Purpose |
|---|---|---|
| `/.well-known/**` | Anonymous | OIDC discovery and metadata |
| `/connect/**` | Anonymous or authenticated per OIDC flow | Authorize, token, refresh, userinfo, revocation, introspection, device, callback, and sign-out protocol endpoints |
| `/Account/**` | Anonymous or authenticated per compatibility flow | Legacy account URL compatibility redirects and any non-HiveSpace IdentityServer protocol pages that remain required; HiveSpace login/register/logout UI is frontend-owned |

`/identity/**` and `/api/v1/identity/**` are intentionally not part of the target contract.

## Account and Email Verification

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/accounts/login` | Anonymous | Password login from frontend-owned UI; sets secure HttpOnly access/refresh token cookies and CSRF token |
| POST | `/api/v1/accounts/register` | Anonymous | Frontend-owned public account registration where allowed; creates identity account, triggers existing profile creation flow, and sets token cookies/browser session |
| POST | `/api/v1/accounts/session/refresh` | Session cookie + CSRF | Bootstrap after reload or refresh/rotate the browser session through the gateway |
| POST | `/api/v1/accounts/logout` | Session cookie + CSRF | Clear token cookies and CSRF cookie |
| POST | `/api/v1/accounts/email-verification` | `RequireAdminOrUser` | Send verification email from the identity-owned verification workflow |
| POST | `/api/v1/accounts/email-verification/verify` | Anonymous | Verify email token and update identity-owned verification state |

Account registration and credential flows for HiveSpace browser apps are served through versioned account REST endpoints via ApiGateway, not IdentityService-rendered UI. After successful account creation, IdentityService publishes `IdentityUserCreatedIntegrationEvent`. Successful login, registration, and refresh responses must not expose access or refresh tokens in JSON; token material is stored only in secure HttpOnly cookies.

## Admin Identity Management

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Create an identity-owned admin account |
| GET | `/api/v1/admins` | `RequireAdmin` | List identity-owned admin accounts |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | Suspend, reactivate, or otherwise change identity-owned account status |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | Delete or deactivate identity-owned account access |

Admin profile/store review actions that do not change identity-owned account state remain UserService-owned.
