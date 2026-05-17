# HiveSpace Glossary

## Purpose

Shared vocabulary for specs, plans, service docs, API contracts, and implementation discussions.

## Platform Terms

| Term | Meaning |
|---|---|
| HiveSpace | Multi-service e-commerce platform spanning backend services, frontend apps, config, and specs |
| Spec repo | `hivespace.spec`; planning and documentation source of truth |
| Backend repo | `hivespace.microservice`; .NET service implementation |
| Frontend repo | `hivespace.web`; Vue app and shared package implementation |
| ApiGateway | YARP reverse proxy that receives browser-facing REST traffic |
| Service boundary | Ownership line that defines which service may change data and make business decisions |
| Projection | Local copied reference data owned by one service but sourced from another service's event |

## Users and Roles

| Term | Meaning |
|---|---|
| Buyer | Customer using the storefront app |
| Seller | Merchant using seller app after registering a store |
| Store owner | Seller role/claim granted after store registration and token refresh |
| Admin | Platform operator using admin app |
| System admin | Highest-privilege platform administrator |
| UserRef | Minimal local projection of user data, usually for notification delivery |
| StoreRef | Minimal local projection of store data in CatalogService or OrderService |

## Commerce Terms

| Term | Meaning |
|---|---|
| Product | Sellable catalog item managed by a seller |
| SKU | Concrete purchasable product variant with price/inventory |
| Variant | Product option dimension or selected option combination |
| Category | Catalog hierarchy node used for discovery and attribute rules |
| Attribute | Product metadata definition or value tied to category/product setup |
| Cart | Buyer's selected items and coupon choices before checkout |
| Checkout | Workflow that validates cart, reserves inventory, creates orders, and handles payment |
| Order | Buyer purchase record, commonly split by seller/store where needed |
| Fulfillment | Seller confirmation and post-checkout order handling |
| Coupon | Platform or seller promotion applied to cart/order pricing |
| COD | Cash on delivery; checkout path without immediate online payment redirect |

## Payment Terms

| Term | Meaning |
|---|---|
| Payment | Payment aggregate tied to an order |
| Gateway | External payment provider such as VNPay |
| IPN | Instant payment notification/webhook from a payment gateway |
| Wallet | User balance account managed by PaymentService |
| Escrow balance | Funds held until order conditions are met |
| Transaction | Wallet ledger entry |
| Money | Value object stored as smallest currency unit plus currency |
| VND smallest unit | `long` integer amount used instead of decimals |

## Media Terms

| Term | Meaning |
|---|---|
| Media asset | Record of an uploaded file and processing lifecycle |
| Presigned URL | Upload URL issued by MediaService so the browser can upload directly to storage |
| Confirm upload | API call that marks a file uploaded and associates it with an owning entity |
| Blob storage | Azure Blob Storage in production or Azurite locally |
| Thumbnail | Processed derivative image produced after upload |

## Notification Terms

| Term | Meaning |
|---|---|
| Notification | Persisted user-visible message |
| Channel | Delivery medium such as InApp or Email |
| Event group | Preference grouping for related notification types |
| Template | Scriban/notification body definition per event/channel |
| Delivery attempt | Audit row for one send attempt |
| SignalR hub | Realtime WebSocket endpoint at `/hubs/notifications` |
| `ReceiveNotification` | Client event emitted by NotificationService hub |

## Architecture Terms

| Term | Meaning |
|---|---|
| CQRS | Command/query responsibility split used for backend feature handlers |
| Command | Request to perform a state-changing operation |
| Query | Request to read data without side effects |
| Handler | Application-layer component that processes a command or query |
| Aggregate | Domain consistency boundary |
| Entity | Domain object with identity |
| Value object | Immutable value defined by properties rather than identity |
| Domain event | In-process fact raised by domain model after a domain change |
| Integration event | Cross-service fact published through messaging |
| Saga | Long-running state machine coordinating multiple async steps |
| Transactional outbox | Pattern that atomically commits database changes and outgoing messages |
| Idempotency | Ability to process the same request/message safely more than once |

## Frontend Terms

| Term | Meaning |
|---|---|
| `@hivespace/shared` | Shared frontend runtime package consumed by all apps |
| App shell | Shared layout frame for header/sidebar/page body |
| Pinia store | Frontend state owner for a domain or shared feature |
| Service module | Frontend module that owns HTTP calls for a domain |
| Page | Route entry component |
| Component | Reusable UI unit that does not directly own domain HTTP calls |
| Composable | Reusable Composition API function |
| i18n key | Translation key used instead of hardcoded UI text |

## Local Infrastructure Terms

| Term | Meaning |
|---|---|
| RabbitMQ | Local message broker for MassTransit |
| Redis | Cache/throttle/dedup infrastructure |
| Azurite | Local emulator for Azure Storage |
| SQL Server | Primary local database engine |
| Docker Compose | Local infrastructure startup mechanism under `hivespace.config` |
