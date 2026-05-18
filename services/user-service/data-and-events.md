# UserService Data And Events

## Data Ownership

UserService owns:

- ASP.NET Identity users, roles, claims, logins, tokens.
- Duende IdentityServer configuration and operational grants.
- User profile fields, including email verification state, `AvatarFileId`, and resolved `AvatarUrl`.
- User settings such as culture and theme.
- User addresses.
- Store registration records and seller role transition state.

## Integration Events

### Published Events

| Event | Purpose |
|---|---|
| `UserCreatedIntegrationEvent` | Let downstream services create user projections |
| `UserUpdatedIntegrationEvent` | Refresh downstream user projections |
| `UserEmailVerificationRequestedIntegrationEvent` | Trigger verification notification/email |
| `UserEmailVerifiedIntegrationEvent` | Signal verified email state |
| `StoreCreatedIntegrationEvent` | Let CatalogService/OrderService create store references |
| `StoreUpdatedIntegrationEvent` | Refresh downstream store references |

### Consumed Events

| Event | Producer | Purpose |
|---|---|---|
| `MediaAssetProcessedIntegrationEvent` | MediaService | For `EntityType = "user_avatar"`, update the matching user's `AvatarUrl` only when `AvatarFileId` equals the event `FileId` |

## Projection Consumers

| Consumer | Projection |
|---|---|
| CatalogService | `store_refs` |
| OrderService | `store_refs` |
| NotificationService | `user_refs` |

## Invariants

- User identity state is authoritative only in UserService.
- Store registration is the only supported transition into seller/store-owner capability.
- Other services must not assume store/user display data without a projection event or public API contract.
