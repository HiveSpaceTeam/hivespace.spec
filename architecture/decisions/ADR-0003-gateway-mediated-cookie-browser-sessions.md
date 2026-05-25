# ADR-0003: Gateway-Mediated Cookie Browser Sessions

- **Status**: Draft
- **Date**: 2026-05-25
- **Feature**: `0004-cookie-auth-migration`
- **Deciders**: Project maintainers

## Context

HiveSpace frontend apps currently use OIDC redirect flows and shared frontend auth code that can hold access and refresh token material in browser-readable storage. The cookie auth migration requires admin, seller, and buyer apps to own login/register UI while IdentityService remains the authority for credentials, account state, roles, claims, lockout, email verification, token/session issuance, refresh, and logout.

Downstream services already expect bearer JWT authorization context through ApiGateway. They should not parse browser cookies or duplicate browser CSRF protections. The migration also requires a shared platform browser session accepted across all three frontend apps, with app-specific role and account-state checks still enforced.

## Decision

IdentityService will expose browser auth REST endpoints under the existing `/api/v1/accounts/**` gateway route. Successful login or registration will issue a secure encrypted HttpOnly browser session cookie and a server-issued CSRF token. ApiGateway will decrypt and validate the browser session cookie, attach the contained access token as `Authorization: Bearer <token>` for downstream services, strip browser session cookies before forwarding, and reject cookie-authenticated state-changing browser requests that do not include a valid `X-HiveSpace-CSRF` header.

Frontend apps will stop storing access and refresh tokens in localStorage or sessionStorage. Shared frontend auth code will call gateway account endpoints with credentials enabled, keep only derived session state and CSRF token handling in script-readable state, and continue relying on downstream service policies for app/domain authorization.

## Consequences

### Positive

- Login and registration UI moves to the frontend apps without moving credential ownership out of IdentityService.
- Access and refresh token material is no longer readable by browser scripts.
- Existing downstream service JWT policies continue to work with forwarded bearer context.
- CSRF enforcement is centralized at the browser-facing gateway before downstream state changes.
- The same gateway session can support admin, seller, and buyer apps while each app still applies role rules.

### Negative / Trade-offs

- ApiGateway becomes security mediation infrastructure, not only a route table.
- IdentityService and ApiGateway must share Data Protection key material and cookie/session configuration.
- Immediate account-status revocation between refreshes depends on short access-token lifetime or additional validation because the gateway remains stateless.
- SignalR and any browser API client must be adjusted to use cookie credentials rather than script-held bearer tokens.

### Risks

- Misconfigured cookie `SameSite`, `Secure`, CORS, or credentials settings can break local development or cross-app sessions. Mitigation: verify all three app origins and document cookie attributes.
- Data Protection key mismatch between IdentityService and ApiGateway will make valid sessions unreadable. Mitigation: add startup/config verification and health/log evidence.
- CSRF validation gaps could leave downstream state-changing routes exposed to browser credential replay. Mitigation: enforce method-based gateway middleware tests and negative browser tests.
- Old OIDC storage keys may leave stale tokens in browser storage after migration. Mitigation: shared auth bootstrap cleans known OIDC keys and verification checks local/session storage.

## Alternatives Considered

| Option | Why rejected |
|---|---|
| Keep SPA-held OIDC access and refresh tokens | Fails the requirement that no access or refresh token is available to browser scripts. |
| Let every downstream service read and validate the auth cookie | Duplicates browser-session logic across services and violates the current bearer authorization contract. |
| Use Redis-only opaque sessions from the start | Strong revocation model, but adds mandatory distributed session state and operational dependency for the first migration. It can be revisited if immediate revocation becomes required. |
| Gateway introspects every request with IdentityService | Adds a synchronous IdentityService dependency and latency to every protected API call. |
| Keep IdentityService Razor Pages and style them per app | Does not satisfy frontend-owned login/register UI or the no IdentityService-hosted UI outcome. |
| Keep IdentityService Razor Pages only to redirect old URLs | Preserves obsolete UI infrastructure. Minimal compatibility routes can handle old URL redirects without retaining Razor auth UI. |

## Follow-Up

- Update `shared/api-catalog.md` and IdentityService/ApiGateway service docs for new account endpoints and gateway auth mediation.
- Generate implementation tasks for IdentityService browser auth endpoints, ApiGateway cookie/CSRF middleware, shared frontend auth replacement, and app-owned auth pages.
- Verify no access or refresh token remains in browser-readable storage after login, refresh, registration, or logout.
