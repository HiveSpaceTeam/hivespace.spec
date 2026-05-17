# Contribution Guide

This guide captures the code standards, workflows, and mandatory rules for contributing to HiveSpace. It applies to both `hivespace.microservice` (backend) and `hivespace.web` (frontend).

---

## General Principles (Both Repos)

### Think Before Coding

- State assumptions explicitly before implementing when they affect design.
- If multiple interpretations or trade-offs exist, surface them — do not pick silently.
- If something is unclear and the repo does not answer it, stop and ask.

### Simplicity First

- Implement only the requested behavior. No speculative abstractions or configurability.
- If the solution feels larger than the problem, simplify first.
- Every changed line should trace directly to the user's request.

### Surgical Changes

- Touch only the code and docs required for the request.
- Do not refactor adjacent code, comments, or formatting unless the task requires it.
- Remove only imports/variables created by your own change; mention unrelated cleanup separately.

### Goal-Driven Execution

- Define a concrete verification target before implementing.
- Every changed line should trace directly to the request and to a verification step.

---

## Backend (hivespace.microservice)

### Architecture Rules

All feature work **must** use Clean Architecture (Domain → Application → Infrastructure → Api) and CQRS:

- **Never** use the service-based/controller pattern for new features — this exists only in UserService legacy code.
- All new routes use Carter modules (`ICarterModule`). Never use `MapGet` on `WebApplication` directly.
- Duende IdentityServer middleware lives **only** in UserService. All other services use `AddJwtBearer` only.

### Feature Folder Structure (Mandatory)

```text
Application/[Feature]/
  Commands/
    [Action][Entity]Command.cs
    [Action][Entity]CommandHandler.cs
    [Action][Entity]Validator.cs
  Queries/
    Get[Entity]Query.cs
    Get[Entity]QueryHandler.cs
```

### DDD Entity Rules

- Entities: **private setters + protected parameterless constructor** + static `Create(...)` factory method.
- Value objects: Never reconstruct from own properties (`new Money(x.Amount, x.Currency)` is wrong). Use `Money.Copy(amount)` or `ValueObject.Copy<T>()`.
- Aggregate roots: Implement `IAggregateRoot` interface.

### EF Core Rules

- Table names: `snake_case` via `builder.ToTable("table_name")`. Never use PascalCase defaults.
- Migrations per service:
  ```bash
  dotnet ef migrations add <Name> \
    --project src/HiveSpace.{Service}/{Service}.Infrastructure \
    --startup-project src/HiveSpace.{Service}/{Service}.Api
  ```
- SagaDbContext must register all three outbox entities:
  ```csharp
  modelBuilder.AddInboxStateEntity();
  modelBuilder.AddOutboxStateEntity();
  modelBuilder.AddOutboxMessageEntity();
  ```
- Hangfire schema is initialized by `UseSqlServerStorage(...)` — **never** create EF migrations for Hangfire tables.

### MassTransit / Messaging Rules

- Consumers must **never return silently** when an entity is not found. Always `throw new NotFoundException(...)`.
- Use domain exceptions in consumers (`NotFoundException`, `InvalidFieldException`), never `System.InvalidOperationException`.
- Publish integration events **before** `SaveChangesAsync()` — same outbox rule as in handlers.
- All cross-service event/command contracts live in `libs/HiveSpace.Infrastructure.Messaging.Shared/` only.

### NuGet Package Management

- All versions in `Directory.Packages.props` **only**. Never add `Version=` in any `.csproj` file.

### Redis / DI Lifetime

- `StackExchange.Redis` `ConnectionMultiplexer` must be registered as **Singleton** only. Scoped or Transient causes connection pool exhaustion.

### Build Verification

```bash
dotnet restore
dotnet build   # Expect 6 nullability warnings in HiveSpace.Domain.Shared — non-blocking
```

No test projects exist — `dotnet test` returns immediately.

### Git / PR Rules

- **Never** stage or commit any `*.json` file. Ask user to stage JSON files manually.
- PR flow (enforced by hook):
  1. `bash scripts/sync-config.sh`
  2. `npx gitnexus analyze`
  3. Start new session → run `/review`
  4. Apply review fixes → then `gh pr create`
- Delete all temp files (logs, scratch, `.codex-build`, `*.lscache`) before finishing.
- Run `gitnexus_impact` on every symbol before modifying it. Stop and warn on HIGH or CRITICAL risk.
- Never rename symbols with find-and-replace — use `gitnexus_rename`.

---

## Frontend (hivespace.web)

### Component Development Rules

- **`<script setup lang="ts">` only.** No Options API anywhere.
- Always type `defineProps<{ ... }>()` and `defineEmits<{ ... }>()`.
- Routed views: `src/pages/*Page.vue` only. Non-route UI belongs in `src/components/{feature}/`.
- Tailwind CSS utility classes only; custom CSS only for non-utility styles.
- Formatting: 100-char line width, single quotes, no semicolons, 2-space indent.
- TypeScript strict mode: `noUnusedLocals`, `noUnusedParameters`.

### Shared-First Rule (Highest Priority)

Always check `packages/shared/src/` **before** creating anything:
- `useAuth`, `useAppStore`, `ApiService`, `AppUser` — never re-implement.
- `Pagination`, `Status`, `UserType`, `AuthConfig` — use from shared.
- Layout shells, shared icons — use from shared.
- Never create local `Button.vue`, `Modal.vue`, `Spinner.vue` duplicates.

### Modal Pattern

```typescript
import { useModal } from '@hivespace/shared'
const { openModal, closeModal } = useModal()
const result = await openModal(MyModalComponent, { prop1: value1 })
```

- `ModalManager` must exist in `App.vue`.
- Never use `ref<boolean>` for modal visibility.
- Use `ConfirmModal` for all destructive confirmation flows.

### Loading Pattern

- Interactive page: `<Spinner />` (size: 'sm' | 'md' | 'lg', default 'md')
- Blocking wait: `<FullscreenLoader :visible="bool" :message="string" />`
- **Never** write raw `<div class="animate-spin ...">` inline.

### Store and Service Pattern

- Pinia stores only — no prop drilling, no direct service calls from components.
- Import `useAppStore` directly from `@hivespace/shared`; never re-export it locally.
- Services own HTTP calls only; use the singleton from `src/services/api.ts`.
- `storeToRefs` required when destructuring stores to preserve reactivity.
- Store actions: wrap with `try/finally` calling `useAppStore().setLoading(true/false)`.
- Toast notifications: `useAppStore().notifySuccess/notifyError/notifyInfo` only.

### Implementation Order (Mandatory for Every Feature)

1. **Reuse check** — inspect `@hivespace/shared` first
2. **Types** — `src/types/{module}.types.ts` → export from `src/types/index.ts`
3. **Service** — `src/services/{module}.service.ts` using app singleton
4. **Store** — `src/stores/{module}.store.ts`, export from `src/stores/index.ts`
5. **Components** — `src/components/{feature}/`
6. **Pages** — `src/pages/{Feature}Page.vue`
7. **Route** — `src/router/index.ts`
8. **i18n** — classify keys (common vs. module-owned) before finishing

### Type Naming Rules

| Purpose | Naming Pattern |
|---------|---------------|
| Domain models | Plain singular: `Order`, `User`, `Product` |
| Endpoint request | `*Request`: `CreateProductRequest` |
| Endpoint query | `*Query`: `GetUserListQuery` |
| Endpoint response | `*Response`: `GetOrderDetailResponse` |
| Route params | `*Params` only for actual route params |
| Paged responses | Use `PaginationMetadata` from `@hivespace/shared` |
| Forbidden | `Dto`, `Api` suffix, `PagedResponse`, `CategoryResponse` |

### i18n Rules

- Default locale: `vi` (Vietnamese); fallback: `en`.
- Files: `src/i18n/locales/{en,vi}/{module}.json`.
- **Before adding any key**: classify as `common` or module-owned in the same change.
- `common.*` for: shell navigation, header/footer, generic errors, pagination, session-state messages, reusable actions.
- Module namespace for: feature forms/tables/filters, domain statuses, feature notifications.
- Never reuse one key as both a string value and an object namespace.

### Verification (Run After Every Task)

From the affected app directory:

```bash
pnpm lint
pnpm type-check
```

**Known baseline failures** (do not fix unless targeting those files):
- `type-check`: ~11 pre-existing errors (missing services, type violations in demo files)
- `lint`: ~15 pre-existing errors in demo components

After any task, verify no equivalent shared composable was introduced locally.

---

## Shared Package Development

When modifying `packages/shared`:

1. Make changes in `packages/shared/src/`
2. Run `pnpm build:shared` (from monorepo root) before consuming apps will see changes
3. Run `pnpm type-check` from `packages/shared/` to validate
4. Consuming apps don't need to be restarted if using watch mode with Turbo

---

_Last generated: 2026-05-16_
