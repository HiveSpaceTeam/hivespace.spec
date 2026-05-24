# HiveSpace Coding Conventions

## Purpose

These conventions are the planning baseline for all HiveSpace implementation work. They apply unless an affected repo has a stricter `AGENTS.md`.

Before editing code, read the relevant source repo instructions:

- Backend: `../hivespace.microservice/AGENTS.md`
- Frontend: `../hivespace.web/AGENTS.md`

## Universal Rules

- Keep changes scoped to the requested feature or bug.
- Prefer existing local patterns over new abstractions.
- Do not introduce duplicate API endpoints, events, shared UI primitives, or service responsibilities.
- Record new public APIs in `shared/api-catalog.md`.
- Record new cross-service commands/events in `shared/event-catalog.md`.
- Respect service-owned data; never read another service database directly.
- Use comments sparingly and only for code whose intent is not obvious.

## Backend Rules

### Architecture

- New feature work uses CQRS plus Minimal API endpoints.
- IdentityService may maintain controller/Razor Page/service code where ASP.NET Identity and Duende IdentityServer require it.
- Standard services follow `Domain -> Application -> Infrastructure -> Api`.
- Keep business rules in domain/application code, not endpoint handlers.
- Use MassTransit sagas only for multi-service workflows that need async coordination, timeout, or compensation.

### Entities and Domain

- Entities use private setters.
- Entities include a protected parameterless constructor for EF Core.
- Creation goes through static factory methods such as `Create(...)`.
- Domain events are added through `AddDomainEvent(...)`.
- Use strongly typed IDs where the service/domain already does so.
- Do not hard-delete business records; use status fields or soft-delete conventions.

### Data and Money

- Each service owns its database, DbContext, migrations, and table schema.
- Table and column names use snake_case where explicitly configured.
- Monetary values are stored as `long` in the smallest currency unit, especially VND.
- Do not use `decimal` for persisted money unless a source repo has already established that exact type for the affected model.
- Copy value objects using the existing copy helper/pattern; do not reconstruct a value object from its own properties when EF Core tracking needs distinct instances.

### Packages and Dependencies

- NuGet versions belong in `Directory.Packages.props`.
- Do not add `Version=` to individual `.csproj` package references.
- Reuse shared libraries before adding service-local infrastructure.

### Messaging

- Outgoing integration messages must be published through the transactional outbox pattern.
- Consumers must be idempotent.
- Consumers must not silently return when required entities are missing; surface a domain/not-found failure so retries and dead-letter handling preserve observability.
- Message contracts live in `HiveSpace.Infrastructure.Messaging.Shared`.

### Errors and Observability

- Use existing domain exceptions for business and not-found failures.
- Preserve correlation IDs across gateway, service, message, and log boundaries.
- All services should expose health checks.
- Saga transitions should log correlation ID, source/target state, timestamp, and actor where available.

## Frontend Rules

### Architecture

- Vue components use `<script setup lang="ts">`.
- Use Composition API and Pinia setup stores.
- Apps import shared runtime code only from `@hivespace/shared`.
- Apps must not import directly from another app.
- Domain HTTP calls belong in service modules and stores, not components or pages.
- Use `storeToRefs` when destructuring store state.

### Shared First

Check `../hivespace.web/packages/shared/src` before creating:

- UI components
- layout shells
- modal flows
- icons
- composables
- validation helpers
- stores or store factories
- services or service factories
- feature types

Use shared modal primitives:

- `useModal`
- `useConfirmModal`
- `ModalManager`
- `ModalWrapper`
- `ConfirmModal`

Use shared loading primitives instead of inline spinner markup:

- `Spinner`
- `FullscreenLoader`

### Frontend Build Order

For a new feature, implement in this order:

1. Types
2. Services
3. Stores
4. Components
5. Pages
6. Router entries
7. i18n resources

### Naming

- Vue files: PascalCase.
- TypeScript service/util files: kebab-case.
- Stores: `[module].store.ts` where the app uses that convention.
- Variables/functions: camelCase.
- Use arrow functions unless a top-level declaration needs hoisting.

### i18n

- All user-facing strings use translation keys.
- Update English and Vietnamese resources together.
- Use `common` for app-shell and reusable text.
- Use feature-owned namespaces for domain copy.
- Do not reuse one i18n key as both a string and an object namespace.

### Styling

- Use Tailwind CSS utilities.
- Tailwind v4 configuration is CSS-first.
- Do not add `tailwind.config.js` for v3-style configuration.
- Avoid inline styles and scoped CSS unless a utility cannot express the behavior.

## Documentation Rules

- `architecture/overview.md` is the system overview.
- `services/_inventory.md` is the service index.
- `services/<service-name>/` owns service-specific documentation.
- `shared/api-catalog.md` owns public API inventory.
- `shared/event-catalog.md` owns integration message inventory.
- `shared/glossary.md` owns shared vocabulary.
- Do not make required planning docs depend on generated documentation directories.

## Verification Rules

Backend:

- Prefer targeted tests if present.
- Otherwise run the narrowest relevant `dotnet build` or service build.
- Expect known development warnings only when already documented by the source repo.

Frontend:

- Prefer `pnpm type-check`, `pnpm lint`, or targeted app builds when configured.
- Do not run formatters that rewrite files unless formatting is the task.
- Preserve known baseline failures and call them out rather than fixing unrelated areas.

Docs-only changes:

- Verify changed markdown files are non-empty.
- Search for stale links to deleted or soon-to-be-deleted docs.
- Confirm catalog entries and service names match the source repos.
