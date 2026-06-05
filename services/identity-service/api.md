# IdentityService API

## Direct IdentityServer Authority Endpoints

These endpoints are served directly by IdentityService at `http://localhost:5001`. They are not forwarded by ApiGateway and are not versioned under `/api/v1`.

| Entry point | Auth | Purpose |
|---|---|---|
| `/.well-known/**` | Anonymous | OIDC discovery and metadata |
| `/connect/**` | Anonymous or authenticated per OIDC flow | Authorize, token, refresh, userinfo, revocation, introspection, device, callback, and sign-out protocol endpoints |
| `/Account/**` | Unsupported legacy URL space | Must not render old HiveSpace account pages or compatibility redirects after legacy page removal |

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
| GET | `/api/v1/accounts/external/google/challenge` | Anonymous | Start buyer/seller Google authentication from sign-in or sign-up and preserve app/return URL context |
| GET | `/api/v1/accounts/external/google/complete` | Anonymous | Complete Google sign-in, create/sign in a normal user only when no same-email password account exists, or start required frontend account linking |
| POST | `/api/v1/accounts/external/google/link` | Temporary Google link state + CSRF/link token | Link Google to an existing password account after consent and password confirmation, mark verified matching email, and issue browser session cookies |
| DELETE | `/api/v1/accounts/external/google/link` | Temporary Google link state + CSRF/link token | Cancel pending Google account linking without issuing a session or creating a duplicate same-email account |

Account registration and credential flows for HiveSpace browser apps are served through versioned account REST endpoints via ApiGateway, not IdentityService-rendered UI. After successful account creation, IdentityService publishes `IdentityUserCreatedIntegrationEvent`. Successful login, registration, and refresh responses must not expose access or refresh tokens in JSON; token material is stored only in secure HttpOnly cookies.

Google sign-in is limited to buyer and seller app contexts. New Google-authenticated users are normal user accounts only when no same-email local password account exists. If a verified Google email matches an existing unlinked password account, Google sign-in must stop at required linking or safe exits to password sign-in, password reset, or another Google account. Admin accounts cannot be created or signed in through Google.

## Admin Identity Management

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Create an identity-owned admin account |
| GET | `/api/v1/admins` | `RequireAdmin` | List identity-owned admin accounts |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | Suspend, reactivate, or otherwise change identity-owned account status |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | Delete or deactivate identity-owned account access |

Admin profile/store review actions that do not change identity-owned account state remain UserService-owned.
