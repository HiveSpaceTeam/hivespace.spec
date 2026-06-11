# Feature Specification: Email Verification Before First Sign-In

- **Feature Branch**: `0007-email-verification-signup`
- **Created**: 2026-06-08
- **Status**: Implemented
- **Implemented**: 2026-06-11
- **Input**: User description: "Require buyer and seller email/password account registration to create a pending account, send a verification email, show confirmation that the email was sent, and block sign-in until email verification is completed."

## Clarifications

### Session 2026-06-08

- Q: When should downstream user-profile creation and related registration side effects happen for email/password sign-up? -> A: Create only the pending identity account at registration; create profile and publish downstream events after successful email verification.
- Q: If someone registers with an email that already belongs to a pending email/password account, what should the product do? -> A: Do not create another account; show that the account is pending and offer or route the user to request another verification email.
- Q: What protection should the unauthenticated resend-verification flow use by default? -> A: Allow resend by email without sign-in, but enforce a per-email cooldown and return the same generic success response either way.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Register and receive verification email (Priority: P1)

As a buyer or seller signing up with email and password, I want the system to create my account and send me a verification link so I can activate my account securely before first sign-in.

**Why this priority**: This is the core business behavior of the feature. Without this flow, the account lifecycle remains unchanged and unverified users can still access the system.

**Independent Test**: A new buyer or seller can submit the sign-up form and receive a clear confirmation that the account was created in a pending state and that a verification email has been sent.

**Acceptance Scenarios**:

1. **Given** a new buyer provides valid sign-up details, **When** they submit the sign-up form, **Then** the system creates a pending account, sends a verification email, and shows a confirmation message instead of signing them in.
2. **Given** a new seller provides valid sign-up details, **When** they submit the sign-up form, **Then** the system creates a pending account, sends a verification email, and shows a confirmation message instead of signing them in.
3. **Given** a user signs up with an email address that already belongs to a pending email/password account, **When** they submit the sign-up form, **Then** the system does not create a duplicate account and routes the user to request another verification email.
4. **Given** a user signs up with an email address that already belongs to an active account, **When** they submit the sign-up form, **Then** the system explains that the account cannot be created with that email and does not create a duplicate account.

---

### User Story 2 - Verify email and activate account (Priority: P1)

As a newly registered user, I want the verification link to activate my account so I can sign in only after proving ownership of my email address.

**Why this priority**: Sending a verification email only has value if completing verification changes the account from pending to usable.

**Independent Test**: A pending buyer or seller can open the verification link and complete account activation without any manual support intervention.

**Acceptance Scenarios**:

1. **Given** a user has a pending account and a valid verification link, **When** they open the link, **Then** the system marks the account as verified and active and completes downstream profile creation and related registration side effects.
2. **Given** a user has just completed successful verification, **When** the verification flow finishes, **Then** the user is sent to sign-in and informed that the account is now ready to use.
3. **Given** a user opens a verification link that is invalid, expired, or already used, **When** the system cannot complete verification, **Then** the user is shown a clear failure message and a way to request another verification email.

---

### User Story 3 - Prevent sign-in before verification (Priority: P2)

As a newly registered user, I want the system to clearly block sign-in before verification so I understand that activation is required before account access is allowed.

**Why this priority**: Blocking unverified access is the policy outcome the feature is meant to enforce after registration.

**Independent Test**: A pending user can attempt to sign in and is consistently blocked until the account has been verified.

**Acceptance Scenarios**:

1. **Given** a user has a pending account, **When** they try to sign in before verifying their email, **Then** the system denies access and explains that email verification is still required.
2. **Given** a user has a pending account, **When** they request another verification email, **Then** the system accepts the request without sign-in, applies cooldown protection, and returns the same generic success response regardless of account state.
3. **Given** a user has completed verification, **When** they sign in with valid credentials, **Then** the system allows access normally.

### Edge Cases

- A user signs up successfully but does not receive the first verification email and must request another one.
- A user tries to request another verification email before the resend cooldown has elapsed.
- A user clicks a verification link after the account has already been verified.
- A user clicks a verification link after the link has expired.
- A user tries to sign in repeatedly while the account is still pending.
- A user attempts to register with an email address that already belongs to a pending email/password account and must be routed to resend verification without creating a duplicate account.
- A resend-verification request for an unknown, active, or pending email address returns the same generic success response.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST require email verification before a buyer or seller who registered with email and password can sign in for the first time.
- **FR-002**: The system MUST create newly registered buyer and seller email/password accounts in a pending state rather than granting immediate account access.
- **FR-003**: The system MUST create only the pending identity account during successful email/password registration and MUST defer downstream profile creation and related registration side effects until successful email verification.
- **FR-004**: The system MUST send a verification email to the address provided during successful email/password registration.
- **FR-005**: The system MUST show a clear confirmation state after successful registration that tells the user a verification email has been sent.
- **FR-006**: The system MUST NOT sign the user in automatically after successful email/password registration.
- **FR-007**: The system MUST activate a pending account only after the user completes email verification successfully.
- **FR-008**: The system MUST complete downstream profile creation and related registration side effects only after successful email verification.
- **FR-009**: The system MUST send the user to sign-in after successful verification and tell them that their account is now ready to use.
- **FR-010**: The system MUST block sign-in attempts for pending accounts and explain that email verification is still required.
- **FR-011**: The system MUST allow a pending user to request another verification email without requiring a successful sign-in first.
- **FR-012**: The system MUST expose resend-verification for pending users without sign-in, enforce a per-email cooldown, and return the same generic success response regardless of whether the email belongs to an unknown, active, or pending account.
- **FR-013**: The system MUST give the user a clear recovery path when a verification link is invalid, expired, or already used.
- **FR-014**: The system MUST prevent creation of duplicate accounts for an email address that is already associated with an existing account.
- **FR-015**: The system MUST route users who attempt to register with an email already tied to a pending email/password account into the resend-verification recovery path instead of creating or replacing the pending account.
- **FR-016**: The feature MUST apply to buyer and seller email/password registration flows.
- **FR-017**: The feature MUST NOT change Google-based account creation, sign-in, or account-linking behavior.
- **FR-018**: The feature MUST preserve the existing post-verification experience for users whose account becomes eligible for normal access after verification is completed.

### Key Entities

- **Pending Account**: A newly registered buyer or seller account that exists but cannot sign in until email verification is completed.
- **Verification Email**: The activation message sent to a newly registered or still-pending user so they can verify ownership of their email address.
- **Verification Link**: The one-time activation path the user opens to complete email verification.
- **Activation State**: The account lifecycle state that determines whether the user is still pending or is allowed to sign in.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of successful buyer and seller email/password registrations end in a pending account state rather than an authenticated session.
- **SC-002**: 100% of pending users who attempt to sign in before verification are blocked from account access.
- **SC-003**: 100% of users who complete verification through a valid link can proceed to sign-in without manual support action.
- **SC-004**: At least 95% of successful registrations result in the user reaching a confirmation state that clearly explains the next step is email verification.
- **SC-005**: Support requests caused by confusion about first-time account activation decrease compared with the prior immediate-sign-in registration flow.

## Assumptions

- The scope is limited to buyer and seller email/password registration journeys.
- Existing email verification messaging and verification experience can be extended for this feature rather than replaced with a separate user-facing concept.
- Downstream profile creation and related registration side effects move from registration time to successful email verification.
- A pending user can request another verification email without having an active session.
- The public resend-verification flow uses a per-email cooldown and a generic success response for anti-enumeration behavior.
- Admin account creation and access rules are outside the scope of this feature.
