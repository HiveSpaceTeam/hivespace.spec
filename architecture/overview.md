# HiveSpace Architecture Overview

## Purpose

This is the architecture entry point for `hivespace.spec`. It is intentionally self-contained so generated documentation can be deleted later without losing required planning context.

Use this file with:

- `shared/api-catalog.md`
- `shared/event-catalog.md`
- `shared/coding-conventions.md`
- `shared/glossary.md`
- `services/_inventory.md`
- `services/<service-name>/`

## Repository Map

| Repository | Path | Role |
|---|---|---|
| Spec | `hivespace.spec` | Product specs, architecture docs, service reference, catalogs, constitution |
| Backend | `../hivespace.microservice` | .NET 8 microservice backend |
| Frontend | `../hivespace.web` | Vue 3 pnpm/Turbo frontend monorepo |
| Config | `../hivespace.config` | Docker Compose and infrastructure configuration |

The spec repo is the planning source of truth. Backend and frontend repos are the implementation source of truth.

## Platform Shape

HiveSpace is a multi-service e-commerce platform with three browser apps and seven backend services.

```text
Admin app      Seller app      Buyer app
    |              |              |
    +--------------+--------------+
                   |
          REST + JWT through YARP
                   |
              ApiGateway
                   |
    +--------------+--------------+----------------+----------------+
    |              |              |                |                |
UserService  CatalogService  OrderService  PaymentService  NotificationService
                                                    |                |
                                               MediaService      SignalR hub

Backend services communicate through MassTransit/RabbitMQ for async workflows.
Each business service owns its own SQL Server database.
```

## Frontend System

`../hivespace.web` is a pnpm workspace managed by Turbo.

| Workspace | Package | Audience | Dev port |
|---|---|---|---:|
| `apps/admin` | `@hivespace/admin` | Platform administrators | 5173 |
| `apps/seller` | `@hivespace/seller` | Sellers and merchants | 5174 |
| `apps/buyer` | `@hivespace/buyer` | Buyers and storefront users | 5175 |
| `packages/shared` | `@hivespace/shared` | Shared runtime package for all apps | - |
| `packages/demo` | `@hivespace/demo` | Development demo pages | - |
| `packages/eslint-config` | `@hivespace/eslint-config` | Shared lint rules | - |
| `packages/tsconfig` | `@hivespace/tsconfig` | Shared TypeScript configs | - |
| `packages/vite-config` | `@hivespace/vite-config` | Shared Vite aliases/helpers | - |

Frontend rules:

- Apps may import shared runtime code from `@hivespace/shared`.
- Apps must not import directly from other apps.
- Domain data flows through service modules and Pinia stores.
- User-facing strings use i18n resources.
- Tailwind CSS v4 is CSS-first; do not add `tailwind.config.js`.

Shared frontend capabilities live under `../hivespace.web/packages/shared/src`:

- UI components: common controls, layouts, modals, tables, charts.
- Composables: auth, modal, confirmation, debounce, cooldown, formatting, validation, notification hub.
- Feature factories: notifications, media upload, user profile, user settings, auth refresh.
- Shared services/types/stores/i18n/icons/assets.

## Backend System

`../hivespace.microservice` is a .NET 8 solution with ASP.NET Core services and shared libraries.

| Service | Source path | Responsibility | Port |
|---|---|---|---:|
| ApiGateway | `src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway` | Reverse proxy and external route entry point | 5000 / 7001 |
| UserService | `src/HiveSpace.UserService` | Identity, OIDC, users, roles, addresses, stores | 5001 |
| CatalogService | `src/HiveSpace.CatalogService` | Products, SKUs, categories, attributes, inventory-facing data | 5002 |
| MediaService | `src/HiveSpace.MediaService` | Upload URLs, media records, image processing | 5003 |
| OrderService | `src/HiveSpace.OrderService` | Cart, checkout, orders, coupons, fulfillment | 5004 |
| PaymentService | `src/HiveSpace.PaymentService` | Payments, gateways, wallets, transactions | 5005 |
| NotificationService | `src/HiveSpace.NotificationService` | In-app/email notification delivery, preferences, SignalR | 5006 |

Shared backend libraries:

| Library | Purpose |
|---|---|
| `HiveSpace.Core` | Core helpers, filters, pagination/config utilities |
| `HiveSpace.Domain.Shared` | Aggregate, entity, value object, domain event, domain exception primitives |
| `HiveSpace.Application.Shared` | CQRS request/handler abstractions and pipeline behavior |
| `HiveSpace.Infrastructure.Authorization` | Authorization attributes, policies, role requirements |
| `HiveSpace.Infrastructure.Persistence` | EF Core persistence, repositories, transactions, idempotency, interceptors |
| `HiveSpace.Infrastructure.Messaging` | MassTransit/RabbitMQ/Kafka setup and outbox integration |
| `HiveSpace.Infrastructure.Messaging.Shared` | Cross-service integration events and commands |

## Backend Architecture Pattern

New backend feature work uses CQRS plus Minimal API endpoints, except where maintaining existing UserService identity code requires legacy controllers.

Standard layered services:

```text
Domain -> Application -> Infrastructure -> Api
```

- Domain owns aggregates, invariants, value objects, domain events, and domain exceptions.
- Application owns commands, queries, validators, handlers, DTOs, and ports.
- Infrastructure owns EF Core, repository implementations, messaging consumers, gateway clients, storage/email integrations.
- Api owns endpoints, middleware, authorization wiring, health checks, and dependency registration.

Lite services still use CQRS and Minimal API entrypoints where implemented, but with fewer projects and less DDD ceremony.

## Integration Model

| Flow | Mechanism | Notes |
|---|---|---|
| Browser API calls | REST through YARP ApiGateway | Gateway base URL defaults to `https://localhost:7001`; APIs are versioned under `/api/v1` |
| Browser auth | OIDC PKCE against UserService | `oidc-client-ts`; refresh-token renewal, not iframe renewal |
| Backend workflow commands/events | MassTransit over RabbitMQ | Contracts live in `HiveSpace.Infrastructure.Messaging.Shared` |
| Long-running checkout/fulfillment | MassTransit sagas | OrderService coordinates, other services own their steps |
| Realtime updates | SignalR Notification hub | `/hubs/notifications` |
| Media storage | Azure Blob Storage or Azurite | MediaService issues upload URLs and confirms records |
| Email delivery | Resend and FluentEmail | NotificationService owns templates, attempts, preferences |

## Core Runtime Flows

### Authentication

```text
Browser app
  -> UserService IdentityServer authorize endpoint
  -> OIDC login + callback
  -> access token + refresh token
  -> ApiService attaches Authorization: Bearer <token>
  -> ApiGateway and downstream service validate JWT
```

### Checkout

```text
Buyer app
  -> POST /api/v1/orders/checkout/preview
  -> POST /api/v1/orders/checkout
  -> OrderService CheckoutSaga
  -> CatalogService reserves inventory
  -> PaymentService initiates or records payment when needed
  -> OrderService clears cart and starts fulfillment
  -> NotificationService informs seller and buyer
```

OrderService coordinates the saga. CatalogService owns product/inventory facts. PaymentService owns payment state. NotificationService owns notification persistence and delivery.

### Media Upload

```text
Frontend
  -> POST /api/v1/media/presign-url
  -> upload bytes directly to Blob/Azurite
  -> POST /api/v1/media/{fileId}/confirm
  -> MediaService records ownership and processing state
```

### Notifications

```text
Any service
  -> Notify or domain-specific integration event
  -> NotificationService consumer
  -> notification row + delivery attempts
  -> SignalR push and/or email
```

## Data Ownership

Each service owns its own database and migrations. No feature may introduce shared DbContexts or direct cross-service database reads.

| Service | Owns | Must not own |
|---|---|---|
| UserService | Accounts, OIDC clients/grants, roles, user settings, addresses, stores | Orders, catalog, payment, notification delivery |
| CatalogService | Categories, products, SKUs, product attributes, catalog projections | Checkout, payment, user identity |
| OrderService | Cart, checkout, orders, coupons, saga state, order projections | Payment gateway processing, catalog management, identity |
| PaymentService | Payment aggregate, gateway transactions, wallets, wallet transactions | Order lifecycle, catalog, notifications |
| NotificationService | Notifications, templates, preferences, delivery attempts, realtime hub | Business decisions from other domains |
| MediaService | Media asset records, upload confirmation, processing state | Product/order/user business records |
| ApiGateway | Routes and reverse proxy config | Business data |

Cross-service data is copied only through explicit contracts, such as `store_refs`, `product_refs`, `sku_refs`, and `user_refs`.

## Local Runtime

Local infrastructure is expected from `../hivespace.config/docker`.

| Dependency | Port |
|---|---:|
| SQL Server | 1433 |
| RabbitMQ AMQP | 5672 |
| RabbitMQ UI | 15672 |
| Azure Service Bus emulator | 5673 |
| Redis | 6379 |
| Azurite Blob | 10000 |
| Azurite Queue | 10001 |
| Azurite Table | 10002 |

Common commands:

```bash
cd ../hivespace.config/docker
docker compose up -d

cd ../../hivespace.microservice
dotnet restore
dotnet build

cd ../hivespace.web
pnpm install
pnpm dev:admin
pnpm dev:seller
pnpm dev:buyer
```

## Planning Rules

Before planning a feature:

1. Read this overview.
2. Read `shared/api-catalog.md` and `shared/event-catalog.md`.
3. Read `services/_inventory.md`.
4. Read each affected `services/<service-name>/` document.
5. Read `shared/coding-conventions.md`.
6. Inspect implementation source in `../hivespace.web` or `../hivespace.microservice` for final truth before editing code.

Do not add an API, event, service responsibility, shared component, or workflow that duplicates an existing catalog entry.
