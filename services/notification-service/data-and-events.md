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
| `UserEmailVerificationRequestedIntegrationEvent` | Send verification email/notification for IdentityService-owned verification state |
| `NotifySellerNewOrder` | Notify seller of a new order |
| `NotifyBuyerOrderConfirmed` | Notify buyer that order was confirmed |
| `NotifyBuyerOrderCancelled` | Notify buyer that order was cancelled |

## Published Saga Events

| Event | Purpose |
|---|---|
| `SellerNewOrderNotifiedIntegrationEvent` | Continue fulfillment saga after seller notification |
| `BuyerNotifiedIntegrationEvent` | Continue/complete saga after buyer notification |

## Delivery Rules

- Every notification should have an idempotency key.
- Delivery attempts should be recorded for observability.
- User preferences apply per channel and event group.
- In-app notifications may be pushed through SignalR and persisted.
- Email notifications use templates and provider integration.
- IdentityService owns email verification state; NotificationService only owns delivery and delivery observability.

## Publisher Policy

- Fulfillment continuation publishes remain MassTransit consume-context orchestration messages, not application/background publisher calls.
