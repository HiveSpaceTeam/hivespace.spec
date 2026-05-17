# NotificationService Data And Events

## Data Ownership

NotificationService owns:

- `notifications`
- `delivery_attempts`
- `notification_templates`
- `user_channel_preferences`
- `user_group_preferences`
- `user_refs`

## Consumed Events And Commands

| Contract | Purpose |
|---|---|
| `UserCreatedIntegrationEvent` | Create user delivery reference |
| `UserUpdatedIntegrationEvent` | Refresh user delivery reference |
| `UserEmailVerificationRequestedIntegrationEvent` | Send verification email/notification |
| `NotifySellerNewOrder` | Notify seller of a new order |
| `NotifyBuyerOrderConfirmed` | Notify buyer that order was confirmed |
| `NotifyBuyerOrderCancelled` | Notify buyer that order was cancelled |

## Published Saga Events

| Event | Purpose |
|---|---|
| `SellerNewOrderNotified` | Continue fulfillment saga after seller notification |
| `BuyerNotified` | Continue/complete saga after buyer notification |

## Delivery Rules

- Every notification should have an idempotency key.
- Delivery attempts should be recorded for observability.
- User preferences apply per channel and event group.
- In-app notifications may be pushed through SignalR and persisted.
- Email notifications use templates and provider integration.
