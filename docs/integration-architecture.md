# Integration Architecture

## Overview

HiveSpace uses a hybrid communication model:

- **Synchronous**: All external (frontend → backend) and most cross-service queries route through the YARP API Gateway via HTTPS/REST.
- **Asynchronous**: Cross-service commands and events use MassTransit over RabbitMQ. Long-running workflows use MassTransit sagas.
- **Real-time push**: NotificationService exposes a SignalR hub that buyer/seller frontends connect to for real-time alerts.

---

## Integration Points

### 1. Frontend → API Gateway (Synchronous REST)

| From | To | Type | Details |
|------|----|------|---------|
| @hivespace/admin | YARP ApiGateway | HTTPS REST | All admin API calls via axios singleton (`src/services/api.ts`). Base URL: `VITE_GATEWAY_BASE_URL` |
| @hivespace/seller | YARP ApiGateway | HTTPS REST | Same pattern. Port 5174 in dev. |
| @hivespace/buyer | YARP ApiGateway | HTTPS REST | Same pattern + SignalR connection. Port 5175 in dev. |

All requests carry a Bearer JWT access token (injected by axios interceptor from OIDC token store).

### 2. Frontend → IdentityServer (OIDC Auth)

| From | To | Type | Details |
|------|----|------|---------|
| All frontend apps | UserService (IdentityServer) | OIDC / browser redirect | Login: redirect to IdentityServer authorize endpoint |
| All frontend apps | UserService (IdentityServer) | HTTPS token endpoint | Token refresh: refresh_token grant (no iframe/silent callback) |
| All frontend apps | UserService (IdentityServer) | HTTPS end_session | Logout: post_logout_redirect_uri |

oidc-client-ts manages PKCE flow. Callback routes: `{app}/callback/login` (signin), `{app}/callback/logout` (signout).

### 3. API Gateway → Backend Services (Proxy)

| Route Pattern | Proxied To | Notes |
|---------------|-----------|-------|
| `/api/users/*` | UserService | JWT validated by API Management policy |
| `/api/catalog/*` | CatalogService | — |
| `/api/orders/*` | OrderService | — |
| `/api/payments/*` | PaymentService | — |
| `/api/notifications/*` | NotificationService | — |
| `/api/media/*` | MediaService | File upload, YARP proxies multipart |

YARP reads backend service URLs from configuration (populated by CI with Azure Container App / App Service FQDNs in production; `appsettings.Development.json` for local).

### 4. Service → Service (Async Events via RabbitMQ)

MassTransit routes events/commands between services via RabbitMQ (port 5672). Key flows:

#### Checkout Saga (OrderService)

```
Frontend (buyer)
  → POST /api/orders/checkout
  → OrderService (CheckoutSaga state machine)
    → Publishes: ReserveInventoryCommand → CatalogService
    ← InventoryReservedEvent or InventoryReservationFailedEvent
    → Publishes: MarkOrderAsCODCommand (COD path)
    → Publishes: ClearCartCommand → UserService/CartService
    ← CartClearedEvent
    → Spawns: FulfillmentSaga (per seller, 3-day window)
      → Publishes: NotifySellerEvent → NotificationService
      ← SellerConfirmedEvent or timeout
```

#### Notification Flow

```
Any service
  → Publishes: NotifyCustomerEvent / NotifySellerEvent
  → NotificationService (consumer)
    → Sends in-app notification (database + SignalR push)
    → Sends email (Resend + FluentEmail)
```

### 5. Frontend → NotificationService (Real-time SignalR)

| From | To | Type | Details |
|------|----|------|---------|
| @hivespace/buyer | NotificationService | WebSocket (SignalR) | Connects to `/hubs/notifications` hub for real-time order/notification updates |
| @hivespace/seller | NotificationService | WebSocket (SignalR) | Same hub, filtered by seller identity |

Connection established after OIDC login; Bearer token sent as query param or header.

### 6. MediaService → Azure Blob Storage

| From | To | Type | Details |
|------|----|------|---------|
| Frontend apps | MediaService | HTTPS multipart | File upload via API Gateway |
| MediaService | Azure Blob Storage | Azure SDK | Stores processed images/files; Azurite emulator locally |
| MediaService | Azure Functions runtime | HTTP trigger | MediaService API runs as Azure Functions Worker (port on Azure) |

---

## Authentication Data Flow

```
1. User clicks "Login" in browser
2. Frontend (oidc-client-ts) → redirects to UserService IdentityServer /authorize
3. UserService: authenticates user, issues id_token + access_token + refresh_token
4. Frontend: stores tokens (session storage / memory), sets up refresh schedule
5. axios interceptor: attaches Authorization: Bearer {access_token} to every request
6. API Gateway: validates JWT signature (IdentityServer public keys via JWKS endpoint)
7. Backend services: validate JWT via AddJwtBearer (same JWKS keys)
8. On 401: oidc-client-ts silently requests new access_token using refresh_token
```

---

## Data Flow Diagram (High Level)

```
Browser
  ├─ OIDC login ─────────────────→ UserService (IdentityServer)
  ├─ axios + JWT ─────────────→ YARP ApiGateway
  │                                 ├─→ UserService REST
  │                                 ├─→ CatalogService REST
  │                                 ├─→ OrderService REST
  │                                 ├─→ PaymentService REST
  │                                 ├─→ NotificationService REST
  │                                 └─→ MediaService (Azure Function)
  └─ SignalR ─────────────────→ NotificationService (Hub)

OrderService ──(RabbitMQ)──→ CatalogService (inventory reservation)
OrderService ──(RabbitMQ)──→ UserService (cart clearing)
OrderService ──(RabbitMQ)──→ NotificationService (notify seller/customer)
CatalogService ──(RabbitMQ)──→ OrderService (reservation result)
NotificationService ──(SignalR push)──→ Browser
MediaService ──(Azure SDK)──→ Azurite / Azure Blob Storage
```

---

## Shared Contracts (Messaging)

Integration events are defined in `hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/`. All services reference this library for event/command contract types to avoid coupling to each other's domain models.

---

## Cross-Origin Configuration

- **Local dev**: Frontend apps run on `localhost:517{3,4,5}`, API Gateway on `localhost:5000` / `https://localhost:7001`. CORS must be configured in each backend service to allow the frontend origins.
- **Production**: Frontend served from Azure Static Web Apps; API Gateway on custom domain via Azure API Management.

---

_Last generated: 2026-05-16_
