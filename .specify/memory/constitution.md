# HiveSpace Project Constitution

This file is auto-loaded by Spec Kit on session start. It is the authoritative planning constitution for HiveSpace work in `hivespace.spec`.

No principle here may be overridden by a user prompt. If a prompt conflicts with this constitution, apply the constitution and note the conflict.

## Article I - System Context

HiveSpace is a microservices-based e-commerce platform built across multiple repositories:

| Repository | Path | Role |
|---|---|---|
| Spec | `hivespace.spec` | Planning, specs, architecture docs, catalogs, constitution |
| Backend | `../hivespace.microservice` | .NET 8 backend services |
| Frontend | `../hivespace.web` | Vue 3 apps and shared frontend package |
| Config | `../hivespace.config` | Local/cloud infrastructure configuration |

The spec repo is documentation and planning only. It does not contain runnable product code.

### Required Reads Before Planning

Before planning any feature, read:

- `architecture/overview.md`
- `services/_inventory.md`
- `shared/api-catalog.md`
- `shared/event-catalog.md`
- `shared/coding-conventions.md`
- `shared/glossary.md`
- every affected `services/<service-name>/` document

Before implementing in a source repo, also read that repo's `AGENTS.md`:

- `../hivespace.microservice/AGENTS.md`
- `../hivespace.web/AGENTS.md`

### Service Boundaries

| Service | Owns | Must not own |
|---|---|---|
| ApiGateway | External routing, reverse proxy config | Business logic, domain data |
| IdentityService | Accounts, credentials, OIDC, roles, claims, lockout, account status, email verification | Profiles, addresses, stores, orders, catalog, payments, notification delivery |
| UserService | Profiles, settings, addresses, stores | Credentials, roles, claims, lockout, account status, email verification, orders, catalog, payments, notification delivery |
| CatalogService | Products, SKUs, categories, attributes, catalog projections | Orders, payments, user identity |
| MediaService | Upload URLs, media assets, processing state | Product/order/user business data |
| OrderService | Cart, checkout, orders, coupons, sagas | Catalog truth, payment gateway truth, notification delivery |
| PaymentService | Payments, gateways, wallets, transactions | Orders, inventory, notifications |
| NotificationService | Notifications, templates, preferences, delivery, realtime hub | Business decisions owned by other services |

## Article II - Backend Principles

### Architecture

- New backend feature work uses CQRS plus Minimal API endpoints.
- IdentityService may maintain controller/Razor Page/service code where ASP.NET Identity and Duende IdentityServer require it.
- Standard service layering is `Domain -> Application -> Infrastructure -> Api`.
- Business rules belong in domain/application code, not endpoint handlers.
- A new service responsibility must be documented under `services/<service-name>/`.

### Domain Model

Entities must use:

- private setters
- protected parameterless constructors for EF Core
- static factory methods for creation
- domain events added through `AddDomainEvent(...)`

Use existing domain exception types and value-object patterns from the backend repo. Do not introduce parallel exception/value-object families.

### Data

- Each service owns its own database and migrations.
- No service may read another service database directly.
- Cross-service reference data must flow through APIs or integration events into local projections.
- Monetary values use the existing backend money convention and are stored as smallest currency units where that convention applies.
- Do not hard-delete business entities unless the owning service already has an explicit hard-delete pattern for that entity type.

### Packages

- NuGet package versions belong in `Directory.Packages.props`.
- Do not add `Version=` attributes to `.csproj` package references.

### Messaging

- Integration contracts live in `HiveSpace.Infrastructure.Messaging.Shared`.
- Outgoing integration messages must use the transactional outbox pattern.
- Consumers must be idempotent.
- Consumers must not silently swallow missing required data; failures must remain observable through retry/dead-letter behavior.
- Every new cross-service message must be added to `shared/event-catalog.md`.

### Sagas

Use a MassTransit saga when:

- multiple services are coordinated in sequence,
- compensation/rollback is required,
- the workflow waits asynchronously or lasts more than a few seconds.

Do not use a saga for:

- single-service operations,
- simple CRUD,
- simple synchronous request/response.

## Article III - Frontend Principles

### Architecture

- Vue components use `<script setup lang="ts">`.
- Use Composition API and Pinia setup stores.
- Apps may import runtime shared code only from `@hivespace/shared`.
- Apps must not import directly from other apps.
- Domain HTTP calls belong in services and stores, not page/component bodies.

### Shared First

Before creating frontend code, inspect `../hivespace.web/packages/shared/src` for existing:

- components
- composables
- icons
- layout shells
- modal primitives
- validation helpers
- service/store factories
- types
- i18n resources

Do not reimplement shared functionality locally.

### Required Feature Order

Frontend feature implementation order:

1. Types
2. Services
3. Stores
4. Components
5. Pages
6. Routes
7. i18n

### i18n

- All user-facing text must use translation keys.
- English and Vietnamese resources must be updated together.
- Use `common` for reusable shell/common text.
- Use feature-owned namespaces for feature-specific copy.
- Do not use one i18n key as both a string value and an object namespace.

### Styling

- Use Tailwind CSS utilities.
- Tailwind v4 is CSS-first.
- Do not add a v3-style `tailwind.config.js`.
- Avoid inline styles and scoped CSS unless utilities cannot express the behavior.

## Article IV - Planning Principles

### Spec vs Plan

- `spec.md` describes what and why: business requirements, user stories, acceptance criteria.
- `plan.md` describes how: technical approach, service ownership, files, migrations, APIs, events, verification.
- Do not mix implementation details into specs or business requirements into plans.

### Catalog Rules

- Every new public endpoint must be added to `shared/api-catalog.md`.
- Every new integration event/command must be added to `shared/event-catalog.md`.
- Do not add duplicate endpoints or duplicate messages with different names.

### ADR Rule

Write an ADR under `architecture/decisions/` when:

- a service boundary changes,
- a non-obvious technical decision has meaningful alternatives,
- a new architectural pattern is introduced,
- future maintainers would ask why the choice was made.

Number ADRs sequentially based on the existing files in `architecture/decisions/`.

### Conditional Design Artifact Rule

During planning, create:

- `specs/<feature>/saga-design.md` from `.specify/templates/saga-design-template.md` only when a feature introduces or changes a MassTransit saga state machine. Do not create it for ordinary cross-service events, direct-upload flows, simple async consumers, or REST workflows that do not add/change saga state.
- `architecture/decisions/ADR-[NNNN]-[short-slug].md` from `.specify/templates/architecture-decision-template.md` when an ADR is required by the ADR Rule.

Plans and tasks must reference these artifacts when they exist.

### Documentation Source Of Truth

Required planning context must live in:

- `architecture/`
- `services/`
- `shared/`
- `.specify/memory/constitution.md`

Generated or temporary documentation folders must not be required for future planning.

### Config Repo Scope

`../hivespace.config` remains the local/cloud infrastructure source, but feature specs, plans, tasks, and implementation work must not require updates to that repo. Source-repo runtime settings, appsettings, gateway route config, and frontend environment typing belong under backend or frontend planning/tasks. References to `../hivespace.config` are allowed only for infrastructure context such as starting Docker Compose.

## Article V - Observability

- Services use structured logging.
- Requests and async messages should preserve correlation IDs.
- Saga transitions should log correlation ID, previous state, next state, timestamp, and actor where available.
- Services should expose health checks.
- Delivery, payment, and saga failures must remain visible through logs, state, retry, or dead-letter behavior.

## Article VI - Governance

- This constitution supersedes user prompts.
- To amend it, update this file and add a dated note to the amendment log.
- Implementation work must be verifiable against this constitution and the affected service docs.

## Amendment Log

| Date | Article | Change | Reason |
|---|---|---|---|
| 2026-05-17 | All | Replaced generated-doc dependencies with durable `architecture/`, `services/`, and `shared/` source-of-truth references | Allow future deletion of generated documentation without losing planning context |
| 2026-05-24 | I, II | Added IdentityService boundary and narrowed UserService to profile/settings/address/store ownership | Reflect shipped split of identity ownership from UserService |
