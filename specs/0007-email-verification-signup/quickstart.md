# Quickstart: Email Verification Before First Sign-In

Use this checklist after implementation in `../hivespace.microservice` and `../hivespace.web`.

## Prerequisites

1. Read `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md`.
2. Read `../hivespace.web/AGENTS.md` and `../hivespace.web/CLAUDE.md`.
3. Start the backend through Aspire AppHost as documented in [../0006-aspire-setup/plan.md](../0006-aspire-setup/plan.md).
4. Start the buyer and seller frontend apps.

## Backend checks

1. Register a new buyer email/password account through `POST /api/v1/accounts/register`.
2. Confirm the response indicates pending verification and no auth cookies are issued.
3. Confirm IdentityService stores the account in pending state with `EmailConfirmed = false`.
4. Confirm `UserEmailVerificationRequestedIntegrationEvent` is published and NotificationService receives the delivery request.
5. Confirm no UserService profile is created yet for that user ID.

## Verification-to-readiness checks

1. Complete verification through `POST /api/v1/accounts/email-verification/verify`.
2. Confirm the account becomes active and verified in IdentityService.
3. Confirm `IdentityUserReadyIntegrationEvent` is published exactly once when the account becomes usable.
4. Confirm UserService creates the matching profile idempotently from the ready event.
5. If retained, confirm `UserEmailVerifiedIntegrationEvent` still publishes as the identity-owned verified-email fact.

## Immediate-ready checks

1. Create a new Google-based account through the unchanged external-account flow.
2. Confirm the newly created account is immediately usable without pending verification state.
3. Confirm `IdentityUserReadyIntegrationEvent` is published exactly once for that new account.
4. Confirm UserService creates the matching profile idempotently from the same ready-event contract.

## Recovery checks

1. Attempt login before verification and confirm the response is blocked with the verification-required business error.
2. Call `POST /api/v1/accounts/email-verification/resend` for:
   - the pending account email;
   - an unknown email;
   - an already-active email.
3. Confirm all resend requests return the same generic success response.
4. Confirm repeated resend requests during cooldown do not trigger repeated delivery.
5. Open an invalid or expired verification link flow and confirm the UI offers resend recovery.

## Frontend checks

1. Buyer registration lands on a verification-sent confirmation state, not an authenticated session.
2. Seller registration lands on the same confirmation pattern and still supports the later seller onboarding path after activation and sign-in.
3. Blocked login routes the user into resend-verification recovery in both buyer and seller apps.
4. English and Vietnamese copy both cover pending, resend, expired-link, and verified-success states.
