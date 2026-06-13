# Verification Tasks: System Testing Quality Gate

Run these checks after completing backend, frontend, and docs/catalog tasks. Each section must pass before moving to the next group.

---

## Backend Verification

### Verify

- [ ] V001 [US1] Verify backend builds with all new test projects
  - Command: `dotnet restore && dotnet build` from `../hivespace.microservice`
  - Expected: zero build errors; all new test projects compiled; no `Version=` attributes present in any new `.csproj` (search with `grep -r 'Version=' tests/`)
  - Do not proceed to V002 if this fails
  - Acceptance: `dotnet build` exits with code 0; no `Version=` found in new project files

- [ ] V002 [US1][US2] Run full backend test suite
  - Command: `dotnet test` from `../hivespace.microservice`
  - Expected: all test projects discovered and pass; test output lists `HiveSpace.Testing.Shared`, all 7 service test projects; no unexpected skips without documented reason
  - Acceptance: `dotnet test` exits with code 0; test count is non-zero across all projects

- [ ] V003 [US2] Run service-specific backend tests independently
  - Run each command independently from `../hivespace.microservice`:
    - `dotnet test tests/HiveSpace.IdentityService.Tests`
    - `dotnet test tests/HiveSpace.UserService.Tests`
    - `dotnet test tests/HiveSpace.CatalogService.Tests`
    - `dotnet test tests/HiveSpace.OrderService.Tests`
    - `dotnet test tests/HiveSpace.PaymentService.Tests`
    - `dotnet test tests/HiveSpace.MediaService.Tests`
    - `dotnet test tests/HiveSpace.NotificationService.Tests`
  - Expected: each command succeeds in isolation; no command requires other service test projects to be present
  - Acceptance: all 7 commands exit with code 0 independently; impact-based gating is viable

- [ ] V004 [US1] Verify backend quality-gate runner scope options
  - Commands from `../hivespace.microservice`:
    - `.\quality-gate.ps1 -Scope docs-only`
    - `.\quality-gate.ps1 -Scope backend:PaymentService`
    - `.\quality-gate.ps1 -Scope release`
  - Expected for `docs-only`: no `dotnet test` invoked; output is valid JSON with all `status: "not_applicable"` results
  - Expected for `backend:PaymentService`: only `HiveSpace.PaymentService.Tests` invoked; output is valid JSON with one `checkResult` for PaymentService
  - Expected for `release`: all test projects invoked; output is valid JSON; `gate.result` is `pass` when tests pass
  - Acceptance: all three scope options produce JSON matching `contracts/quality-gate-report.md`; `docs-only` does not invoke `dotnet test`

---

## Frontend Verification

### Verify

- [ ] V005 [US1] Verify frontend installs and test scripts exist
  - Commands from `../hivespace.web`:
    - `pnpm install`
    - `pnpm test`
  - Expected: install completes without resolution errors; Turbo fans out `test` task to buyer, seller, admin, and shared package workspaces; all tests pass
  - Acceptance: `pnpm test` exits with code 0; Turbo output shows test tasks running for all four workspaces

- [ ] V006 [US2] Run app-specific frontend tests independently
  - Run each command independently from `../hivespace.web`:
    - `pnpm --filter buyer test`
    - `pnpm --filter seller test`
    - `pnpm --filter admin test`
    - `pnpm --filter @hivespace/shared test`
  - Expected: each runs in isolation without requiring other apps; each test file is discovered by vitest
  - Acceptance: all four commands exit with code 0

- [ ] V007 [US1] Verify lint and type-check baselines are not worsened
  - Commands from `../hivespace.web`:
    - `pnpm lint`
    - `pnpm type-check`
  - Expected: error counts must not exceed documented pre-existing baseline (known buyer/seller/admin errors remain; no new errors introduced by test files or vitest configs)
  - Check: run before and after adding test files to confirm no increase in error count
  - Acceptance: no new lint or type-check failures are attributable to changes from this feature; existing baseline errors remain unchanged

- [ ] V008 [US1] Verify frontend quality-gate runner scope options
  - Commands from `../hivespace.web`:
    - `node scripts/quality-gate.mjs --scope docs-only`
    - `node scripts/quality-gate.mjs --scope frontend:buyer`
    - `node scripts/quality-gate.mjs --scope release`
  - Expected for `docs-only`: no `pnpm test` invoked; output is valid JSON with all `status: "not_applicable"` results
  - Expected for `frontend:buyer`: only buyer workspace tests invoked; output has one `checkResult` for buyer
  - Expected for `release`: all workspaces tested; `gate.result` is `pass` when tests pass
  - Acceptance: all three scopes produce JSON matching `contracts/quality-gate-report.md`; `docs-only` does not invoke `pnpm test`

---

## Quality Gate Coverage

### Verify

- [ ] V009 [US2] Verify coverage map covers all critical journeys from FR-002 to FR-005
  - For each required journey, confirm at least one task ID maps to a repeatable check or a documented coverage gap:
  
  | Journey | FR | Backend task | Frontend task | Gap? |
  | ------- | -- | ------------ | ------------- | ---- |
  | Buyer authentication/session | FR-002 | B004 (SessionTests) | F007 (auth.test.ts) | — |
  | Storefront catalog discovery | FR-002 | B007 (StorefrontDiscoveryTests) | F007 (catalog.test.ts) | — |
  | Cart updates | FR-002 | B008 (CartTests) | F007 (cart.test.ts) | — |
  | Checkout initiation | FR-002 | B008 (CheckoutTests) | F007 (checkout.test.ts) | — |
  | Payment result handling | FR-002 | B009 (PaymentResultStateTests) | F007 (payment-result.test.ts) | — |
  | Buyer order visibility | FR-002 | B008 (BuyerOrderTests) | F007 (orders.test.ts) | — |
  | Notifications | FR-002 | B011 (DeliveryVisibilityTests) | F007 (notifications.test.ts) | — |
  | Seller store access | FR-003 | B006 (StoreOnboardingTests) | F009 (access.test.ts) | — |
  | Seller product management | FR-003 | B007 (SellerProductManagementTests) | F009 (products.test.ts) | — |
  | Seller coupon/promotion | FR-003 | B008 (CouponTests) | F009 (promotions.test.ts) | Partial if surface absent |
  | Seller order review/confirm/reject | FR-003 | B008 (SellerOrderTests) | F009 (orders.test.ts) | — |
  | Admin account visibility | FR-004 | B004 (AdminAccountTests) | F011 (accounts.test.ts) | — |
  | Admin identity-affecting actions | FR-004 | B004 (AdminAccountTests) | F011 (accounts.test.ts) | — |
  | Admin profile/store review | FR-004 | B006 (ProfileTests) | F011 (profile-settings.test.ts) | — |
  | Public route ownership | FR-005 | D001 (api-catalog verification) | — | ApiGateway excluded; coverage via contract doc |
  | Authorization expectations | FR-005 | B004 (session tests), B006–B011 (handler auth) | F007/F009/F011 (access) | — |
  | Cross-service event handling | FR-005 | B007–B011 (InMemoryMessageCapture) | — | — |
  | Checkout/fulfillment workflow | FR-005 | B008 (CheckoutSagaTests, FulfillmentSagaTests) | — | — |
  | Payment idempotency | FR-005 | B009 (VNPayReturnIdempotencyTests, VNPayWebhookIdempotencyTests) | — | — |
  | Media upload confirmation | FR-005 | B010 (UploadConfirmationTests) | — | — |
  | Notification delivery visibility | FR-005 | B011 (DeliveryVisibilityTests) | F007 (notifications.test.ts) | — |
  
  - Any journey marked `Gap?` must produce an explicit CoverageGap entry in quality gate output or be accepted risk
  - Acceptance: every journey in the table has at least one task ID; no required FR-002 to FR-005 journey is uncovered without a documented gap

---

## Diagnostic and Safety Verification

### Verify

- [ ] V010 [US3] Simulate a product failure and verify diagnostic output
  - Process:
    1. In `HiveSpace.OrderService.Tests/CheckoutSagaTests`, temporarily change one assertion to force failure (e.g. assert wrong status)
    2. Run `.\quality-gate.ps1 -Scope backend:OrderService` from `../hivespace.microservice`
    3. Observe output; then revert the intentional failure
  - Expected: output JSON has `checkResult.status: "fail"`, `failureCategory: "product_behavior"`, `blockingDecision: "merge_blocking"`, and a `summary` naming the failing test and expected vs actual values; `gate.result` is `"fail"`
  - Acceptance: output identifies the journey (`cart-and-checkout`), failure category (`product_behavior`), and blocking decision (`merge_blocking`); revert confirmed clean after test

- [ ] V011 [US3] Verify environment failure is distinct from product failure
  - Process:
    1. Temporarily configure a test in `HiveSpace.UserService.Tests` to attempt a real SQL connection that is not available
    2. Run `dotnet test tests/HiveSpace.UserService.Tests`
    3. Observe that the quality gate runner (B003) reports `failureCategory: "environment_readiness"` not `product_behavior`
    4. Revert the configuration change
  - Expected: test output or quality gate output distinguishes "could not connect to database" (environment) from a behavioral assertion failure (product)
  - Note: if the test is designed with fakes correctly (B002 `BlobStorageFake`, `EmailDeliveryFake`, etc.), most environment failures can be detected by whether the error is a connection error vs an assertion error; document the strategy in runner comments if needed
  - Acceptance: environment-class failures are classified as `environment_readiness`; product-class failures are classified as `product_behavior`; both are distinct in report output

- [ ] V012 [US3] Verify coverage gap visibility and accepted risk workflow
  - Process:
    1. Run quality gate for a scope where a known CoverageGap exists (e.g. seller coupon/promotion surface if absent)
    2. Confirm `coverageGaps` array in output is non-empty with `journey`, `reason`, `riskLevel`, `resolution`
    3. Add an `AcceptedRisk` entry manually to the runner or config; re-run and confirm `acceptedRisks` array populated
  - Expected: `coverageGaps` is visible and non-empty when gaps exist; `acceptedRisks` entry includes `scope`, `approvingRole`, `reason`, `expiresWhen`; gap entry maps to a `checkResult` with `status: "accepted_risk"` and `blockingDecision: "blocked_until_risk_accepted"`
  - Acceptance: quality gate report schema fully populated for gap + accepted risk scenario; reviewer can identify approving role and expiry without additional context

---

## Spec Artifact Verification

### Verify

- [ ] V013 Verify spec artifacts and ADR are complete
  - Check: `specs/0008-system-testing-quality-gate/contracts/quality-gate-report.md` exists and contains valid JSON samples
  - Check: `architecture/decisions/ADR-0007-system-testing-quality-gate.md` has `Status: Accepted`
  - Check: `shared/api-catalog.md` and `shared/event-catalog.md` are unchanged from before this feature (no new rows added)
  - Check: no service docs under `services/` were modified for this feature
  - Acceptance: all checks pass; catalogs unchanged; ADR accepted
