# HiveSpace Service Inventory

## Purpose

This is the service index for feature planning. Each service folder contains the service-specific source of truth for boundaries, APIs, data ownership, messaging, and planning notes.

## Backend Services

| Service | Folder | Source path | Owns |
|---|---|---|---|
| ApiGateway | `services/api-gateway/` | `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway` | External routing and reverse proxy config |
| IdentityService | `services/identity-service/` | `../hivespace.microservice/src/HiveSpace.IdentityService` | Credentials, authentication, authorization, roles, claims, account status, lockout, email verification, IdentityServer |
| UserService | `services/user-service/` | `../hivespace.microservice/src/HiveSpace.UserService` | Profiles, settings, addresses, stores |
| CatalogService | `services/catalog-service/` | `../hivespace.microservice/src/HiveSpace.CatalogService` | Products, SKUs, categories, attributes, catalog projections |
| MediaService | `services/media-service/` | `../hivespace.microservice/src/HiveSpace.MediaService` | Media assets, upload URLs, processing state |
| OrderService | `services/order-service/` | `../hivespace.microservice/src/HiveSpace.OrderService` | Cart, checkout, orders, coupons, sagas |
| PaymentService | `services/payment-service/` | `../hivespace.microservice/src/HiveSpace.PaymentService` | Payments, gateways, wallets, transactions |
| NotificationService | `services/notification-service/` | `../hivespace.microservice/src/HiveSpace.NotificationService` | Notifications, templates, preferences, delivery, realtime hub |

## Frontend Applications

Frontend apps are not backend services, but they are first-class clients of the service APIs.

| App | Source path | Audience |
|---|---|---|
| Admin | `../hivespace.web/apps/admin` | Platform administrators |
| Seller | `../hivespace.web/apps/seller` | Sellers and merchants |
| Buyer | `../hivespace.web/apps/buyer` | Storefront buyers |
| Shared package | `../hivespace.web/packages/shared` | Runtime package consumed by all apps |

## Boundary Rules

- New backend behavior must be assigned to exactly one owning service.
- A service may keep local projections of another service's facts only through integration events.
- No service may read another service database directly.
- Browser-facing APIs go through ApiGateway.
- Async cross-service coordination goes through MassTransit contracts in `HiveSpace.Infrastructure.Messaging.Shared`.
- Frontend apps must check `@hivespace/shared` before creating app-local reusable UI, composables, stores, services, or types.

## When Planning A Feature

1. Identify the owning service from the table above.
2. Read that service folder.
3. Check `shared/api-catalog.md` for existing endpoints.
4. Check `shared/event-catalog.md` for existing events and saga messages.
5. Inspect the implementation source in the backend/frontend repo before editing code.
