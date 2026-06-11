# Research: Email Verification Before First Sign-In

## Decision: Use a separate anonymous resend endpoint

**Rationale:** The existing `POST /api/v1/accounts/email-verification` route is authenticated and represents an identity-owned send action for already-authenticated flows. Public resend for pending users needs different auth, anti-enumeration behavior, and cooldown handling. A separate `POST /api/v1/accounts/email-verification/resend` contract keeps those semantics explicit and avoids weakening the authenticated endpoint.

**Alternatives considered:**

| Option | Why rejected |
| --- | --- |
| Reuse `POST /api/v1/accounts/email-verification` and make it anonymous | Mixes authenticated and anonymous behaviors into one contract and makes cooldown/generic-success requirements harder to document and test cleanly. |
| Add resend behavior only inside registration failure responses | Does not satisfy the requirement for an independent public recovery path when the first email is not received or the link expires. |

## Decision: Replace the old profile-creation event entirely with a single downstream readiness event

**Rationale:** The clarified spec says registration should create only a pending identity account, while downstream profile creation and related side effects must happen only after the account is actually usable. The cleanest boundary is a single readiness fact, `IdentityUserReadyIntegrationEvent`, published by IdentityService when downstream provisioning may begin. For email/password accounts, that happens after successful verification. For new Google-created accounts, that happens immediately after successful account creation because the account is already usable. Keeping `IdentityUserCreatedIntegrationEvent` alongside this new event would preserve duplicate semantics, so the old event should be removed entirely.

**Alternatives considered:**

| Option | Why rejected |
| --- | --- |
| Reuse `IdentityUserCreatedIntegrationEvent` but publish it later for password accounts | Blurs the meaning of "created" across flows and makes Google-based immediate creation harder to reason about. |
| Use separate created and activated provisioning events | Adds unnecessary branching for UserService, even though both flows mean the same downstream thing: the account is ready for provisioning. |
| Reuse `UserEmailVerifiedIntegrationEvent` as the profile-creation trigger | Couples downstream provisioning to all email-verified scenarios, including cases like existing-account linking, where profile creation is not the intended reaction. |
| Add a saga between IdentityService and UserService | Over-coordinates a simple readiness-to-projection handoff that does not need compensation or long-running orchestration. |

## Decision: Keep verification delivery contracts unchanged

**Rationale:** `UserEmailVerificationRequestedIntegrationEvent` already models the need for NotificationService to deliver a verification email. `UserEmailVerifiedIntegrationEvent` can remain as a separate identity-owned fact only if other consumers still need verified-email state independent of downstream provisioning. The feature changes when profile creation happens, not how notification delivery works.

**Alternatives considered:**

| Option | Why rejected |
| --- | --- |
| Introduce a new resend-specific delivery event | Duplicates the existing delivery fact without adding meaningful ownership separation. |
| Force all consumers to infer provisioning from `UserEmailVerifiedIntegrationEvent` | Makes downstream consumers infer too much from a verification-state fact and increases accidental coupling. |

## Decision: Block pending-account login in IdentityService with a stable recoverable error

**Rationale:** Sign-in admission is identity-owned. The frontend needs a predictable way to route the user into resend-verification recovery, but UserService or ApiGateway should not own that decision. A stable business error from `POST /api/v1/accounts/login` keeps the policy localized to IdentityService.

**Alternatives considered:**

| Option | Why rejected |
| --- | --- |
| Let login succeed with a limited session | Violates the requirement that email/password accounts cannot sign in before verification. |
| Let ApiGateway block pending users after cookie issuance | Pushes account-state policy out of the owning service and still creates a session that the product should not allow. |

## Decision: No saga is required

**Rationale:** The feature uses a single-service registration and verification workflow plus ordinary downstream event publication after the account becomes usable. There is no long-running state machine, multi-step compensation, or timeout-driven orchestration that would justify a MassTransit saga.

**Alternatives considered:**

| Option | Why rejected |
| --- | --- |
| Add a registration saga to coordinate pending account, email delivery, and profile creation | Email delivery and profile creation are already modeled as separate async concerns, and the failure modes are observable without saga-owned state. |
