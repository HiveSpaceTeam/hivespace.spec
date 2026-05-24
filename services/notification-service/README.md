# NotificationService

## Responsibility

NotificationService owns notification persistence, delivery preferences, in-app delivery, email delivery, retries, and realtime notification push.

Source path:

```text
../hivespace.microservice/src/HiveSpace.NotificationService
```

## Owns

- Notification records.
- Delivery attempts.
- Notification templates.
- User channel and event-group preferences.
- Realtime SignalR hub.
- Email delivery through configured providers.
- User reference projection for delivery personalization.

## Must Not Own

- Order, payment, catalog, or identity business decisions.
- Whether a buyer/seller action is valid.
- Product, order, payment, or user source-of-truth data.

## Architecture

NotificationService is a lighter service. New work should use the repo's CQRS/Minimal API direction and keep delivery orchestration separate from business-domain decisions.

## Runtime

| Item | Value |
|---|---|
| Local HTTP | `http://localhost:5006` |
| Gateway prefixes | `/api/v1/notifications`, `/api/v1/notification-preferences`, `/hubs/notifications` |
| Database | SQL Server, notification records/templates/preferences |
| Cache/jobs | Redis and Hangfire-style retry/throttle support where configured |
| Email | Resend/FluentEmail |

## Planning Notes

- Other services publish notification intent; NotificationService decides delivery mechanics.
- Notification preferences affect delivery, not whether source-domain events occurred.
- SignalR clients receive `ReceiveNotification` events through the shared frontend realtime composable.
- Fulfillment continuation events use standardized `*IntegrationEvent` names while remaining MassTransit consume-context orchestration messages per [ADR-0002](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md).

## Detail

- Domain model: `domain.md`
- Public API: `api.md`
- Data and events: `data-and-events.md`
- Workflows: `workflows.md`
