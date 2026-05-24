# UserService Data And Events

## Data Ownership

UserService owns:

- User profile fields, including `AvatarFileId` and resolved `AvatarUrl`.
- User settings such as culture and theme.
- User addresses.
- Store registration records and store lifecycle data.
- User-owned profile, address, settings, and store seed data.

## Integration Events

### Published Events

| Event | Purpose |
|---|---|
| `UserCreatedIntegrationEvent` | Let downstream services create profile/display user projections after profile creation |
| `UserUpdatedIntegrationEvent` | Refresh downstream user projections |
| `StoreCreatedIntegrationEvent` | Let CatalogService/OrderService create store references and let IdentityService grant seller access |
| `StoreUpdatedIntegrationEvent` | Refresh downstream store references |

### Consumed Events

| Event | Producer | Purpose |
|---|---|---|
| `IdentityUserCreatedIntegrationEvent` | IdentityService | Create or verify the matching UserService profile for the shared public user ID |
| `MediaAssetProcessedIntegrationEvent` | MediaService | For `EntityType = "user_avatar"`, update the matching user's `AvatarUrl` only when `AvatarFileId` equals the event `FileId` |

## Projection Consumers

| Consumer | Projection |
|---|---|
| CatalogService | `store_refs` |
| OrderService | `store_refs` |
| IdentityService | Seller role/claims from `StoreCreatedIntegrationEvent` |
| NotificationService | `user_refs` |

## Invariants

- Profile, settings, address, and store state are authoritative only in UserService.
- IdentityService is authoritative for authentication, roles, claims, lockout, account status, and email verification.
- Store registration is the only supported UserService trigger for seller/store-owner role propagation.
- Other services must not assume store/user display data without a projection event or public API contract.
