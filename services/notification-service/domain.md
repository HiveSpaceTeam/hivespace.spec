# NotificationService Domain Model

## Purpose

NotificationService owns user-visible notification records, templates, delivery attempts, channel/event-group preferences, delivery references, and realtime push state.

Implementation source:

```text
../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Core/DomainModels
```

## Core Model

| Model | Type | Meaning |
|---|---|---|
| `Notification` | Aggregate-like entity | User notification with channel, event type, idempotency key, payload, status, attempt count, read/sent timestamps, and delivery attempts |
| `DeliveryAttempt` | Child entity | One provider send attempt with attempt number, success flag, provider response, and error message |
| `NotificationTemplate` | Entity | Template per event type/channel/locale with subject and Scriban body template |
| `UserChannelPreference` | Entity | Per-user enable/disable preference for a delivery channel |
| `UserGroupPreference` | Entity | Per-user enable/disable preference for an event group within a channel |
| `UserRef` | Projection model | Local copy of user/store delivery data from UserService events |
| `NotificationEventGroup` | Domain helper | Maps event types and roles to preference groups |

## Business Rules

- Notifications are created with a stable idempotency key, expected to prevent duplicate delivery for the same event/source/channel.
- Notification payload is stored as JSON for audit and replay.
- New notifications start as `Pending`.
- `MarkSent()` records sent time and moves status to `Sent`.
- `MarkFailed(...)`, `MarkThrottled()`, and `MarkDead(...)` preserve failed delivery outcomes for retry/dead-letter visibility.
- `MarkRead()` sets status to `Read` and records read time; this applies to user-facing in-app notifications.
- Attempt count increments independently from persisted `DeliveryAttempt` rows.
- Delivery attempts record provider response or error message for observability.
- Templates are keyed by event type, channel, and locale; updates replace subject/body and refresh `UpdatedAt`.
- Channel preferences apply at the whole-channel level.
- Group preferences apply within a channel by event group.
- Event groups differ by role: sellers receive seller/inventory/order/payment groups; buyers receive order/payment/promotion/survey groups; admin groups are currently empty.

## Lifecycle

| Lifecycle | States / transitions |
|---|---|
| Notification status | `Pending`, `Sent`, `Failed`, `Throttled`, `Dead`, `Read` |
| Delivery | Create notification, render template, send through channel provider, record attempt, update status, optionally push SignalR for in-app |
| Preferences | Upsert channel/group preference for the current user and channel |
| User reference | Create/update user delivery reference from user events and store data from store events |

## Cross-Service Facts

- NotificationService consumes user and store events to maintain `UserRef`.
- NotificationService consumes notification intent commands such as `NotifySellerNewOrder`, `NotifyBuyerOrderConfirmed`, and `NotifyBuyerOrderCancelled`.
- NotificationService publishes saga continuation events such as `SellerNewOrderNotifiedIntegrationEvent` and `BuyerNotifiedIntegrationEvent`.
- Source services decide that a business event happened; NotificationService decides delivery mechanics, templates, preference filtering, attempts, and realtime push.
- NotificationService does not own order, payment, catalog, or identity source-of-truth data.
