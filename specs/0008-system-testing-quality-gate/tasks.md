# Tasks: System Testing Quality Gate

- **Input**: Design documents from `specs/0008-system-testing-quality-gate/`
- **Prerequisites**: plan.md (required), spec.md (required), research.md, data-model.md, contracts/quality-gate-report.md, quickstart.md, architecture/decisions/ADR-0007-system-testing-quality-gate.md
- **Detailed tasks**: `specs/0008-system-testing-quality-gate/tasks/`
- **Organization**: Tasks grouped by implementation ownership (backend, frontend, docs/catalog, verification), then by service/app/package, then by action.

---

## Detailed Task Files

| File | Purpose | Task ID prefix | Status |
| ---- | ------- | -------------- | ------ |
| `tasks/backend.md` | Test projects and quality-gate runner in `../hivespace.microservice` | `B###` | Not started |
| `tasks/frontend.md` | Test scripts, Jest configs, and test files in `../hivespace.web` | `F###` | Not started |
| `tasks/docs-catalog.md` | Quality-gate contract and ADR acceptance in `hivespace.spec` | `D###` | Not started |
| `tasks/verification.md` | Build, test, coverage map, and diagnostic checks | `V###` | Not started |

---

## Task Index

### Backend (`tasks/backend.md`) — 14 tasks

| ID | Story | Description |
| -- | ----- | ----------- |
| B001 | — | Update `Directory.Packages.props` — add/verify test package versions |
| B002 | — | Create `HiveSpace.Testing.Shared` project with fakes and builders |
| B003 | US1 | Create backend quality-gate runner `quality-gate.ps1` |
| B004 | US2 | Create `HiveSpace.IdentityService.Tests` — all handlers layer-organized |
| B006 | US2 | Create `HiveSpace.UserService.Tests` — all handlers layer-organized |
| B007 | US2 | Create `HiveSpace.CatalogService.Tests` — all handlers and domain tests layer-organized |
| B008 | US2 | Create `HiveSpace.OrderService.Tests` — all handlers, domain tests, and sagas layer-organized |
| B009 | US2 | Create additional tests in `HiveSpace.PaymentService.Tests` to reach 80% coverage target |
| B010 | US2 | Create `HiveSpace.MediaService.Tests` |
| B011 | US2 | Create `HiveSpace.NotificationService.Tests` |
| B012 | US1 | Update solution file to include all new test projects |
| B013 | — | Update `TESTING.md` developer TDD guide in `../hivespace.microservice` to reference 80% coverage target and B021 enforcement |
| B020 | US2 | PaymentService — add `GetTransactionHistory` handler and `Payment` domain tests |
| B021 | US2 | Add 80% line coverage threshold enforcement to `quality-gate.ps1` (parse Cobertura XML per service; fail gate below threshold) |

### Frontend (`tasks/frontend.md`) — 12 tasks

| ID | Story | Description |
| -- | ----- | ----------- |
| F001 | US1 | Update root `package.json` — add test and quality-gate scripts |
| F002 | US1 | Update root `turbo.json` — add `test` task |
| F003 | US1 | Create frontend quality-gate runner `scripts/quality-gate.mjs` |
| F004 | US1 | Confirm `packages/shared/package.json` has Jest test script; create shared package tests to reach 80% coverage |
| F005 | US1/US2/US3 | Confirm shared test utilities in `packages/shared/src/test-utils/` are present and export all required helpers |
| F006 | US2 | Confirm `apps/buyer/package.json` has Jest test script |
| F007 | US2 | Create buyer critical-path tests co-located in `apps/buyer/src/` — target 80% policy-scoped coverage |
| F008 | US2 | Confirm `apps/seller/package.json` has Jest test script |
| F009 | US2 | Create seller critical-path tests co-located in `apps/seller/src/` — target 80% policy-scoped coverage |
| F010 | US2 | Confirm `apps/admin/package.json` has Jest test script |
| F011 | US2 | Create admin critical-path tests co-located in `apps/admin/src/` — target 80% policy-scoped coverage |
| F012 | — | Update `TESTING.md` developer TDD guide in `../hivespace.web` to reference 80% coverage gate and testing-rules.md |

### Docs/Catalog (`tasks/docs-catalog.md`) — 2 tasks

| ID | Story | Description |
| -- | ----- | ----------- |
| D001 | US1 | Update `contracts/quality-gate-report.md` — verify and finalize schema |
| D002 | US1 | Update `ADR-0007` status to Accepted and add follow-up references |

### Verification (`tasks/verification.md`) — 13 tasks

| ID | Story | Description |
| -- | ----- | ----------- |
| V001 | US1 | Verify backend builds with all new test projects |
| V002 | US1/US2 | Run full backend test suite |
| V003 | US2 | Run service-specific backend tests independently |
| V004 | US1 | Verify backend quality-gate runner scope options |
| V005 | US1 | Verify frontend installs and test scripts exist |
| V006 | US2 | Run app-specific frontend tests independently |
| V007 | US1 | Verify lint and type-check baselines are not worsened |
| V008 | US1 | Verify frontend quality-gate runner scope options |
| V009 | US2 | Verify coverage map covers all critical journeys (FR-002 to FR-005) |
| V010 | US3 | Simulate a product failure and verify diagnostic output |
| V011 | US3 | Verify environment failure is distinct from product failure |
| V012 | US3 | Verify coverage gap visibility and accepted risk workflow |
| V013 | — | Verify spec artifacts and ADR are complete |

**Total: 41 tasks** (14 backend, 12 frontend, 2 docs/catalog, 13 verification) — B006–B009, B011 are comprehensive layer-organized tasks covering all handlers and domain tests; B020 adds remaining PaymentService coverage; B021 adds 80% threshold enforcement to quality-gate.ps1; B005 (ApiGateway.Tests) excluded

---

## Dependency Order

```
Phase 1 — Spec / Contract (run first, no source-repo dependencies)
  D001  Finalize quality-gate-report.md contract
  D002  Accept ADR-0007

Phase 2 — Backend foundations (parallel)
  B001  Update Directory.Packages.props
  B002  Create HiveSpace.Testing.Shared  ← B004–B011 all depend on this

Phase 3 — Backend service test projects (parallel after B001 + B002)
  B004  HiveSpace.IdentityService.Tests
  B006  HiveSpace.UserService.Tests
  B007  HiveSpace.CatalogService.Tests
  B008  HiveSpace.OrderService.Tests
  B009  HiveSpace.PaymentService.Tests (update)
  B010  HiveSpace.MediaService.Tests
  B011  HiveSpace.NotificationService.Tests

Phase 4 — Backend runner + solution + guide (after B004–B011)
  B003  quality-gate.ps1
  B012  Update solution file
  B013  TESTING.md developer guide (no dependencies; can run in parallel with B003/B012)
  B020  PaymentService additional handler + Payment domain tests (after B009)
  B021  Add 80% coverage threshold to quality-gate.ps1 (after B003; reads Cobertura XML output)

Phase 5 — Frontend foundations (parallel with Phase 2–4)
  F001  root package.json
  F002  turbo.json
  F004  packages/shared/package.json
  F005  packages/shared/src/test-utils/  ← F007, F009, F011 all depend on this

Phase 6 — Frontend app setup (parallel after F001 + F002 + F005)
  F006  apps/buyer/package.json + Jest config confirmation
  F008  apps/seller/package.json + Jest config confirmation
  F010  apps/admin/package.json + Jest config confirmation

Phase 7 — Frontend app tests (parallel after F006, F008, F010)
  F007  Buyer tests
  F009  Seller tests
  F011  Admin tests

Phase 8 — Frontend runner + guide (after F007 + F009 + F011)
  F003  scripts/quality-gate.mjs
  F012  TESTING.md developer guide (no dependencies; can run in parallel with F003)

Phase 9 — Verification (after all implementation phases)
  V001–V004  Backend build, test, runner
  V005–V008  Frontend install, test, runner
  V009        Coverage map review
  V010–V012   Diagnostic, environment, and accepted-risk scenarios
  V013        Spec artifact check
```

---

## Story Traceability

| Story | Priority | Detailed task IDs | Independent acceptance |
| ----- | -------- | ----------------- | ---------------------- |
| US1 — Verify Changes Before Merge | P1 | B001–B003, B012, D001–D002, F001–F003, V001, V002, V004, V005, V007, V008, V013 | Run `.\quality-gate.ps1 -Scope backend:PaymentService` and `node scripts/quality-gate.mjs --scope frontend:buyer`; each produces a clear pass/fail result matching the contract schema |
| US2 — Protect Critical User Journeys | P2 | B004, B006–B011, B020–B021, F005–F011, V003, V006, V009 | Review V009 coverage map; every FR-002 to FR-005 journey maps to at least one passing test or a documented gap |
| US3 — Diagnose Failures And Gaps | P3 | F005, V010–V012 | Introduce a deliberate failure; confirm quality gate report names journey, expected behavior, and blocking level; environment failures classified separately from product failures |

---

## MVP Scope (minimal path to US1 pass)

To satisfy US1 (quality gate produces a clear pass/fail for a typical change) with the smallest task set:

1. **D001** Finalize contract
2. **B001** Package versions
3. **B002** Shared test infrastructure
4. **B009** Extend PaymentService.Tests (already exists — lowest friction backend service)
5. **B003** Backend quality gate runner
6. **B012** Solution file update
7. **F001, F002** Root workspace test scripts
8. **F004, F005** Shared package test utilities
9. **F006, F007** Buyer tests (highest-traffic critical path)
10. **F003** Frontend quality gate runner
11. **V001, V002, V004, V005, V008** Build and runner verification

All other B, F, D, V tasks expand coverage to US2 and US3 and are required for full critical-path gate readiness.

---

## Completion Checklist

- [ ] All 4 detailed task files exist under `tasks/`
- [ ] All 41 task IDs are unique across all files (no duplicate B/F/D/V numbers)
- [ ] Every implementation task (B, F, D) includes file path, exact change detail, forbidden behavior, and acceptance check
- [ ] `tasks.md` dependency order matches detailed task dependencies
- [ ] Verification tasks cover: build, full test run, service-specific tests, runner scope options, coverage map, diagnostic output, and spec artifact check
- [ ] `contracts/quality-gate-report.md` is finalized and referenced in B003, F003
- [ ] `shared/api-catalog.md` and `shared/event-catalog.md` unchanged (no new entries added by this feature)
- [ ] No tasks reference `../hivespace.config` for implementation work
