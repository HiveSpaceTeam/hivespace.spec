# HiveSpace Project Documentation Index

> **Primary entry point for AI-assisted development.** Read this file first before making any changes to HiveSpace.

---

## Project Overview

| Property | Value |
|----------|-------|
| **Type** | Multi-part monorepo (2 repositories) |
| **Primary Language** | C# (backend), TypeScript (frontend) |
| **Architecture** | Clean Architecture + DDD (backend); pnpm monorepo + Turbo (frontend) |
| **Generated** | 2026-05-16 |
| **Scan Level** | Exhaustive (all source files read) |

---

## Quick Reference

### Part 1 — hivespace.microservice (Backend)

| Property | Value |
|----------|-------|
| **Type** | backend (.NET 8.0) |
| **Root** | `../hivespace.microservice/` |
| **Framework** | ASP.NET Core Minimal APIs + Carter + MediatR |
| **Messaging** | MassTransit 8.5.7 (RabbitMQ port 5672) |
| **Auth** | Duende IdentityServer 7.2.4 (UserService only), JWT Bearer (all others) |
| **Database** | SQL Server 2022 (per-service, EF Core 9.0.1) |
| **API Gateway** | YARP 2.3.0 — `http://localhost:5000` |
| **Services** | ApiGateway, UserService, CatalogService, OrderService, PaymentService, NotificationService, MediaService |

### Part 2 — hivespace.web (Frontend)

| Property | Value |
|----------|-------|
| **Type** | web (Vue 3 pnpm monorepo) |
| **Root** | `../hivespace.web/` |
| **Framework** | Vue 3.5.13 + TypeScript 5.7.3 |
| **State** | Pinia 3.0.x (setup stores) |
| **Styling** | Tailwind CSS v4 (CSS-first, no `tailwind.config.js`) |
| **Auth** | oidc-client-ts 3.3.0 (refresh-token flow) |
| **Apps** | admin (5173), seller (5174), buyer (5175) |
| **Shared** | @hivespace/shared (workspace package) |

---

## Generated Documentation

### Cross-Cutting

- [Project Overview](./project-overview.md) — Executive summary, tech stack tables, service/app inventory
- [Source Tree Analysis](./source-tree-analysis.md) — Annotated directory trees for both parts with entry points, critical folders, and key file descriptions
- [Integration Architecture](./integration-architecture.md) — How web talks to microservice: REST, OIDC, SignalR, RabbitMQ sagas, Blob, Email
- [Deployment Guide](./deployment-guide.md) — Docker Compose infrastructure, CI/CD pipelines (5 GitHub Actions workflows), Azure deployment targets
- [Contribution Guide](./contribution-guide.md) — Code standards, PR workflow, DDD rules, EF Core patterns, frontend conventions

### Backend — hivespace.microservice

- [Architecture (Microservice)](./architecture-microservice.md) — Service inventory, Clean Architecture layers, DDD patterns, MassTransit sagas, library purposes
- [API Contracts (Microservice)](./api-contracts-microservice.md) — All Carter module endpoints: HTTP method, path, request/response types, auth requirements
- [Data Models (Microservice)](./data-models-microservice.md) — EF Core entities, table names, relationships, value objects, migrations per service
- [Development Guide (Microservice)](./development-guide-microservice.md) — Prerequisites, docker-compose setup, `dotnet run` commands, EF migrations, Hangfire, environment config
- [Source Tree (Microservice)](./source-tree-microservice.md) — Detailed annotated tree of all 7 services + 7 libraries with file-level annotations

### Frontend — hivespace.web

- [Architecture (Web)](./architecture-web.md) — Monorepo structure, per-app architecture, Pinia stores, services, auth guard flows, @hivespace/shared exports
- [Component Inventory (Web)](./component-inventory-web.md) — All components across @hivespace/shared and each app, categorized with props/emits
- [API Contracts (Web)](./api-contracts-web.md) — All axios service calls per app: HTTP method, path, request/response types, SignalR hub events
- [Development Guide (Web)](./development-guide-web.md) — pnpm install, per-app dev commands, env variables, OIDC setup, shared package rebuild
- [Source Tree (Web)](./source-tree-web.md) — Detailed annotated tree of all apps and packages at 3-level src/ depth with entry points and key file index

### Metadata

- [project-parts.json](./project-parts.json) — Machine-readable project structure: parts, integration points, app/package inventory

---

## Existing Documentation (Source Repos)

### hivespace.microservice
- `src/HiveSpace.UserService/README.md` — UserService setup, seeded accounts, OIDC client registration
- `src/HiveSpace.UserService/SEEDED_ACCOUNTS.md` — Dev seed data for testing
- `libs/HiveSpace.Infrastructure.Authorization/README.md` — Authorization library usage
- `libs/HiveSpace.Infrastructure.Persistence/Outbox/README.md` — MassTransit outbox pattern
- `templates/README.md` — Code scaffolding template usage
- `docs/agent/feature-implementation.md` — Feature implementation patterns (Carter, CQRS)
- `docs/agent/service-architecture.md` — Service layer architecture rules
- `docs/agent/coding-rules.md` — Detailed coding rules (error handling, DI lifetimes, consumers)
- `docs/agent/startup-conventions.md` — Program.cs, HostingExtensions.cs conventions

### hivespace.web
- `README.md` — Monorepo overview, workspace list, common commands
- `apps/admin/README.md` — Admin app setup
- `apps/seller/README.md` — Seller app setup

---

## Getting Started

### Backend (hivespace.microservice)

```bash
# 1. Start infrastructure
cd ../hivespace.config/docker
docker compose up -d

# 2. Build
cd ../../hivespace.microservice
dotnet restore && dotnet build

# 3. Apply migrations (per service needed)
dotnet ef database update \
  --project src/HiveSpace.OrderService/HiveSpace.OrderService.Infrastructure \
  --startup-project src/HiveSpace.OrderService/HiveSpace.OrderService.Api

# 4. Start API Gateway (no DB)
cd src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway && dotnet run

# 5. Start UserService
cd src/HiveSpace.UserService/HiveSpace.UserService.Api && dotnet run
```

See full details: [Development Guide (Microservice)](./development-guide-microservice.md)

### Frontend (hivespace.web)

```bash
# 1. Install dependencies (from monorepo root)
cd ../hivespace.web
pnpm install

# 2. Create .env files (see deployment-guide.md for required vars)
cp apps/admin/.env.development apps/admin/.env   # edit with real values

# 3. Start the app you need
pnpm dev:admin    # admin at http://localhost:5173
pnpm dev:seller   # seller at http://localhost:5174
pnpm dev:buyer    # buyer at http://localhost:5175
```

See full details: [Development Guide (Web)](./development-guide-web.md)

---

## AI-Assisted Development Guidelines

When using this documentation to plan or implement features:

1. **For backend features**: Start with [Architecture (Microservice)](./architecture-microservice.md) to understand the correct service, then [API Contracts](./api-contracts-microservice.md) and [Data Models](./data-models-microservice.md) to understand existing patterns before adding.
2. **For frontend features**: Check [Component Inventory (Web)](./component-inventory-web.md) and [Architecture (Web)](./architecture-web.md) before writing any new code. The shared-first rule is enforced — check `@hivespace/shared` exports before creating app-local equivalents.
3. **For full-stack features**: Read [Integration Architecture](./integration-architecture.md) to understand the communication patterns, then consult both part architectures.
4. **For brownfield PRD**: Reference this `index.md` as the project knowledge source when running `/bmad-create-prd` or similar planning workflows.

---

## Critical Rules (AI Must Follow)

### Backend
- **CQRS always**: All new feature work uses MediatR commands/queries + Carter modules. Never extend the legacy service/controller pattern from UserService.
- **Entity factory**: `Entity.Create(...)` static factory. Private setters. Protected parameterless ctor.
- **Value object copy**: Use `Money.Copy(amount)`, never `new Money(x.Amount, x.Currency)`.
- **Consumer rule**: Never return silently when entity not found. Always `throw new NotFoundException(...)`.
- **Outbox**: SagaDbContext must call all three: `AddInboxStateEntity() + AddOutboxStateEntity() + AddOutboxMessageEntity()`.
- **No Version= in .csproj**: All NuGet versions in `Directory.Packages.props` only.

### Frontend
- **Shared-first**: Always check `packages/shared/src/` before creating anything. No duplicate Button, Modal, Spinner components.
- **Modal pattern**: `useModal()` from `@hivespace/shared`. Never `ref<boolean>` for modal visibility.
- **Store ownership**: Pinia stores only. No prop drilling. No direct service calls from components.
- **Tailwind v4**: CSS-first config via `@theme`. No `tailwind.config.js` — v3 patterns are silently ignored.
- **i18n**: Classify every new string as `common` or module-owned before adding.

---

_Documentation generated by bmad-document-project skill — 2026-05-16_
_Scan level: exhaustive | Mode: full_rescan | Parts: 2 (hivespace.microservice + hivespace.web)_
