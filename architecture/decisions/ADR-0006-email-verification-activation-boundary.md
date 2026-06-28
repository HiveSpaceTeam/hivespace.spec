# ADR-0006: Use A Single Downstream Readiness Event For Account Provisioning

- **Status**: Draft
- **Date**: 2026-06-08
- **Feature**: `0007-email-verification-signup`
- **Deciders**: Project maintainers

## Context

HiveSpace currently documents email/password registration as creating the identity account, publishing downstream profile-creation side effects, and establishing a browser session immediately. The new feature requires buyer and seller email/password registration to create only a pending identity account, send verification email, avoid sign-in before verification, and start downstream user-profile creation only after successful email verification.

The design must also preserve existing Google-based creation and linking behavior, keep IdentityService as the owner of account lifecycle and verification state, and avoid pushing account-state policy into UserService or ApiGateway.

## Decision

Use a single readiness event, not raw account creation or separate created/activated provisioning events, as the downstream provisioning boundary.

IdentityService will:

- create pending email/password accounts at registration time;
- publish `UserEmailVerificationRequestedIntegrationEvent` for initial send and resend;
- block login while the account is pending;
- publish a new `IdentityUserReadyIntegrationEvent` when downstream provisioning may begin;
- publish that ready event after successful email verification for pending email/password accounts;
- publish that same ready event immediately after successful Google account creation when the account is already usable;
- continue publishing `UserEmailVerifiedIntegrationEvent` only if the platform still needs a separate identity-owned verification fact.

UserService will create the matching profile from `IdentityUserReadyIntegrationEvent` for both pending email/password and immediate Google-created account flows. `IdentityUserCreatedIntegrationEvent` should be removed entirely from the target design.

For this feature, the ready event contract carries the public user ID, email, optional username/full name, and readiness timestamp needed for downstream profile bootstrap. Buyer/seller app context and readiness-reason metadata are not part of the required story-0007 contract.

## Consequences

### Positive

- Keeps service ownership clean: IdentityService owns lifecycle admission; UserService reacts only when the account is ready for downstream provisioning.
- Gives UserService one canonical provisioning trigger regardless of whether readiness came from email verification or immediate Google account creation.
- Removes the old duplicate event so the shared contract set has one meaning for downstream profile provisioning.
- Avoids overloading `UserEmailVerifiedIntegrationEvent` with profile-creation semantics that do not apply to every verified-email scenario.
- Preserves existing Google-based account creation behavior without forcing all creation paths through the pending-account model.
- Makes the downstream readiness boundary explicit for testing, idempotency, and catalog documentation.

### Negative / Trade-offs

- Adds a new shared integration event and a new UserService consumer.
- Registration and verification API contracts change across backend and frontend repos.
- Planning and catalog updates must explain the distinction between account creation and downstream activation clearly.

### Risks

- Ready events could be published more than once if verification retries or immediate-create flows are not idempotent.
- UserService could accidentally continue creating profiles from the old password-registration path if implementation leaves stale consumers or handlers in place.
- Frontend apps could assume registration still creates a session unless shared auth contracts are updated consistently.

Mitigation: keep ready-event publication behind idempotent readiness transitions in IdentityService, make UserService readiness handling idempotent by public user ID, and update shared auth contracts before app-level page changes.

## Alternatives Considered

| Option | Why rejected |
| --- | --- |
| Reuse `IdentityUserCreatedIntegrationEvent` for delayed password activation | Blurs the meaning of "created" across account flows and makes immediate-create flows harder to reason about. |
| Use separate created and activated provisioning events | Adds unnecessary branching for UserService when both flows mean the same downstream thing: the account is ready for provisioning. |
| Reuse `UserEmailVerifiedIntegrationEvent` as the profile-creation trigger | Couples downstream provisioning to all verification outcomes, including cases where no new profile should be created. |
| Add a registration saga | Overly heavy for a single-service lifecycle change followed by ordinary event publication. |

## Follow-Up

- Add `IdentityUserReadyIntegrationEvent` to `shared/event-catalog.md` after task generation.
- Remove `IdentityUserCreatedIntegrationEvent` from `shared/event-catalog.md` and service docs as part of the same change.
- Update `shared/api-catalog.md` after task generation for the changed register/login/verify semantics and the new resend endpoint.
- Update IdentityService and UserService docs during the catalog/documentation phase.
