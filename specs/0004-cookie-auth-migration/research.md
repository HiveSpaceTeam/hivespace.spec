# Research: Cookie Auth Migration

## Decision: Use frontend-owned auth forms backed by IdentityService REST endpoints through ApiGateway

**Rationale:** The specification requires login, registration, refresh, and logout to use versioned `/api/v1/accounts/**` gateway routes while IdentityService remains authoritative for credentials, account state, roles, claims, lockout, email verification, and session issuance. Keeping browser auth requests behind the existing account route avoids direct frontend calls to service implementation URLs and preserves the existing public route ownership model.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Keep OIDC redirect login and only restyle IdentityService pages | Violates the requirement that frontend apps own the user-facing login/register UI and normal sign-in does not render IdentityService UI. |
| Frontend calls IdentityService authority directly for login/register | Conflicts with the clarified requirement that browser auth actions use `/api/v1/accounts/**` through ApiGateway. |
| Move credential ownership to frontend or UserService | Violates service boundaries. IdentityService owns credentials, roles, claims, account status, and token/session issuance. |

## Decision: Store token material in HttpOnly cookies without application-level encryption, not web storage

**Rationale:** The current shared frontend auth code uses `oidc-client-ts` and stores access/refresh token material in browser storage. The migration requires no access token or refresh token to be available to scripts, and the planning update explicitly requires the token to be placed in cookies without application-level encryption. IdentityService will therefore set raw token cookies protected by browser cookie controls (`HttpOnly`, `Secure`, `SameSite=None`) and by the tokens' own signing, expiry, and refresh rotation. This keeps tokens unavailable to JavaScript while allowing ApiGateway to read the access-token cookie directly and forward the bearer context expected by downstream services.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Continue SPA-held access and refresh tokens with stricter storage cleanup | Fails the no script-readable token requirement. |
| Store opaque session IDs in Redis only | Strong revocation properties, but it does not satisfy the updated direction to place token material in cookies and introduces a mandatory distributed session store. |
| Use only ASP.NET Identity application cookies downstream | Downstream services already expect bearer authorization context and should not read browser cookies directly. |
| Encrypt token material in a Data Protection cookie envelope | Stronger cookie-at-rest confidentiality, but conflicts with the explicit update that token cookies should not use application-level encryption. |

## Decision: ApiGateway reads and validates the access-token cookie directly

**Rationale:** IdentityService issues the access-token cookie and refresh-token cookie, while ApiGateway only needs the access token to attach `Authorization: Bearer <token>` to forwarded downstream requests. Because the cookie value is not encrypted by the application, the gateway can validate the JWT using the existing authority/signing-key configuration and expiry checks instead of sharing ASP.NET Core Data Protection keys.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Gateway calls IdentityService to introspect every request | Adds synchronous service dependency and latency to every protected API call. |
| Gateway stores sessions in its own database | Gives gateway account/session data ownership, which conflicts with the boundary that gateway should not own business or identity state. |
| Frontend exchanges cookie for bearer token before each call | Re-exposes bearer tokens to browser scripts. |
| Share Data Protection keys to decrypt an auth envelope | Unnecessary for the updated no-encryption cookie design and adds config coupling between IdentityService and ApiGateway. |

## Decision: Gateway validates CSRF and performs cookie-to-bearer forwarding

**Rationale:** Downstream services should keep existing JWT/bearer policy behavior. ApiGateway is the one browser-facing entry point that sees the cookie, so it is the correct place to reject missing/invalid CSRF on cookie-authenticated state-changing requests and to translate the session into the authorization header expected by existing services.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Validate CSRF independently in every downstream service | Duplicates browser-session security logic and requires every service to understand gateway cookies. |
| Let IdentityService proxy all authenticated API calls | Centralizes unrelated business traffic in IdentityService and violates service boundaries. |
| Rely only on SameSite cookies without a token header | The specification explicitly requires a server-issued CSRF token in a custom header. |

## Decision: Use a readable signed CSRF token separate from the HttpOnly auth cookie

**Rationale:** Frontend scripts need a value they can place in `X-HiveSpace-CSRF`, while access and refresh tokens must remain unreadable. A signed CSRF token returned by auth/session endpoints or stored in a non-HttpOnly secure cookie supports reloads, tabs, and shared app behavior without exposing bearer credentials.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Put the CSRF token only in memory | Breaks reload and second-tab behavior unless every first state-changing request performs a refresh bootstrap. |
| Put the access token in a readable cookie and reuse it as CSRF proof | Re-exposes credential material and mixes authentication with CSRF protection. |
| Require CSRF on anonymous login/register before any token exists | Adds bootstrap complexity not required by the spec, which scopes CSRF to cookie-authenticated state-changing browser requests. |

## Decision: Create a shared frontend `AuthLayout` for all auth pages

**Rationale:** The demo `Signin.vue` and `Signup.vue` already define the intended two-column auth surface with `FullScreenLayout` and `CommonGridShape`. Recreating that structure in each app would duplicate layout, spacing, dark-mode behavior, and right-panel composition. A shared `AuthLayout` in `@hivespace/shared` lets each page provide only the left-side form while the right panel consistently uses the shared `CommonGridShape` background and accepts page/app-specific center image and translated text.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Copy the demo page layout into every admin/seller/buyer auth page | Creates duplicated production UI that will drift across apps. |
| Import demo `Signin.vue`/`Signup.vue` directly into production apps | Violates the app dependency boundary and keeps demo-only social buttons/routes/logging coupled to production. |
| Keep right-side images/backgrounds app-local without a shared shell | Fails the requirement that every page share the same auth layout and `CommonGridShape` background. |

## Decision: Reuse existing integration events unchanged

**Rationale:** Registration still creates an IdentityService account and publishes `IdentityUserCreatedIntegrationEvent`; UserService still creates the matching profile from that event. Email verification and seller role propagation continue through existing identity/user/store events. The feature changes browser auth transport, not cross-service event semantics.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Add a new `BrowserSessionCreatedIntegrationEvent` | No other service needs a durable cross-service fact for session creation. |
| Add a new registration event separate from `IdentityUserCreatedIntegrationEvent` | Duplicates the existing profile-creation contract. |
| Synchronously call UserService during registration | Violates the existing event-driven boundary and creates unnecessary registration coupling. |

## Decision: Do not introduce a MassTransit saga

**Rationale:** The feature is a request/response browser authentication migration with local IdentityService account/session changes and gateway request mediation. It does not require multi-service compensation, long-running async waits, or saga state transitions.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Add a registration saga across IdentityService and UserService | Existing eventual profile creation already uses `IdentityUserCreatedIntegrationEvent`; the browser can handle brief profile projection delay. |
| Add a session renewal saga | Session renewal is a synchronous IdentityService-owned account/session operation. |

## Decision: Delete old IdentityService Razor auth UI and keep only minimal compatibility redirects

**Rationale:** Existing bookmarks, stale redirects, and IdentityServer return URLs may still hit `/Account/Login`, `/Account/Register`, or `/Account/Logout`, but IdentityService should no longer keep user-facing Razor UI for these flows. Deleting the Razor Page UI removes the obsolete surface area, while minimal redirect routes or middleware preserve compatibility with old URLs and satisfy the requirement that those URLs redirect to frontend routes.

**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| Keep Razor Pages and make their `OnGet` handlers redirect-only | Leaves obsolete UI/page infrastructure in the service when the feature goal is to move all HiveSpace auth UI to frontend apps. |
| Delete the Razor Pages without compatibility redirects | Risks broken old links and violates the clarified old-URL redirect requirement. |
| Continue rendering the pages with a deprecation banner | Still violates the no IdentityService-hosted UI requirement. |
| Redirect every old URL to one fixed frontend app | Fails the requirement to use return URL or app/client hints when available. |
