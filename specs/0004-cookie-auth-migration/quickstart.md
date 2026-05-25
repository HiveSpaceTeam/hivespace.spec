# Quickstart: Cookie Auth Migration

This quickstart is for implementation verification after tasks are generated and code changes are made.

## Prerequisites

- Read `../hivespace.microservice/AGENTS.md` and `../hivespace.web/AGENTS.md` before source changes.
- Start local infrastructure from `../hivespace.config/docker`.
- Run backend services through the gateway and frontend apps on their normal dev ports:
  - Admin: `http://localhost:5173`
  - Seller: `http://localhost:5174`
  - Buyer: `http://localhost:5175`
  - ApiGateway: `http://localhost:5000` or `https://localhost:7001`
  - IdentityService: `http://localhost:5001`
- For local HTTP gateway verification, set frontend `VITE_GATEWAY_BASE_URL=http://localhost:5000`.

## Backend Verification

1. Build IdentityService and ApiGateway.
2. Confirm IdentityService exposes:
   - `POST /api/v1/accounts/login`
   - `POST /api/v1/accounts/register`
   - `POST /api/v1/accounts/session/refresh`
   - `POST /api/v1/accounts/logout`
3. Confirm ApiGateway routes `/api/v1/accounts/**` to IdentityService.
4. Confirm ApiGateway decrypts the auth cookie, forwards `Authorization: Bearer <token>` to downstream services, and strips browser auth cookies before forwarding.
5. Confirm state-changing cookie-authenticated requests without `X-HiveSpace-CSRF` are rejected at ApiGateway.

## Frontend Verification

1. Open each app signed out.
2. Sign in through the app-owned sign-in page.
3. Verify the network request targets `/api/v1/accounts/login` through ApiGateway.
4. Verify no IdentityService-hosted login UI is rendered.
5. Verify browser localStorage and sessionStorage do not contain access tokens or refresh tokens.
6. Reload the app and confirm the shared auth bootstrap or router guard calls `POST /api/v1/accounts/session/refresh` through the configured gateway origin.
7. Call a protected workflow and confirm the downstream service accepts the request according to existing policies.
8. Sign out and confirm protected routes require sign-in after refresh.

## Cross-App Session Check

1. Sign in through buyer or seller.
2. Open another HiveSpace frontend app in the same browser.
3. Confirm the shared gateway session is recognized.
4. Confirm the target app still applies its own role rules. For example, a non-admin session must not enter admin protected routes.

## Registration Check

1. Register a new buyer or seller candidate through the frontend-owned registration form.
2. Confirm `POST /api/v1/accounts/register` returns success and sets the session cookie.
3. Confirm IdentityService created one account.
4. Confirm UserService creates the matching profile through the existing `IdentityUserCreatedIntegrationEvent` flow.
5. Confirm duplicate email registration returns a safe conflict error and creates no duplicate account.

## Old IdentityService URL Check

Open each old URL directly:

- `http://localhost:5001/Account/Login`
- `http://localhost:5001/Account/Register`
- `http://localhost:5001/Account/Logout`

Expected result: each redirects to a configured frontend route using safe return URL or app/client hints when present. No IdentityService Razor Page UI files or page handlers should remain for login, registration, or logout.

## CSRF Check

1. Sign in normally and capture a protected state-changing request.
2. Replay it without `X-HiveSpace-CSRF`.
3. Expected result: ApiGateway returns `400` or equivalent CSRF rejection before the downstream service changes state.
4. Replay it with the valid CSRF header.
5. Expected result: request reaches the downstream service and is authorized according to existing policies.
