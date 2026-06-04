# Feature Specification: Google Login and Account Linking

- **Feature Branch**: `0005-google-login-linking`
- **Created**: 2026-05-31
- **Status**: Implemented
- **Implemented**: 2026-06-04
- **Input**: User description: "I want to add a feature to allow user to login the system with Google based on the current flow in Identity Service, then remove all the old flow of Identity server with old URL and all Pages, if user already create account with email/password then sign in with Google with that email, system must display user to ask if they want to link the account, then require them to type in password, then let them login and link"

## Clarifications

### Session 2026-05-31

- Q: Which account audiences should be allowed to use Google sign-in? -> A: Buyer and seller accounts only.
- Q: What account type should be created when a new user signs up with Google from the seller app? -> A: Create a normal user account; seller status still requires existing seller onboarding.
- Q: How should old user-facing IdentityServer account URLs behave after legacy page removal? -> A: No compatibility required during development; old user-facing account URLs may return a non-user-facing error or not-found response.
- Q: What should happen to an existing account's email verification state after successful Google linking with a verified Google email? -> A: Mark the existing account email as verified.

### Session 2026-06-01

- Q: When should Google sign-in show the account-link verification prompt? -> A: Only when the Google verified email matches an existing HiveSpace email/password account that has no Google link. If there is no matching email/password account, continue the normal Google sign-in/sign-up path.
- Q: Can the user continue Google sign-in without linking when the verified Google email matches an existing email/password account? -> A: No. The user must link the existing account or use password sign-in, password reset, or another Google account; the system must not create a duplicate Google-only account for the same email.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Sign In With Google (Priority: P1)

As a buyer or seller, I can choose Google from the normal HiveSpace sign-in or sign-up experience and enter the system without using the legacy hosted IdentityServer account pages.

**Why this priority**: This is the primary user value and must work before account linking or cleanup matters.

**Independent Test**: Can be fully tested by starting from the buyer or seller app sign-in page, choosing Google, completing Google authentication with an email that is not already registered, and reaching the expected signed-in app state.

**Acceptance Scenarios**:

1. **Given** an unauthenticated user with no matching HiveSpace email/password account for their Google email, **When** they complete Google sign-in, **Then** the system follows the normal Google path by signing in an already linked Google account or creating a normal HiveSpace user account and signing them in.
2. **Given** an unauthenticated user on the buyer or seller app sign-in page, **When** they choose Google, **Then** they complete the flow without seeing old IdentityServer-hosted account pages.
3. **Given** an unauthenticated user on the buyer or seller app sign-up page, **When** they choose Google, **Then** they enter the same Google authentication flow and complete account creation, account linking, or error handling without seeing old IdentityServer-hosted account pages.
4. **Given** Google does not provide a usable email for the user, **When** the user returns to HiveSpace, **Then** sign-in is rejected with a clear message and no account is linked or created.

---

### User Story 2 - Link Existing Password Account (Priority: P1)

As a user who already registered with email and password, I can sign in with Google using the same email, confirm that I want to link the accounts, prove ownership with my password, and then continue signed in.

**Why this priority**: This protects existing accounts from accidental takeover while allowing current users to adopt Google login.

**Independent Test**: Can be fully tested by creating an email/password account, starting Google sign-in with the same email, completing the link prompt and password confirmation, and verifying the user is signed in with the same account.

**Acceptance Scenarios**:

1. **Given** an existing HiveSpace account uses local email/password credentials and has no Google login linked, **When** the user completes Google sign-in with the same verified email, **Then** the system asks whether they want to link Google to the existing account.
2. **Given** the user accepts linking and Google reports the same email as verified, **When** they enter the correct account password, **Then** Google is linked to the existing account, the account email is marked verified, and the user is signed in.
3. **Given** the user declines linking, **When** they leave the linking prompt, **Then** the existing account remains unchanged, no duplicate Google-only account is created, and the user is not signed in through Google.
4. **Given** the user accepts linking but enters an incorrect password, **When** the password check fails, **Then** Google is not linked and the user receives a clear retry or cancellation path.

---

### User Story 3 - Return With Already Linked Google Account (Priority: P2)

As a returning user with Google already linked, I can sign in with Google directly without repeating the linking prompt.

**Why this priority**: This confirms linking is durable and keeps future sign-ins fast.

**Independent Test**: Can be fully tested after User Story 2 by signing out, choosing Google again, and verifying the user reaches the app without password confirmation.

**Acceptance Scenarios**:

1. **Given** a user has already linked Google to their HiveSpace account, **When** they complete Google sign-in with the same Google identity, **Then** they are signed in without seeing the link-account prompt.
2. **Given** the linked HiveSpace account is disabled, locked, or otherwise not allowed to sign in, **When** the user completes Google sign-in, **Then** sign-in is blocked using the same account-status rules as password login.

---

### User Story 4 - Remove Legacy Account Page Flow (Priority: P2)

As a user, I only interact with the frontend-owned HiveSpace authentication screens and no longer depend on old IdentityServer account URLs or pages for login, registration, logout, or account-link decisions.

**Why this priority**: Removing the old hosted page flow prevents split experiences and stale URLs after Google sign-in is introduced.

**Independent Test**: Can be fully tested by navigating to known old account URLs and verifying they do not render legacy account pages or become part of the supported user journey.

**Acceptance Scenarios**:

1. **Given** a user navigates to a legacy account login, registration, logout, consent, grants, device, diagnostics, or external-login page URL, **When** the request is handled, **Then** the URL does not render a legacy account page and may return a non-user-facing error or not-found response.
2. **Given** the buyer or seller app requires authentication, **When** it sends a user to sign in, **Then** the destination is the app's current HiveSpace sign-in screen with Google and password options where applicable.
3. **Given** the admin app requires authentication, **When** it sends an admin to sign in, **Then** the destination remains the supported admin sign-in flow without Google sign-in.

### Edge Cases

- Google returns an email that matches an existing local email/password account but the email is not verified by Google: the system must not link automatically and must require a safe fallback or rejection.
- The matching email/password account has no usable password, is locked, disabled, deleted, or pending administrative restrictions: linking must not bypass existing account-status rules.
- If no existing local email/password account matches the verified Google email, the system must not show the linking prompt and must continue the normal Google sign-in/sign-up outcome.
- If an existing local email/password account matches the verified Google email, the system must not create a second HiveSpace identity for that email after declined consent, wrong password, abandoned linking, or expired pending-link state.
- The user starts linking but closes the browser, navigates away, or the Google session expires: no partial link is completed.
- The user attempts repeated wrong passwords during linking: existing failed-login and lockout protections must apply consistently.
- A Google account email changes after linking: the previously linked Google identity remains tied only to the verified external identity, not to an arbitrary new email match.
- Multiple HiveSpace buyer or seller roles or app contexts share the same account: Google sign-in must preserve the user's existing roles and app access after successful login.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST offer Google from the current HiveSpace-owned sign-in and sign-up experiences for buyer and seller browser apps.
- **FR-002**: The system MUST complete Google sign-in using the same successful-session behavior as the current IdentityService login flow.
- **FR-003**: The system MUST create a new normal `ApplicationUser` identity account and link the Google external login when a user completes Google sign-in with a verified email that does not match an existing local email/password account and is not already linked to a Google login.
- **FR-004**: The system MUST detect when a Google sign-in email matches an existing local email/password account that does not already have Google linked.
- **FR-005**: The system MUST ask the user for explicit consent before linking Google to an existing email/password account.
- **FR-006**: The system MUST require the user to enter the existing account password before linking Google to that account.
- **FR-007**: The system MUST link Google only after both user consent and correct password confirmation succeed.
- **FR-008**: The system MUST sign the user in to the existing account immediately after a successful link.
- **FR-009**: The system MUST reject linking and preserve the existing account unchanged when consent is declined, password confirmation fails, or the linking flow is abandoned.
- **FR-010**: The system MUST allow future Google sign-ins for an already linked account without repeating the link prompt.
- **FR-011**: The system MUST enforce existing account status, lockout, role, and email-verification rules during Google sign-in and linking.
- **FR-012**: The system MUST ensure Google sign-in does not expose browser-readable access or refresh tokens.
- **FR-013**: The system MUST remove the legacy IdentityService `Pages` implementation from supported user journeys for login, registration, logout, consent, grants, device, diagnostics, and external-account linking while preserving required non-page IdentityServer protocol behavior.
- **FR-014**: The system MUST prevent old account URLs from rendering stale login, registration, logout, consent, or account-link pages, with no requirement to preserve compatibility redirects during development.
- **FR-021**: The system MUST delete the old `AccountCompatibilityEndpoints` compatibility redirect surface instead of redirecting `/Account/Login`, `/Account/Register`, or `/Account/Logout` to frontend pages.
- **FR-015**: The system MUST present clear user-facing outcomes for Google cancellation, Google failure, account mismatch, declined linking, incorrect password, and blocked account states.
- **FR-016**: The system MUST keep existing account profile creation and role propagation behavior intact when accounts are created or accessed through Google.
- **FR-017**: The system MUST NOT offer Google sign-in or Google account creation for admin accounts.
- **FR-018**: The system MUST create new Google-authenticated users as normal user accounts, even when Google sign-in starts from the seller app.
- **FR-019**: The system MUST require the existing seller onboarding flow before a Google-authenticated normal user receives seller access.
- **FR-020**: The system MUST mark an existing account email as verified when Google is successfully linked using the same verified Google email.
- **FR-022**: The system MUST NOT show the account-link prompt when no existing local email/password account matches the verified Google email; it MUST continue the normal Google sign-in/sign-up path instead.
- **FR-023**: The system MUST NOT allow a user to continue Google sign-in as a separate Google-only account when the verified Google email matches an existing unlinked local email/password account; the allowed exits are linking, password sign-in, password reset, or choosing another Google account.

### Key Entities *(include if feature involves data)*

- **HiveSpace Account**: The identity account used for authentication, account status, roles, credentials, and linked external providers.
- **Google Login Link**: The association between a HiveSpace account and a verified Google identity that can be used for future sign-ins.
- **Account Linking Request**: A temporary user decision flow created only when a Google verified email matches an existing local email/password account and requires consent plus password confirmation.
- **Browser Session**: The authenticated state established after successful password login, registration, Google login, or account linking.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of successful Google sign-ins for already linked accounts reach the target app signed-in state within 30 seconds after the user completes Google authentication.
- **SC-002**: 100% of attempted Google links to existing email/password accounts require both explicit user consent and correct password confirmation.
- **SC-003**: 0 successful account links occur after declined consent, wrong password confirmation, abandoned linking, blocked account status, or unusable Google email.
- **SC-004**: 100% of known legacy account page URLs no longer render legacy IdentityServer user-facing pages after the feature ships.
- **SC-005**: Users who create or first access accounts through Google receive the same downstream profile and role behavior as users who register through the current account flow.
- **SC-006**: Support or QA can verify password login, Google login, Google linking, logout, and session refresh from buyer and seller apps without using old IdentityServer account pages.
- **SC-007**: Support or QA can verify the admin app does not expose Google sign-in and still uses the supported admin sign-in flow.
- **SC-008**: 100% of successful links using a verified Google email result in the matching HiveSpace account email being verified.
- **SC-009**: 0 duplicate HiveSpace accounts are created from Google sign-in when the verified Google email already belongs to an existing local email/password account.

## Assumptions

- Google sign-in is available to buyer and seller browser apps only; admin authentication remains outside the Google sign-in scope.
- A Google email may be trusted for automatic new-account creation only when Google reports the email as verified.
- Matching an existing local email/password account by verified Google email is enough to start the linking prompt, but never enough to complete linking without password confirmation.
- A matching existing local email/password account blocks separate Google account creation for the same email until the user links, signs in with password, resets password, or uses another Google account.
- A verified Google email is acceptable proof for marking the matching HiveSpace account email as verified only after successful password-confirmed linking.
- Existing lockout, disabled-account, deleted-account, and role rules remain authoritative and are not relaxed for Google sign-in.
- Existing profile creation and account-created messaging remain the source of downstream profile setup for new identity accounts.
- New Google-authenticated users start as normal user accounts; seller role assignment remains owned by the existing seller/store onboarding flow.
- Removing old IdentityServer pages means deleting the IdentityService `Pages` folder and removing Razor Pages registration/mapping while preserving non-page protocol behavior still required by the authentication system.
- The project is still in development, so preserving old user-facing account URL compatibility is not required.
