# Feature Specification: Cookie Auth Migration

- **Feature Branch**: `0004-cookie-auth-migration`
- **Created**: 2026-05-25
- **Status**: Implemented
- **Implemented**: 2026-05-31
- **Input**: User description: "Update the current Identity service login, register, and token handling flow so IdentityService no longer owns UI pages. Move all login/register UI pages to the frontend apps. Frontend forms submit authentication requests to IdentityService. Successful password login returns 200 and establishes an HttpOnly cookie. After login, the app calls the gateway with that cookie; the gateway extracts the token, attaches it to forwarded requests, and downstream services continue receiving authenticated requests."

## Clarifications

### Session 2026-05-25

- Q: Should signing in through one HiveSpace frontend app create a browser session accepted by the gateway for all three apps, or should sessions be scoped by app? → A: Shared platform session across admin, seller, and buyer apps; app role rules still decide access.
- Q: Should the new frontend-submitted authentication REST calls be exposed through the gateway or called directly on the IdentityService authority? → A: Expose login, registration, refresh, and logout as `/api/v1/accounts/**` routes through ApiGateway to IdentityService.
- Q: What CSRF protection level should the migration require for cookie-authenticated browser requests? → A: Require SameSite secure cookies plus a server-issued CSRF token sent in a custom header for all state-changing browser requests.
- Q: What should happen when a browser opens old IdentityService-hosted login, registration, or logout UI URLs after migration? → A: Redirect to the appropriate frontend route using return URL or app/client hint when available, with a safe default frontend fallback.
- Q: How should browser sessions be renewed after the initial cookie-auth login? → A: Frontend apps call a gateway `/api/v1/accounts/**` refresh route before expiry or after reload; failure signs the user out.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sign In From Frontend-Owned UI (Priority: P1)

As an admin, seller, or buyer, I want to sign in from the frontend app I am using, so that authentication feels like part of that app and does not redirect me to an IdentityService-hosted page.

**Why this priority**: This is the primary behavior change and is required before protected app workflows can use the new session model.

**Independent Test**: Can be tested by opening each frontend app while signed out, signing in with a valid password account, and confirming the user reaches the correct protected app area without seeing an IdentityService UI page.

**Acceptance Scenarios**:

1. **Given** a signed-out user is in the admin, seller, or buyer app, **When** they enter valid credentials in that app's sign-in form, **Then** the app establishes an authenticated browser session and routes them according to their app role rules.
2. **Given** a signed-out user enters invalid credentials, **When** they submit the sign-in form, **Then** the app shows a clear authentication failure and does not establish a session.
3. **Given** an account is inactive, locked, or not allowed for the selected app, **When** the user attempts to sign in, **Then** the app displays a safe failure message and keeps the user signed out.
4. **Given** a signed-out user submits the app sign-in form, **When** the browser sends the authentication request, **Then** it uses a versioned `/api/v1/accounts/**` gateway route rather than a direct IdentityService authority URL.

---

### User Story 2 - Register From Frontend-Owned UI (Priority: P2)

As a buyer or seller candidate, I want to create an account from the frontend app, so that registration no longer depends on an IdentityService-hosted page.

**Why this priority**: Registration is the second major UI page being moved out of IdentityService and must continue creating identity-owned accounts and matching user profiles.

**Independent Test**: Can be tested by completing a frontend registration form with a new email address and confirming the user is signed in, has an identity account, and receives the normal profile lifecycle behavior.

**Acceptance Scenarios**:

1. **Given** a visitor provides valid registration details, **When** they submit the frontend registration form, **Then** the system creates the identity-owned account, starts the existing profile creation flow, and establishes an authenticated browser session.
2. **Given** the email is already registered, **When** the visitor submits registration, **Then** the app shows a clear conflict message and does not create a duplicate account.
3. **Given** the password or confirmation is invalid, **When** the visitor submits registration, **Then** the app shows validation feedback and no authenticated session is established.
4. **Given** a visitor submits a registration form, **When** the browser sends the registration request, **Then** it uses a versioned `/api/v1/accounts/**` gateway route rather than a direct IdentityService authority URL.

---

### User Story 3 - Use Protected APIs Through Gateway Session (Priority: P3)

As a signed-in user, I want protected app actions to work across HiveSpace frontend apps without browser-readable tokens, so that my session is safer while existing service permissions still apply.

**Why this priority**: The new auth flow only succeeds if authenticated requests continue through the gateway and downstream services receive the same authorization context they require today.

**Independent Test**: Can be tested by signing in, calling protected profile/cart/product/order/admin actions through the app, refreshing the page, and confirming the session remains usable until logout or expiry.

**Acceptance Scenarios**:

1. **Given** a user is signed in, **When** the app calls a protected backend capability through the gateway, **Then** the request is accepted according to the user's role and account state.
2. **Given** a user is signed in, **When** browser scripts inspect local or session storage, **Then** no access token or refresh token is available to script-readable storage.
3. **Given** the user's browser session expires or becomes invalid, **When** the app calls a protected capability, **Then** the app treats the user as signed out and routes them back to sign in.
4. **Given** a user signs in through one HiveSpace frontend app, **When** the browser calls gateway APIs from another HiveSpace frontend app, **Then** the shared platform session is accepted and the target app's role and account-state rules determine access.
5. **Given** a signed-in user reloads a frontend app or nears session expiry, **When** the app calls the gateway refresh route, **Then** the session is renewed without exposing tokens to browser scripts or the user is treated as signed out if renewal fails.

---

### User Story 4 - Sign Out And Session Cleanup (Priority: P4)

As a signed-in user, I want sign-out to end my browser session, so that later requests from the same browser are no longer authenticated.

**Why this priority**: Session cleanup is necessary for user control, shared-device safety, and predictable route guard behavior.

**Independent Test**: Can be tested by signing in, signing out, refreshing the app, and confirming protected routes require sign-in again.

**Acceptance Scenarios**:

1. **Given** a user is signed in, **When** they choose sign out, **Then** the authenticated browser session is cleared and the app returns to its signed-out state.
2. **Given** a user has signed out, **When** the app calls a protected backend capability, **Then** the request is rejected as unauthenticated.

### Edge Cases

- A user opens an old IdentityService account URL directly after UI migration and is redirected to the appropriate frontend route using return URL or app/client hint when available, with a safe default frontend fallback.
- A user has a valid browser session but no longer satisfies the target app's role rules.
- A user signs in successfully before their profile projection is available.
- A user refreshes the browser or opens a second tab after signing in.
- A user's session expires while they are filling out a form and the next refresh attempt fails.
- A protected state-changing request is submitted without the expected server-issued CSRF token header.
- Existing email verification and seller-registration flows require fresh authorization state after account or role changes.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: IdentityService MUST stop being the owner of user-facing login, registration, and logout UI pages for HiveSpace browser apps.
- **FR-002**: Admin, seller, and buyer frontend apps MUST own the sign-in experience used by their audiences.
- **FR-003**: Frontend apps that offer account creation MUST own the registration experience for their audiences.
- **FR-004**: Users MUST be able to authenticate with email and password from the frontend app.
- **FR-005**: Successful password sign-in MUST return a successful response and establish a secure browser session without exposing access tokens or refresh tokens to browser scripts.
- **FR-006**: Successful registration MUST create an identity-owned account, preserve the existing matching-profile creation behavior, and establish a secure browser session.
- **FR-007**: Failed sign-in and registration attempts MUST return clear, safe, user-actionable errors without revealing sensitive account details.
- **FR-008**: Authenticated browser requests through the gateway MUST be forwarded to downstream services with the authorization context required by existing service policies.
- **FR-009**: Downstream service role and account-state authorization behavior MUST remain consistent with the current authenticated-user experience.
- **FR-010**: Frontend apps MUST renew browser sessions by calling a gateway `/api/v1/accounts/**` refresh route before expiry or after reload, and MUST treat failed renewal as signed-out state without exposing tokens to browser scripts.
- **FR-011**: The system MUST support sign-out that clears the browser session and prevents further authenticated requests.
- **FR-012**: The system MUST protect all cookie-authenticated browser state-changing requests with SameSite secure cookies plus a server-issued CSRF token sent in a custom header.
- **FR-013**: Existing email verification and seller access transition flows MUST continue to work after the auth UI migration.
- **FR-014**: Old IdentityService-hosted login, registration, and logout UI entry points MUST redirect browsers to the appropriate frontend route using return URL or app/client hint when available, with a safe default frontend fallback.
- **FR-015**: The migration MUST cover admin, seller, and buyer apps in the same feature so users do not experience mixed authentication patterns across HiveSpace apps.
- **FR-016**: A browser session established by any HiveSpace frontend app MUST be accepted by the gateway for admin, seller, and buyer app requests, with each target app still enforcing its role and account-state access rules.
- **FR-017**: Frontend-submitted login, registration, session refresh, and logout requests MUST use versioned `/api/v1/accounts/**` routes through ApiGateway to IdentityService.

### Key Entities

- **Browser Session**: The shared platform signed-in browser state that proves the user authenticated and allows the gateway to forward authorized requests for admin, seller, and buyer apps.
- **Identity Account**: The IdentityService-owned record for credentials, account status, roles, claims, lockout, and email verification.
- **Frontend Auth Form**: The app-owned login or registration screen that collects user input and starts authentication.
- **Authorization Context**: The user identity, roles, claims, and account attributes that downstream services use for access decisions.
- **Browser Auth Route**: A versioned `/api/v1/accounts/**` gateway route used by frontend apps for login, registration, session refresh, and logout while IdentityService owns the underlying account/session behavior.
- **CSRF Token**: A server-issued anti-forgery value that frontend apps send in a custom header on cookie-authenticated state-changing browser requests.
- **Session Renewal**: A frontend-initiated gateway account-route call that refreshes the shared browser session before expiry or after reload without exposing tokens to browser scripts.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of normal admin, seller, and buyer sign-in attempts are completed without rendering an IdentityService-hosted UI page.
- **SC-002**: 95% of successful sign-ins reach the appropriate post-login app destination within 5 seconds under normal local development conditions.
- **SC-003**: 100% of successful frontend registrations create exactly one identity account and trigger the existing matching-profile behavior.
- **SC-004**: 100% of browser-accessible storage checks after sign-in show no access token or refresh token stored in script-readable storage.
- **SC-005**: 100% of protected app workflows tested through the gateway preserve existing role-based access outcomes.
- **SC-006**: 100% of sign-out attempts clear the browser session so protected routes require sign-in again after refresh.
- **SC-007**: 100% of frontend login, registration, refresh, and logout calls observed in browser network traces target `/api/v1/accounts/**` gateway routes.
- **SC-008**: 100% of tested cookie-authenticated state-changing browser requests without the required CSRF header are rejected before downstream state changes occur.
- **SC-009**: 100% of tested old IdentityService-hosted login, registration, and logout UI URLs redirect to a frontend route rather than rendering IdentityService UI.
- **SC-010**: 100% of tested frontend reload and near-expiry renewal paths call a gateway `/api/v1/accounts/**` refresh route and either preserve the session or route to signed-out state on failure.

## Assumptions

- All three frontend apps are in scope for sign-in migration.
- Authenticated browser sessions are shared across admin, seller, and buyer apps at the gateway/platform level; app-specific access remains controlled by role and account-state rules.
- Frontend apps call gateway account routes for browser auth actions; IdentityService authority endpoints remain direct only for OIDC discovery/protocol needs that are still required.
- Session renewal is frontend-initiated through gateway account routes rather than automatic gateway renewal on arbitrary API requests.
- Public self-registration applies only where the frontend app already offers or is intended to offer registration; admin account creation remains governed by existing admin identity management rules.
- IdentityService remains the source of truth for accounts, credentials, roles, claims, lockout, account status, email verification, and token/session issuance.
- UserService remains the source of truth for profiles, settings, addresses, and stores.
- Downstream services continue to make authorization decisions from the forwarded authorization context rather than reading browser cookies directly.
- Social/external login is outside the first migration unless explicitly added during clarification or planning.
