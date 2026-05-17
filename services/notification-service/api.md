# NotificationService API

## Notifications

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/notifications` | `Authorize` | List notifications |
| GET | `/api/v1/notifications/unread-count` | `Authorize` | Get unread notification count |
| PUT | `/api/v1/notifications/{id}/read` | `Authorize` | Mark notification read |

## Preferences

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/api/v1/notification-preferences` | `Authorize` | Get notification preferences |
| PUT | `/api/v1/notification-preferences/{channel}` | `Authorize` | Enable/disable channel |
| PUT | `/api/v1/notification-preferences/{channel}/{eventGroup}` | `Authorize` | Enable/disable event group per channel |

## SignalR

| Hub | Event | Purpose |
|---|---|---|
| `/hubs/notifications` | `ReceiveNotification` | Push a notification event to connected clients |

```ts
interface NotificationHubEvent {
  id: string
  eventType: string
  payload: string
  createdAt: string
}
```

`payload` is serialized JSON.
