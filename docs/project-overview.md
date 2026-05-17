# HiveSpace Project Overview

## Executive Summary

HiveSpace is a multi-service e-commerce platform built as two separate repositories:

- **hivespace.microservice** — .NET 8.0 backend, comprising 7 microservices communicating via REST (YARP API Gateway) and async messaging (MassTransit/RabbitMQ). Each service owns its own SQL Server database and follows Clean Architecture + DDD.
- **hivespace.web** — Vue 3 + TypeScript frontend monorepo (pnpm + Turbo), containing 3 production SPA apps (admin, seller, buyer) and a shared UI/feature package.

All frontend apps authenticate via OIDC (Duende IdentityServer in UserService), and all API calls are routed through the YARP API Gateway.

---

## Repository Map

| Repository | Path | Purpose |
|------------|------|---------|
| hivespace.microservice | `../hivespace.microservice/` | Backend microservices (API, business logic, data) |
| hivespace.web | `../hivespace.web/` | Frontend SPAs and shared UI library |
| hivespace.planning | `.` | Architecture planning and documentation (this repo) |
| hivespace.config | `../hivespace.config/` | Infrastructure config (Docker Compose, Bicep) |

---

## Technology Summary

### Backend (hivespace.microservice)

| Category | Technology | Version |
|----------|-----------|---------|
| Runtime | .NET / ASP.NET Core | 8.0 |
| ORM | EF Core (SQL Server) | 9.0.1 |
| CQRS | MediatR | 13.0.0 |
| Messaging | MassTransit + RabbitMQ | 8.5.7 |
| Auth provider | Duende IdentityServer | 7.2.4 |
| Auth consumers | JWT Bearer | — |
| API Gateway | YARP | 2.3.0 |
| Routing | Carter modules | 8.2.1 |
| Validation | FluentValidation | 12.0.0 |
| Background jobs | Hangfire | 1.8.9 |
| Logging | Serilog | 3.1.1 |
| Cache | StackExchange.Redis | 2.12.8 |
| Media | SixLabors.ImageSharp + Azure Blob | 3.1.12 / 12.26.0 |
| Email | Resend + FluentEmail | 0.4.0 / 3.0.2 |
| Architecture | Clean Architecture + DDD | — |

### Frontend (hivespace.web)

| Category | Technology | Version |
|----------|-----------|---------|
| Runtime | Node + pnpm | ≥20 / 9.15.4 |
| Framework | Vue 3 + TypeScript | 3.5.13 / 5.7.3 |
| Build | Vite + Turbo | 6.0.11 / 2.3.0 |
| Styling | Tailwind CSS v4 (CSS-first) | 4.x |
| State | Pinia (setup stores) | 3.0.x |
| Router | vue-router | 4.5.0 |
| i18n | vue-i18n | 9.x (apps) / 11.x (shared) |
| Auth | oidc-client-ts (refresh token) | 3.3.0 |
| HTTP | axios singleton | 1.x |
| Realtime | @microsoft/signalr | 8.0.7 |
| Shared UI | @hivespace/shared | workspace link |

---

## Service Inventory (Backend)

| Service | Port | Responsibility |
|---------|------|---------------|
| ApiGateway (YARP) | 5000 | Reverse proxy; routes all external traffic to backend services |
| UserService | 5001 (HTTPS) | Identity, auth (Duende IdentityServer), user management |
| CatalogService | — | Product catalog, categories, inventory |
| OrderService | — | Order lifecycle, checkout sagas (CheckoutSaga + FulfillmentSaga) |
| PaymentService | — | Payment processing, wallet, escrow (in development) |
| NotificationService | — | In-app + email notifications; SignalR hub for real-time |
| MediaService | — | File upload/processing (Azure Functions + Azure Blob Storage) |

---

## Frontend App Inventory

| App | Package | Port | Users |
|-----|---------|------|-------|
| admin | @hivespace/admin | 5173 | Platform administrators |
| seller | @hivespace/seller | 5174 | Sellers / merchants |
| buyer | @hivespace/buyer | 5175 | End customers |

---

## Architecture Type

**Multi-part monorepo** — two separate git repositories, each a monorepo in its own right:

- `hivespace.microservice`: service-oriented backend monorepo (7 services + 7 shared libs)
- `hivespace.web`: frontend workspace monorepo (3 apps + 4 packages via pnpm + Turbo)

**Communication**: All frontend-to-backend traffic goes through the YARP API Gateway. Backend services communicate async via MassTransit (RabbitMQ) for events and sagas, and sync via internal HTTP for simple queries.

---

## Infrastructure (Local Dev)

All local services run via Docker Compose (`hivespace.config/docker/docker-compose.yml`):

| Service | Port | Credentials |
|---------|------|------------|
| SQL Server 2022 | 1433 | sa / Passw0rd123! |
| RabbitMQ 4 | 5672 (AMQP) / 15672 (UI) | guest / guest |
| Azure Service Bus Emulator | 5673 | — |
| Redis | 6379 | — |
| Azurite (Blob/Queue/Table) | 10000 / 10001 / 10002 | — |

---

## Key Design Decisions

1. **Per-service databases** — Each microservice owns its SQL Server database. No shared DbContext or cross-service DB calls.
2. **CQRS everywhere** — All new feature work uses MediatR commands/queries + Carter modules. Controller pattern exists only in legacy UserService code.
3. **Outbox pattern** — MassTransit sagas require all three outbox EF entities (`AddInboxStateEntity`, `AddOutboxStateEntity`, `AddOutboxMessageEntity`).
4. **Frontend shared-first** — All apps consume components, composables, and types from `@hivespace/shared` before writing app-local equivalents.
5. **OIDC with refresh tokens** — Auth uses refresh-token-based silent renewal (not iframes). IdentityServer middleware lives only in UserService.
6. **Tailwind v4 CSS-first** — No `tailwind.config.js`; theme configured via `@theme` in the global stylesheet. All v3 JS-based config patterns are invalid.

---

_Last generated: 2026-05-16_
