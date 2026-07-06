# Verification Tasks

## Backend

### Verify

- [ ] V001 Verify backend service tests, builds, and quality gates for feature `0010`
  - File: `../hivespace.microservice/tests/HiveSpace.UserService.Tests/*`; `../hivespace.microservice/tests/HiveSpace.CatalogService.Tests/*`; `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/*`; `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/*`
  - Run: `dotnet build`, `.\quality-gate.ps1 -Scope backend:UserService`, `.\quality-gate.ps1 -Scope backend:CatalogService`, `.\quality-gate.ps1 -Scope backend:OrderService`, and `.\quality-gate.ps1 -Scope backend:PaymentService`
  - Confirm the new measured Domain/Application paths stay at or above the 80% coverage target; add missing tests before completion if any affected service falls below threshold
  - Acceptance: backend build is green and each affected service passes its scoped quality gate

## Frontend

### Verify

- [ ] V002 Verify shared/admin/seller/buyer tests, type-checks, and coverage for feature `0010`
  - File: `../hivespace.web/packages/shared/src/**/*.test.ts`; `../hivespace.web/apps/admin/src/**/*.test.ts`; `../hivespace.web/apps/seller/src/**/*.test.ts`; `../hivespace.web/apps/buyer/src/**/*.test.ts`
  - Run: `pnpm --filter @hivespace/shared test`, `pnpm --filter @hivespace/admin type-check`, `pnpm --filter @hivespace/seller type-check`, `pnpm --filter @hivespace/buyer type-check`, `.\coverage.ps1 -Workspace shared`, `.\coverage.ps1 -Workspace admin`, `.\coverage.ps1 -Workspace seller`, and `.\coverage.ps1 -Workspace buyer`
  - Treat the known admin/seller baseline issues from `../hivespace.web/AGENTS.md` as baseline only; fix any new errors introduced by this feature
  - Acceptance: the affected frontend workspaces pass their required checks and any configuration-scoped coverage drop below 80% is closed before completion

## Cross-Repo Consistency

### Verify

- [ ] V003 Verify hard-coded currency fallbacks and page-local money formatters are removed from scoped surfaces
  - File: `../hivespace.microservice/src/**/*.cs`; `../hivespace.web/apps/**/src/**/*.{ts,vue}`; `../hivespace.web/packages/shared/src/**/*.{ts,vue}`
  - Search for remaining feature-scope regressions such as `FromVND`, hard-coded `"VND"`, `vi-VN`-only money formatting, and local `formatMoney` helpers in the pages/components listed in `frontend.md`, including shipped admin money-bearing pages
  - Ignore legitimate non-money localization usage outside the feature scope; focus on the paths called out by the plan and task files
  - Acceptance: the scoped code paths no longer rely on silent `VND` defaults or page-local money formatter logic

## Manual Validation

### Verify

- [ ] V004 [US1] Verify admin currency configuration propagation across backend and authenticated clients
  - File: `specs/0010-standardize-money-handling/quickstart.md`
  - User-owned E2E: run the admin configuration flow to enable `USD` and `EUR`, save a new default currency, and confirm seller/buyer authenticated config reads reflect the updated enabled/default set without manual data fixes
  - Confirm the negative path where disabling the current default without choosing a replacement is rejected visibly
  - Confirm the negative path where a dependent service has not yet applied the required policy projection and therefore rejects the write explicitly instead of accepting a fallback currency
  - Acceptance: user confirms the admin workflow behaves as specified and downstream authenticated clients observe the saved policy

- [ ] V005 [US2] Verify seller and buyer money journeys end to end with enabled, disabled, and mixed currencies
  - File: `specs/0010-standardize-money-handling/spec.md`
  - User-owned E2E: create or edit a coupon and product in `USD` or `EUR`, view the resulting values in seller and buyer surfaces, and attempt a mixed-currency cart/coupon or disabled-currency submission to confirm the blocking behavior
  - Confirm shared input/display behavior: `USD`/`EUR` use major-unit entry and display, `VND` uses whole-unit display/input, and invalid historical values render placeholders instead of guessed symbols
  - Acceptance: user confirms the end-to-end commerce flows satisfy the US2 and US3 acceptance scenarios without inconsistent money rendering
