# IdentityService Data And Events

## Data Ownership

IdentityService owns:

- ASP.NET Identity users, roles, claims, logins, tokens, and lockout fields.
- Duende IdentityServer configuration and operational grants.
- Identity-owned account status used by authentication and authorization.
- Email verification state and verification tokens.
- Identity seed data for users, roles, claims, account status, seller authorization references, and OIDC configuration.
- Google external login links and temporary pending-link state.

## Integration Events

### Published Events

| Event | Purpose |
|---|---|
| `IdentityUserCreatedIntegrationEvent` | Let UserService create a matching profile with the same public user ID |
| `UserEmailVerificationRequestedIntegrationEvent` | Trigger NotificationService verification email/notification delivery |
| `UserEmailVerifiedIntegrationEvent` | Signal identity-owned verified email state |

### Consumed Events

| Event | Producer | Purpose |
|---|---|---|
| `StoreCreatedIntegrationEvent` | UserService | Grant seller role/claims and store reference after successful store registration |

## Invariants

- Identity state is authoritative only in IdentityService.
- Profile, settings, addresses, and store lifecycle records are authoritative only in UserService.
- Cross-boundary updates must use integration events with transactional outbox publication and idempotent consumers.
- Missing required event data must remain observable through retry/dead-letter behavior.
- Google account creation reuses `IdentityUserCreatedIntegrationEvent` unchanged for UserService profile creation.
- Google account creation is blocked when the verified Google email already belongs to an existing unlinked local password account; declined, failed, abandoned, or expired linking must not create a duplicate same-email identity.
- Google account linking does not introduce a new integration event. If linking marks email verified and implementation publishes an email-verified fact, it must reuse `UserEmailVerifiedIntegrationEvent` unchanged.

## Publisher Policy

- IdentityService application flows publish identity-owned integration events through a service-owned identity event publisher.
