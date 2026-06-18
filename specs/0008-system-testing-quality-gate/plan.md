# Implementation Plan: System Testing Quality Gate

**Branch:** 0008-system-testing-quality-gate
**Date:** 2026-06-12
**Spec:** [specs/0008-system-testing-quality-gate/spec.md](spec.md)

---

## Phase 0 — Research

### Existing context (read before planning)

- [x] `services/<owner-or-supporting>/README.md` — existing aggregates, events, endpoints
- [x] `shared/event-catalog.md` — verify no event name conflicts
- [x] `shared/api-catalog.md` — verify no endpoint conflicts
- [x] `research.md` — all NEEDS CLARIFICATION items resolved

### Technical unknowns

All unknowns resolved. See `research.md`.

### Research notes

See `specs/0008-system-testing-quality-gate/research.md` for resolved decisions covering test architecture, repo structure, provider substitution strategy, and rerun/accepted-risk policy.

---

## Phase 1 — Architecture & Data Model

### Service placement

This feature does not add domain behavior to any single service. It adds test projects to each backend service and test files to each frontend app. Ownership follows the source repository structure:

- Backend test projects live in `../hivespace.microservice/tests/`
- Frontend test files are co-located in `../hivespace.web/apps/<app>/src/`
- Planning artifacts live in `hivespace.spec` (this repo)

Classify every service mentioned by the feature:

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| IdentityService | Owning service (test scope) | Receives a new test project covering auth, session, and admin account journeys | No service doc or catalog changes; test-only |
| UserService | Owning service (test scope) | Receives a new test project covering profile, address, store, and settings journeys | No service doc or catalog changes; test-only |
| CatalogService | Owning service (test scope) | Receives a new test project covering storefront discovery and seller product journeys | No service doc or catalog changes; test-only |
| OrderService | Owning service (test scope) | Receives a new test project covering cart, checkout, saga, and order journeys | No service doc or catalog changes; test-only |
| PaymentService | Owning service (test scope) | Receives updates to the existing test project covering payment idempotency and wallet journeys | No service doc or catalog changes; test-only |
| MediaService | Owning service (test scope) | Receives a new test project covering presign URL and upload confirmation journeys | No service doc or catalog changes; test-only |
| NotificationService | Owning service (test scope) | Receives a new test project covering notification delivery, preferences, and realtime journeys | No service doc or catalog changes; test-only |
| ApiGateway | Excluded from test coverage | ApiGateway owns only routing and reverse proxy configuration — no domain model, no EF Core, no aggregates, no business logic. Gateway route behavior is validated indirectly through integration tests that hit service endpoints. A dedicated gateway test project would be brittle configuration-level testing without meaningful business-behavior value. See ADR-0007. | No test project; no service doc or catalog changes |
| HiveSpace.Testing.Shared | New shared test library | Fakes, builders, and doubles shared by all backend test projects | No service doc changes; new project under `tests/` |

### New aggregates

None. This feature does not introduce domain aggregates, EF Core tables, or database migrations. Quality-gate planning entities are defined in `data-model.md` as documentation concepts only.

### New repository interfaces

None.

### Integration events (must go through Outbox)

None. No new cross-service events are introduced. Reused common event contracts are unchanged — no `shared/event-catalog.md` update required.

### API endpoints

None. No new or changed public endpoints are introduced — no `shared/api-catalog.md` update required.

---

## Testing Rules

Both repos have `TESTING.md` files defining their rules. This section codifies the constraints the spec enforces as quality gates. Full rules are in `testing-rules.md`.

### Backend (.NET / xUnit)

Source: `../hivespace.microservice/TESTING.md`

- **Framework**: xUnit + FluentAssertions + Coverlet
- **One test file per production class**: `CreateProductCommandHandlerTests.cs` for `CreateProductCommandHandler`
- **Naming**: `Method_Condition_ExpectedOutcome` (e.g., `Handle_ValidCommand_PublishesProductCreatedEvent`)
- **Domain tests**: pure `[Fact]`, no DB, no fakes, no fixtures
- **Application tests**: execute the real Application-layer unit; use `NSubstitute` when orchestration verification is sufficient, and use `IClassFixture<ServiceFixture>` with in-memory EF Core when persistence/query round-tripping must be observed
- **Consumer tests**: `InMemoryMessageCapture` to assert event publishing
- **No live infrastructure**: use `PaymentProviderFake`, `EmailDeliveryFake`, `BlobStorageFake`, `SignalRHubFake` from `HiveSpace.Testing.Shared`
- **Coverage scope** (from `coverage.runsettings`): `[HiveSpace.*.Application]*`, `[HiveSpace.*.Domain]*`, `[HiveSpace.*.Core]*` — excludes Infrastructure, Api, migrations

**Coverage target**: 80% line on measured layers per service  
**Enforcement**: `quality-gate.ps1` — task B021 adds threshold check (currently the script reports only)

### Frontend (Jest)

Source: `../hivespace.web/TESTING.md`

- **Framework**: Jest 29 + jsdom + `@testing-library/vue` + `@pinia/testing`
- **Co-location**: test files live next to source — `stores/cart.test.ts` alongside `stores/cart.ts`
- **Store tests**: `@pinia/testing`, state and actions only, no component mounting
- **View/page tests**: `@testing-library/vue`, mount with wired stores, stub HTTP via `stubApiResponse`
- **Naming stores**: `action_When_Result` (e.g., `login_WhenCredentialsValid_SetsAuthState`)
- **Naming views**: `renders_What_FromWhere` or `action_CallsEndpoint`
- **No real HTTP**: stub via `createMockAxios()` / `stubApiResponse()`
- **Shared test-utils** from `@hivespace/shared/test-utils`: `createFakeAuthUser`, `createFakeAuthState`, `createMockAxios`, `stubApiResponse`, `createFakeSignalRHub`, `createTestI18n`
- **i18n**: assert keys only, not hardcoded strings

**Coverage target**: 80% policy-scoped line coverage  
**Enforcement**: existing `coverage.ps1` (exits with code 1 below 80%)

---

## Phase 2 — Implementation Plan

See detailed tasks in:

- `specs/0008-system-testing-quality-gate/tasks/backend.md` — backend test projects (B001–B013, B020), threshold enforcement (B021)
- `specs/0008-system-testing-quality-gate/tasks/frontend.md` — frontend test scripts and files (F001–F012)
- `specs/0008-system-testing-quality-gate/tasks/docs-catalog.md` — contract and ADR (D001–D002)
- `specs/0008-system-testing-quality-gate/tasks/verification.md` — build, test, runner, coverage, diagnostic (V001–V013)

### Layer order (backend)

This feature does not follow the standard domain → application → infrastructure → api layer order because no product code is added. The backend implementation order is:

1. Shared infrastructure: `HiveSpace.Testing.Shared` (fakes, builders, doubles) — B001, B002
2. Service test projects per service (parallel after B001/B002) — B004–B011
3. Quality-gate runner `quality-gate.ps1` (after service test projects) — B003
4. Solution file update and developer guide — B012, B013
5. Remaining PaymentService coverage — B020

---

## Phase 2B — Systematic Coverage Expansion

### Motivation

Current line coverage is approximately 8% and branch coverage is 15.9% — insufficient for production confidence. The initial tasks (B001–B013) establish the test project structure and critical-journey tests. Phase 2B adds a handler-by-handler expansion to cover **every class with a public method** in Domain, Application, and Core layers.

### Scope rule

The authoritative backend coverage scope is defined in `../hivespace.microservice/coverage.runsettings`. The table below reflects its include/exclude configuration:

| Layer | Include | Exclude |
|-------|---------|---------|
| Domain | Aggregates, entities, value objects, domain services, specifications | Interfaces, enumerations with no logic |
| Application | Command handlers, query handlers, application services, validators, helpers, factory classes | Interfaces, DTOs |
| Core (Lite Services) | All public-method classes in `*.Core` projects | Interfaces, DTOs; sub-paths: `Core/Infrastructure/**`, `Core/Persistence/**`, `Core/Interfaces/**`, `Core/Extensions/**`, `Migrations/**` |
| **Excluded assemblies** | — | `[HiveSpace.*.Api]*`, `[HiveSpace.*.Infrastructure]*`, `[HiveSpace.ServiceDefaults]*`, `[HiveSpace.AppHost]*`, `[HiveSpace.Core]*` (generic lib), `[HiveSpace.Application.Shared]*`, `[HiveSpace.Domain.Shared]*`, `[HiveSpace.*.Tests]*`, `[HiveSpace.Testing.Shared]*` |

### Full Service vs Lite Service

**Full Service** (separate Domain + Application layers): CatalogService, OrderService, PaymentService, UserService.

**Lite Service** (Core layer only, no separate Domain/Application split): IdentityService, NotificationService, MediaService. These use `IClassFixture<XxxServiceFixture>` for handler tests. Domain logic lives inside the Core project — write pure `[Fact]` tests for any class with state-transition or invariant methods.

### Coverage targets

All services must reach **80% line coverage** on Domain and Application layers. This is enforced by `quality-gate.ps1` (task B021 adds the threshold check — currently the script reports only).

| Service | Measured layers | Target Line % |
|---------|----------------|---------------|
| CatalogService | Domain + Application | ≥80% |
| OrderService | Domain + Application | ≥80% |
| PaymentService | Domain + Application | ≥80% |
| IdentityService | Core (no separate Domain/Application split) | ≥80% |
| NotificationService | Core | ≥80% |
| MediaService | Core | ≥80% |
| UserService | Domain + Application | ≥80% |

### Coverage expansion summary

All CatalogService, OrderService, and UserService handler and domain coverage has been folded directly into B007, B008, and B006 respectively — organized by layer (Domain/, Application/, Consumers/). These tasks are now self-contained comprehensive deliverables.

**PaymentService** remaining coverage (B020):
- `GetTransactionHistoryQueryHandler`; `Payment` aggregate state-transition tests

See tasks B006–B011, B020 in `tasks/backend.md` for full file-by-file details.

### Saga design

No saga required. This feature adds test and quality-gate infrastructure only. No MassTransit saga state machine is introduced or changed.

### Architecture decision

See `architecture/decisions/ADR-0007-system-testing-quality-gate.md`. Key decisions:

- Repo-local test suites coordinated by a shared quality-gate report contract
- ApiGateway excluded from dedicated test coverage (routing config, no business behavior)
- Backend test projects extend the existing xUnit/FluentAssertions/coverlet pattern
- Frontend tests wired through pnpm/Turbo workspace scripts
- Controlled provider substitutes (fakes) for payment, email, blob, and SignalR

---

## Phase 3 — Frontend Plan

### Surface(s)

- [x] buyer (`apps/buyer`)
- [x] seller (`apps/seller`)
- [x] admin (`apps/admin`)
- [x] shared (`packages/shared`)

### Test runner

The project uses **Jest** (via `jest.base.cjs` factory config), not Vitest. Per-app `jest.config.cjs` files exist under `apps/buyer/`, `apps/seller/`, and `apps/admin/`. `packages/shared/jest.config.cjs` also exists. Do not create `vitest.config.ts` files.

Coverage policy: **80% policy-scoped line coverage**, enforced by the existing `coverage.ps1` script (exits with code 1 if any workspace is below 80%). Paths measured by Jest v8 coverage provider:
- Included (apps): `src/pages/**`, `src/stores/**`, `src/composables/**`, `src/router/**`
- Included (shared): `src/features/**`, `src/composables/**`, `src/test-utils/**`
- Excluded: assets, styles, i18n locale files, types, barrel index files, thin `services/**` wrappers, `config/**` constants, presentational `components/**`, icons

### Files to create

This feature creates all test files as new deliverables. Test infrastructure (`jest.config.cjs`, `jest.base.cjs`, shared test-utils, coverage scripts) already exists in the repo and is not recreated.

| File | Status |
| ---- | ------ |
| `packages/shared/src/test-utils/` | Test-utils infrastructure already present; test coverage must reach 80% on measured paths |
| `apps/buyer/src/pages/**/*.test.ts` | Create buyer critical-path tests co-located under `pages/<SubFolder>/` |
| `apps/buyer/src/stores/**/*.test.ts` | Create buyer store tests co-located with source |
| `apps/seller/src/pages/**/*.test.ts` | Create seller critical-path tests co-located under `pages/<SubFolder>/` |
| `apps/seller/src/stores/**/*.test.ts` | Create seller store tests co-located with source |
| `apps/admin/src/pages/**/*.test.ts` | Create admin critical-path tests co-located under `pages/<SubFolder>/` |
| `apps/admin/src/stores/**/*.test.ts` | Create admin store tests co-located with source |
| `scripts/quality-gate.mjs` | Frontend quality-gate runner at workspace root |

See `specs/0008-system-testing-quality-gate/tasks/frontend.md` for full detail.

---

## Constitution Compliance Check

Before implementation starts, verify all of these:

- [x] No hard-deletes planned — N/A (no product data changes)
- [x] All money values are long — N/A (no new domain entities)
- [x] All IDs use ULID — N/A (no new domain entities)
- [x] All events go through MassTransit Outbox — N/A (no new integration events)
- [x] No Version= in any new .csproj dependencies — enforced in B001/B002; all versions must remain in `Directory.Packages.props`
- [x] Both en.json and vi.json will be updated — N/A (no user-facing text changes)
- [x] Frontend text uses $t() keys only — N/A (no new frontend UI)
