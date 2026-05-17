# ApiGateway API Routing

## Route Ownership

| Path | Downstream service |
|---|---|
| `/identity/**` | UserService |
| `/api/v1/users/**` | UserService |
| `/api/v1/accounts/**` | UserService |
| `/api/v1/admins/**` | UserService |
| `/api/v1/stores/**` | UserService |
| `/api/v1/categories/**` | CatalogService |
| `/api/v1/products/**` | CatalogService |
| `/api/v1/media/**` | MediaService |
| `/api/v1/carts/**` | OrderService |
| `/api/v1/coupons/**` | OrderService |
| `/api/v1/orders/**` | OrderService |
| `/api/v1/payments/**` | PaymentService |
| `/api/v1/wallets/**` | PaymentService |
| `/api/v1/notifications/**` | NotificationService |
| `/api/v1/notification-preferences/**` | NotificationService |
| `/hubs/notifications/**` | NotificationService |

## Change Rules

- Add a new route prefix only when no existing prefix clearly owns the API.
- Prefer versioned `/api/v1/...` REST routes.
- Preserve `/hubs/notifications` for SignalR.
- Do not route browser traffic directly to internal service implementation paths.
