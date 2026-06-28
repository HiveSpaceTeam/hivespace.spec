# Feature Specification: Email OTP Sign-In

- **Feature Branch**: `0009-signin-otp-email`
- **Created**: 2026-06-18
- **Status**: Implemented
- **Implemented**: 2026-06-28
- **Input**: User description: "I want to add a feature to allow user to signin with otp code send to email"

## Clarifications

### Session 2026-06-18

- Q: How does the code-entry screen correlate to a specific OTP - via a challenge token returned with the OTP request, email re-submission, or a server-side cookie? -> A: Challenge token. The OTP request response includes an opaque short-lived challenge token; the code-entry screen submits the token alongside the code. The backend looks up the OTP by token, not by email.
- Q: Does the code-entry screen proactively show an OTP expiry countdown, or is expiry discovered only on submission? -> A: Visible countdown timer. The code-entry screen displays a live countdown from the expiry timestamp returned with the OTP request. When it reaches zero, the screen immediately shows an expired state and prompts the user to request a new code without waiting for a failed submission.
- Q: After successful OTP sign-in, where does the user land - a fixed home page or the originally intended destination? -> A: Honor the return URL when present, fall back to home.
- Q: Should this feature design preserve future OTP-based auth flows? -> A: Yes. Keep public v1 behavior sign-in-specific, but structure the internal model for future IdentityService auth flows.

## User Scenarios and Testing

### User Story 1 - Discover and Start OTP Sign-In from the Sign-In Page (Priority: P1)

A buyer or seller on the existing sign-in page sees a visible option to sign in with a one-time code instead of a password. Clicking it opens an OTP sign-in screen where they enter their email address and request the code.

**Why this priority**: If users cannot find the flow, the feature is effectively absent.

**Independent Test**: Visit the sign-in page, enter a registered email, and confirm that the OTP request completes without exposing account existence.

### User Story 2 - Enter OTP Code to Complete Sign-In (Priority: P1)

After requesting a code, the user enters it on a dedicated screen with one input per digit, automatic focus movement, a live expiry countdown, and a back link to password sign-in. On success, the user receives the same browser session as password sign-in.

**Why this priority**: Verification and session issuance are the security-critical path.

**Independent Test**: Request an OTP, enter the code, and confirm the user lands on the authenticated destination.

### User Story 3 - Resend OTP Code (Priority: P2)

A user whose code expired, was invalidated, or never arrived can request a new code from the code-entry screen once the cooldown passes.

**Why this priority**: Improves completion reliability without changing the core auth model.

**Independent Test**: Wait for cooldown expiry, request resend, and confirm the previous code no longer works.

## Requirements

### Functional Requirements

- **FR-001**: The existing sign-in page MUST display a visible link or prompt that allows users to switch to the OTP sign-in flow.
- **FR-001a**: The OTP sign-in screen MUST provide a link back to the standard password sign-in page.
- **FR-001b**: The code-entry screen MUST present individual input boxes - one per code digit - with automatic focus progression.
- **FR-002**: The system MUST send a numeric OTP code to the submitted email address only when a buyer or seller account exists for that address.
- **FR-003**: The system MUST NOT reveal whether a submitted email address corresponds to a registered account when responding to an OTP request.
- **FR-004a**: The system MUST enforce a per-email request cooldown; successive requests within the cooldown window MUST be silently absorbed with a response indistinguishable from a successful send.
- **FR-004b**: A successful OTP request MUST include `canResendAt` so the frontend can render a resend countdown without an additional server round-trip.
- **FR-004c**: A successful OTP request MUST return an opaque short-lived challenge token. The backend MUST verify by token, not by email.
- **FR-005**: OTP codes MUST expire after a fixed time window and be rejected once expired. The request response MUST include the expiry timestamp so the frontend can display a live countdown.
- **FR-006**: OTP codes MUST be single-use.
- **FR-007**: The system MUST limit incorrect OTP attempts per code; exceeding the limit MUST invalidate the current code and require a new request.
- **FR-008**: On successful OTP verification, the system MUST issue a browser session equivalent to successful password sign-in and redirect to the stored valid same-origin destination when present.
- **FR-009**: The user MUST be able to request a resend from the code-entry step once the cooldown has passed.
- **FR-010**: OTP sign-in MUST NOT be available to admin accounts.
- **FR-011**: OTP sign-in for locked or deactivated accounts MUST silently decline to send a code and return the same generic confirmation as for unregistered addresses.

### Key Entities

- **OTP Request**: A pending one-time sign-in attempt tied to an email address; it includes a challenge token, a code, expiry timestamp, attempt count, and used/invalidated state.
- **Cooldown State**: The resend timing associated with an OTP request so the UI can control when another code may be requested.

## Success Criteria

- **SC-001**: A user with a registered email can complete the full OTP sign-in flow in under 2 minutes under normal delivery conditions.
- **SC-002**: 95% of OTP emails are delivered within 30 seconds of the request.
- **SC-003**: An expired or already-used OTP code is rejected 100% of the time with no browser session issued.
- **SC-004**: An OTP request for an unregistered email returns a response indistinguishable from a registered address response.
- **SC-005**: After the maximum failed attempts threshold is reached, the current code cannot be used to sign in and a new code must be requested.
- **SC-006**: No browser session is issued for admin accounts through the OTP flow.

## Assumptions

- OTP sign-in is a supplemental sign-in method alongside email/password.
- Only buyers and sellers may use OTP sign-in.
- Admin accounts remain password-only.
- OTP codes are 6-digit numeric values.
- The default OTP validity window is 10 minutes.
- The maximum incorrect attempt count per code is 5 unless implementation config chooses a different secure default and the plan reflects it.
- Email delivery is handled by NotificationService using existing delivery infrastructure.
- The implementation should not unnecessarily block future OTP-based identity flows, but this specification remains limited to OTP sign-in behavior.
