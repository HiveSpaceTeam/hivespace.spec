# IdentityService API

## Direct IdentityServer Authority Endpoints

These endpoints are served directly by IdentityService at `http://localhost:5001`. They are not forwarded by ApiGateway and are not versioned under `/api/v1`.

| Entry point | Auth | Purpose |
|---|---|---|
| `/.well-known/**` | Anonymous | OIDC discovery and metadata |
| `/connect/**` | Anonymous or authenticated per OIDC flow | Authorize, token, refresh, userinfo, revocation, introspection, device, callback, and sign-out protocol endpoints |
| `/Account/**` | Anonymous or authenticated per page flow | Interactive login, registration, consent, external login, device flow, diagnostics, and access denied pages |

`/identity/**` and `/api/v1/identity/**` are intentionally not part of the target contract.

## Account and Email Verification

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/accounts/email-verification` | `RequireAdminOrUser` | Send verification email from the identity-owned verification workflow |
| POST | `/api/v1/accounts/email-verification/verify` | Anonymous | Verify email token and update identity-owned verification state |

Account registration and credential flows may be served through IdentityServer account pages or identity-owned account REST endpoints. After successful account creation, IdentityService publishes `IdentityUserCreatedIntegrationEvent`.

## Admin Identity Management

| Method | Path | Auth | Purpose |
|---|---|---|---|
| POST | `/api/v1/admins` | `RequireAdmin` | Create an identity-owned admin account |
| GET | `/api/v1/admins` | `RequireAdmin` | List identity-owned admin accounts |
| PUT | `/api/v1/admins/users/status` | `RequireAdmin` | Suspend, reactivate, or otherwise change identity-owned account status |
| DELETE | `/api/v1/admins/users/{userId}` | `RequireAdmin` | Delete or deactivate identity-owned account access |

Admin profile/store review actions that do not change identity-owned account state remain UserService-owned.
