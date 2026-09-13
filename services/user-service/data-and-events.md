# UserService Data And Events

## Data Ownership

UserService owns:

- User profile fields, including `AvatarFileId` and resolved `AvatarUrl`.
- User settings such as culture and theme.
- User addresses.
- Store registration records and store lifecycle data.
- Imported seller store provisioning keys used to create or match stores for catalog import workflows.
- Platform configuration records that hold config-level default/version state for UserService-owned list settings.
- Platform currency item rows that hold enabled/disabled state for supported currencies.
- User-owned profile, address, settings, and store seed data.

## Integration Events

### Published Events

| Event | Purpose |
|---|---|
| `UserCreatedIntegrationEvent` | Let downstream services create profile/display user projections after profile creation |
| `UserUpdatedIntegrationEvent` | Refresh downstream user projections |
| `StoreCreatedIntegrationEvent` | Let CatalogService/OrderService create store references and let IdentityService grant seller access |
| `StoreUpdatedIntegrationEvent` | Refresh downstream store references |
| `PlatformCurrencyPolicyUpdatedIntegrationEvent` | Let CatalogService, OrderService, and PaymentService refresh local currency-policy validation projections after admin configuration changes |

### Consumed Events

| Event | Producer | Purpose |
|---|---|---|
| `IdentityUserReadyIntegrationEvent` | IdentityService | Create or verify the matching UserService profile for the shared public user ID once the account is usable |
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
- Platform currency policy state is authoritative only in UserService even though other services keep local validation projections.
- IdentityService is authoritative for authentication, roles, claims, lockout, account status, and email verification.
- Store registration is the only supported UserService trigger for seller/store-owner role propagation.
- Imported seller store provisioning also publishes the existing `StoreCreatedIntegrationEvent` for newly created stores; it does not grant identity roles directly.
- Other services must not assume store/user display data without a projection event or public API contract.

## Publisher Policy

- UserService application publishing uses service-owned publisher abstractions such as `IStoreEventPublisher` for user/store integration events.
- Platform currency policy publishing should use a service-owned publisher abstraction so outbox-backed event publication stays outside direct endpoint code.
