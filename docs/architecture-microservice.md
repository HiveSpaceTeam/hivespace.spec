# HiveSpace Microservices Architecture

## Executive Summary

HiveSpace is a microservices-based e-commerce platform built on .NET 8.0. It uses Clean Architecture / Domain-Driven Design (DDD) as the standard pattern for most services, with a lightweight 2-project structure for simpler services. Services communicate asynchronously via MassTransit (RabbitMQ/Kafka), with an API Gateway (YARP) handling all client-facing routing. Each service owns its database; cross-service data is shared via local projections rather than inter-service REST calls.

---

## Technology Stack

| Package | Version | Purpose |
|---|---|---|
| .NET | 8.0 | Runtime platform |
| Carter | 8.2.1 | Minimal API endpoint modules |
| MediatR | 13.0.0 | CQRS dispatcher (commands, queries, notifications) |
| EF Core | 9.0.1 | ORM and schema management |
| MassTransit | 8.5.7 | Message bus abstraction, saga engine, outbox |
| Duende IdentityServer | 7.2.4 | OAuth2 / OIDC identity provider (UserService only) |
| YARP | 2.3.0 | Reverse proxy / API Gateway |
| FluentValidation | 12.0.0 | Request validation pipeline |
| Serilog | 3.1.1 | Structured logging |
| Hangfire | 1.8.9 | Background job scheduling (NotificationService) |
| StackExchange.Redis | 2.12.8 | Distributed cache / throttle store |
| Scriban | 5.10.0 | Notification template rendering |
| Resend | 0.4.0 | Email delivery (NotificationService) |
| Azure.Storage.Blobs | 12.26.0 | Blob storage (MediaService) |
| SixLabors.ImageSharp | 3.1.12 | Image processing and thumbnail generation |

All package versions are declared in `Directory.Packages.props` at the solution root. Individual `.csproj` files must never include a `Version=` attribute on `PackageReference` elements.

---

## Architecture Pattern

### Standard Pattern: Clean Architecture / DDD

Applied to: UserService, CatalogService, OrderService, PaymentService.

Layers (inner to outer, each depending only inward):

```
Domain
  └── Application
        └── Infrastructure
              └── Api
```

- **Domain**: Aggregates, entities, value objects, domain events. Entities use private setters, a protected parameterless constructor, and a static `Create(...)` factory method.
- **Application**: CQRS handlers (commands/queries via MediatR), DTOs, interfaces for infrastructure ports.
- **Infrastructure**: EF Core `DbContext`, repository implementations, messaging consumers, external client integrations.
- **Api**: Carter endpoint modules (Minimal API) or MVC controllers (UserService only), middleware, DI registration.

### Lightweight Pattern: 2-Project (Core + Api)

Applied to: MediaService, NotificationService.

No DDD aggregate model. No CQRS pipeline. Business logic lives directly in services within the `Core` project. The `Api` project hosts controllers/endpoints and DI wiring. This mirrors the `MediaService` codebase structure.

---

## Service Overview

| Service | Purpose | Port | Key Dependencies |
|---|---|---|---|
| ApiGateway | YARP reverse proxy; routes all client traffic | http://localhost:5000 | YARP 2.3.0 |
| UserService | Identity, auth (OIDC), user/store/admin management | https://localhost:5001 | Duende IdentityServer, SQL Server |
| CatalogService | Product catalog, categories, attributes | http://localhost:5002 | SQL Server, RabbitMQ, Kafka |
| MediaService | File upload, presign URL, image processing | http://localhost:5003 | Azure Blob Storage, SQL Server |
| OrderService | Cart, checkout sagas, orders, coupons | http://localhost:5004 | SQL Server, RabbitMQ, Kafka |
| PaymentService | Payment processing, wallet management | http://localhost:5005 | SQL Server, RabbitMQ, Kafka |
| NotificationService | In-app, email notifications; SignalR hub | http://localhost:5006 | SQL Server, RabbitMQ, Redis, Hangfire, SignalR |

---

## Per-Service Deep-Dive

### ApiGateway

Built on YARP (Yet Another Reverse Proxy). Receives all HTTP traffic from clients and proxies to the appropriate downstream service based on path prefix. No business logic. Clusters are configured for each downstream service. See `api-contracts-microservice.md` for the full route table.

---

### UserService

**Pattern**: MVC controllers + service layer (not CQRS). Exception to the standard pattern because Duende IdentityServer integrates deeply with ASP.NET Identity and MVC.

**Responsibilities**:
- OAuth2 / OIDC identity provider via Duende IdentityServer 7.2.4
- ASP.NET Identity user management (registration, email verification, status management)
- Store creation (transitions a user to seller role)
- Admin management (create admins, list/manage users)
- User address CRUD

**Database**: SQL Server. Schema includes ASP.NET Identity tables, Duende IdentityServer EF tables, and custom tables.

**Custom AppUser fields**: `DateOfBirth`, `PhoneNumber`, `AvatarUrl`, `Gender`, `Theme`, `Status`.

**Key flows**:
- Email verification: POST send-verification → token stored → POST verify → user email confirmed
- Store creation: POST /stores → creates store record, assigns seller role → publishes `StoreCreated` event consumed by CatalogService

---

### CatalogService

**Pattern**: Clean Architecture / DDD.

**Responsibilities**:
- Product and SKU management for sellers
- Category tree management
- Product attribute definitions and values
- Storefront-facing read models (anonymous product summaries, product detail)
- Maintains local `store_refs` projection synced from UserService events

**Database**: SQL Server. Key tables: `products`, `product_variants`, `product_variant_options`, `sku_variants`, `product_attributes`, `product_categories`, `categories`, `category_attributes`, `attribute_definitions`, `attribute_values`, `store_refs`.

**Messaging**: Publishes product/SKU events consumed by OrderService to keep `product_refs` and `sku_refs` projections current.

---

### OrderService

**Pattern**: Clean Architecture / DDD. Hosts two MassTransit sagas.

**Responsibilities**:
- Cart management (items, coupons, quantity updates)
- Checkout orchestration via `CheckoutSaga`
- Seller confirmation via `FulfillmentSaga`
- Order lifecycle (buyer view, seller view)
- Coupon management (CRUD, usage tracking)
- Local projections: `product_refs`, `sku_refs`, `store_refs`

**Database**: SQL Server + MassTransit outbox/inbox/saga state tables.

#### CheckoutSaga

`MassTransitStateMachine<CheckoutSagaState>` persisted via EF Core.

**State machine transitions**:

```
Initial
  → OrderCreation.Pending        (on InitiateCheckout command)
  → InventoryReservation.Pending (on OrderCreated success)
  → CODMarking.Pending           (COD path: on InventoryReserved success)
    → CartClearing.Pending       (on CODMarked success)
      → FulfillmentSaga started  (on CartCleared success)
        → Completed
  → PaymentInitiation.Pending    (online payment path: on InventoryReserved success)
    → AwaitingPayment            (on PaymentInitiated success; waits for payment callback)
      → PaymentMarking.Pending   (on PaymentConfirmed event)
        → CouponUsageCommit.Pending
          → CartClearing.Pending
            → FulfillmentSaga started
              → Completed
  → Compensating                 (on any step failure)
    → Failed                     (compensation complete: cancel orders + release inventory)
```

Uses `Request<TState, TCommand, TSuccess, TFailure>` pattern for each step with timeout and failure handling.

#### FulfillmentSaga

`MassTransitStateMachine<FulfillmentSagaState>` persisted via EF Core. One saga instance per seller sub-order.

**State machine transitions**:

```
Initial
  → NotifyingSeller              (send seller notification)
  → WaitingForSellerConfirmation (3-day timeout window)
      ↓ Timeout → auto-reject path → Compensating → Failed
      ↓ Seller confirms →
  → ConfirmingInventory          (final inventory confirmation with warehouse)
  → NotifyingBuyer               (send buyer notification)
  → Completed
  → Compensating / Failed        (on any failure)
```

---

### PaymentService

**Pattern**: Clean Architecture / DDD.

**Responsibilities**:
- Payment aggregate lifecycle (create, process gateway response, mark paid/failed)
- VNPay gateway integration (redirect + IPN webhook)
- Wallet aggregate (balance, escrow balance, reward points)
- Transaction history per wallet
- Idempotency on all payment operations

**Database**: SQL Server. Key tables: `payments`, `wallets`, `transactions`. See `data-models-microservice.md` for full schema.

**Money handling**: All monetary values stored as `long` (smallest currency unit, e.g. VND). `Currency` is an enum. The `Money` value object provides `ExceedsCODLimit()` (threshold: 2,000,000 VND) and `CalculateServiceFee()` (rate: 9.9%).

**Messaging**: Consumes checkout saga events to create/update payment records. Publishes payment result events back to CheckoutSaga.

---

### MediaService

**Pattern**: Lightweight 2-project (Core + Api).

**Responsibilities**:
- Generate presigned Azure Blob Storage upload URLs
- Confirm upload completion (entity association)
- Image processing and thumbnail generation via ImageSharp
- Track media asset lifecycle (Pending → Uploaded → Processed → Failed)

**Database**: SQL Server. Single table: `media_assets`.

**Upload flow**:
1. Client calls POST /media/presign-url → receives `presignedUrl` + `fileId`
2. Client uploads directly to Azure Blob Storage using presigned URL
3. Client calls POST /media/{fileId}/confirm with `entityId` → service marks asset as Uploaded, triggers processing

---

### NotificationService

**Pattern**: Lightweight 2-project (Core + Api).

**Responsibilities**:
- Replaces stub `NotifyCustomerConsumer` / `NotifySellerConsumer` from checkout saga
- Delivers in-app notifications (persisted to DB, pushed via SignalR)
- Delivers email notifications (rendered via Scriban templates, sent via Resend)
- User channel and event-group preferences
- Throttling and retry via Hangfire + Redis
- Idempotency via `IdempotencyKey` on every notification record

**Channels (Phase 1)**: InApp, Email. SMS and Push planned for Phase 2.

**Database**: SQL Server + Redis. Key tables: `notifications`, `delivery_attempts`, `notification_templates`, `user_channel_preferences`, `user_group_preferences`, `user_refs`.

**Real-time**: SignalR hub at `/hubs/notifications`. JWT authentication required. Clients subscribe on connect; server pushes `ReceiveNotification` events.

**Background jobs**: Hangfire manages retry scheduling for failed/throttled notification delivery.

---

## Shared Libraries

Located under `hivespace.microservice/libs/`:

| Library | Purpose |
|---|---|
| `HiveSpace.Core` | Core utilities, JWT helpers, shared configuration primitives |
| `HiveSpace.Domain.Shared` | Base classes: `Entity<TId>`, `AggregateRoot<TId>`, `ValueObject`, `DomainEvent`. Known: 6 non-blocking nullability warnings. |
| `HiveSpace.Application.Shared` | CQRS base types: `ICommand`, `IQuery`, `ICommandHandler`, `IQueryHandler`. MediatR pipeline behaviors (validation, logging). |
| `HiveSpace.Infrastructure.Messaging` | RabbitMQ and Kafka configuration helpers, MassTransit bus setup |
| `HiveSpace.Infrastructure.Messaging.Shared` | Shared message contracts (events, commands) crossing service boundaries |
| `HiveSpace.Infrastructure.Authorization` | Auth policy definitions: `RequireSystemAdmin`, `RequireAdmin`, `RequireSeller`, `RequireUser`, `RequireBuyerUser`, `RequireAdminOrUser` |
| `HiveSpace.Infrastructure.Persistence` | Generic EF Core patterns: generic repository, specification pattern, unit of work base |

---

## Cross-Cutting Patterns

### Transactional Outbox

All services that publish messages use the MassTransit EF Core Outbox. Published messages are written to `outbox_message` in the same DB transaction as the domain write. MassTransit delivers them asynchronously. Corresponding `inbox_state` prevents duplicate consumer processing. Tables present in all publishing services: `inbox_state`, `outbox_message`, `outbox_state`.

### Idempotency

- **PaymentService**: `IdempotencyKey` (string 200, unique index) on `payments` table. All payment creation requests must supply an idempotency key.
- **NotificationService**: `IdempotencyKey` (string, unique index) on `notifications` table. Prevents duplicate delivery for the same event.

### Authorization Policies

Defined in `HiveSpace.Infrastructure.Authorization`. Applied as named policy strings in Carter endpoint modules or controller attributes.

| Policy | Meaning |
|---|---|
| `RequireSystemAdmin` | System-level admin only |
| `RequireAdmin` | Platform admin |
| `RequireSeller` | User with seller role |
| `RequireUser` | Any authenticated user |
| `RequireBuyerUser` | Authenticated user in buyer context |
| `RequireAdminOrUser` | Admin or authenticated user |
| `Authorize` (bare) | Any authenticated principal |

### ID Generation

Two strategies selected at startup via configuration:
- `GuidV7Generator`: Generates UUIDv7 (time-ordered GUIDs). Default for aggregate roots with `Guid` PK.
- `SnowflakeIdGenerator`: Twitter Snowflake-style monotonic long IDs. Used where `long` PK is required (e.g. `ProductId`, `SkuId` in catalog and order item references).

### Money Value Object

`Money` is a value object with two fields: `Amount` (stored as `long`, smallest currency unit, e.g. VND) and `Currency` (enum). EF Core maps these as owned type columns (`Amount`, `Currency`) per usage site.

Key methods:
- `ExceedsCODLimit()`: Returns `true` if `Amount > 2_000_000`. Used by CheckoutSaga to route COD vs. online payment.
- `CalculateServiceFee()`: Returns `Amount * 0.099` as a new `Money`. Used for platform fee calculations.

### Cross-Service Data: Local Projections

Services do not make synchronous REST calls to each other for reference data. Each service maintains local projection tables populated by consuming domain events:

| Projection Table | Maintained By | Source Service |
|---|---|---|
| `product_refs` | OrderService | CatalogService product events |
| `sku_refs` | OrderService | CatalogService SKU events |
| `store_refs` | OrderService, CatalogService | UserService store events |
| `user_refs` | NotificationService | UserService user events |

### Table and Column Naming

All EF Core table names use `snake_case` configured via `builder.ToTable("snake_case_name")` in `OnModelCreating`.

### Entity Pattern

```csharp
public class MyEntity : Entity<Guid>
{
    public string Name { get; private set; }

    protected MyEntity() { } // Required by EF Core

    public static MyEntity Create(string name)
    {
        return new MyEntity { Id = GuidV7Generator.New(), Name = name };
    }
}
```

Private setters, protected parameterless constructor, and static `Create` factory are mandatory for all domain entities.
