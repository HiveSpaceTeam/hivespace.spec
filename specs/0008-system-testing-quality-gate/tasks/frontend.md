# Frontend Tasks: System Testing Quality Gate

**Repo**: `../hivespace.web`
**Prerequisites**: Read `../hivespace.web/AGENTS.md` and `CLAUDE.md` before starting. Preserve all pnpm/Turbo workspace conventions. Do not broaden existing lint/type-check baseline failures.

---

## Root Workspace Setup

### Update

- [ ] F001 [US1] Update root `package.json` — add test and quality-gate scripts
  - File: `../hivespace.web/package.json`
  - Add to `scripts`: `"test": "turbo run test"` (Turbo fan-out across all workspaces)
  - Add to `scripts`: `"quality-gate": "node scripts/quality-gate.mjs"`
  - Add to `devDependencies` (root-level, hoisted): `"vitest": "^2.x"` (use the same major version across all workspaces)
  - Preserve all existing script entries (`build`, `lint`, `type-check`, `dev`, etc.); do not rename or remove
  - Acceptance: `pnpm test` invokes Turbo `test` task across all workspaces that define a `test` script; `pnpm quality-gate` resolves without "command not found"

- [ ] F002 [US1] Update root `turbo.json` — add `test` task
  - File: `../hivespace.web/turbo.json`
  - Add task entry:
    ```json
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"],
      "cache": true
    }
    ```
  - Do not modify or remove existing task entries (`build`, `lint`, `type-check`)
  - Acceptance: `pnpm turbo run test` fans out to all workspaces that define a `test` script; `turbo.json` is valid JSON

### Create

- [ ] F003 [US1] Create frontend quality-gate runner `scripts/quality-gate.mjs`
  - File: `../hivespace.web/scripts/quality-gate.mjs`
  - Accept `--scope` CLI flag; allowed values: `docs-only`, `frontend:buyer`, `frontend:seller`, `frontend:admin`, `shared`, `release`
  - For `docs-only`: emit CheckResult with `status: "not_applicable"` for all runtime checks; do not invoke `pnpm test`
  - For `frontend:<appName>`: run shared baseline then `pnpm --filter <appName> test`
  - For `shared`: run `pnpm --filter @hivespace/shared test`
  - For `release`: run `pnpm test` (all workspaces)
  - Rerun policy: on non-zero exit, re-invoke once; if second exit also non-zero with different output, emit `status: "unstable_check"`; if both fail consistently, emit `status: "fail"`
  - Emit JSON output to stdout matching `contracts/quality-gate-report.md`: one `gate` object, one `checkResult` per workspace test run, `coverageGaps: []`, `acceptedRisks: []`
  - Map test pass to `status: "pass", blockingDecision: "none"`; failure to `status: "fail", failureCategory: "product_behavior", blockingDecision: "merge_blocking"`
  - Do not make live API calls; do not use real credentials; use only `pnpm` child process invocations
  - Acceptance: `node scripts/quality-gate.mjs --scope release` produces valid JSON; `node scripts/quality-gate.mjs --scope frontend:buyer` invokes only buyer workspace tests; `node scripts/quality-gate.mjs --scope docs-only` returns `not_applicable` without running tests

---

## Shared Package Test Utilities

### Update

- [ ] F004 [US1] Verify `packages/shared/package.json` — confirm test script and Jest config are present
  - File: `../hivespace.web/packages/shared/package.json`
  - The project uses **Jest** (via `jest.base.cjs` factory), not Vitest. `packages/shared/jest.config.cjs` already exists.
  - Verify `"test": "jest"` (or `"test": "jest --passWithNoTests"`) exists in `scripts`; add it if absent
  - Do not add a vitest config or vitest dependency
  - Acceptance: `pnpm --filter @hivespace/shared test` runs without "test script not found" errors

### Create

- [ ] F005 [US1][US2][US3] Verify shared test utilities in `packages/shared/src/test-utils/`
  - **Already exists** — `packages/shared/src/test-utils/` is present with all required files. Verify each file exports what this task specifies; add missing exports if any are absent.
  - File set (verify, do not recreate):
    - `../hivespace.web/packages/shared/src/test-utils/auth.ts`
    - `../hivespace.web/packages/shared/src/test-utils/api.ts`
    - `../hivespace.web/packages/shared/src/test-utils/notification.ts`
    - `../hivespace.web/packages/shared/src/test-utils/media.ts`
    - `../hivespace.web/packages/shared/src/test-utils/modal.ts`
    - `../hivespace.web/packages/shared/src/test-utils/i18n.ts`
    - `../hivespace.web/packages/shared/src/test-utils/index.ts`
  - `auth.ts`:
    - Export `createFakeAuthUser(overrides?: Partial<AppUser>): AppUser` — returns typed user with configurable `id`, `role` (`buyer` | `seller` | `admin`), `storeId?`
    - Export `createFakeAuthState(role: 'buyer' | 'seller' | 'admin')` — returns Pinia-compatible `auth` store state object; does not call OIDC provider
  - `api.ts`:
    - Export `createMockAxios()` — returns axios instance with base URL pointing to `localhost` test placeholder and interceptors stripped; all requests return `undefined` unless stubbed
    - Export `stubApiResponse(instance: AxiosInstance, method: string, path: string, data: unknown, status = 200): void` — queues a response for the matched path+method; used in `beforeEach` setup
  - `notification.ts`:
    - Export `createFakeSignalRHub()` — returns object with `invocations: Array<{method: string; args: unknown[]}>` and `emit(method: string, args: unknown): void` trigger; does not connect to SignalR endpoint
  - `media.ts`:
    - Export `createFakePresignResponse(overrides?: object)` — returns `{url: 'https://fake-presign.test/upload', uploadRef: 'fake-ref-001'}`
    - Export `simulateUploadConfirm(uploadRef: string)` — returns `{confirmed: true, mediaId: uploadRef}`
  - `modal.ts`:
    - Export `openModal(component: Component, props?: object)` — mounts component with `vue-final-modal` context for testing modal visibility and slot content
    - Export `closeAllModals()` — resets modal state after test
  - `i18n.ts`:
    - Export `createTestI18n(messages?: Record<string, unknown>)` — returns `vue-i18n` instance pre-loaded with English locale keys from `@hivespace/shared`; accepts optional override messages
  - `index.ts`: re-exports all named exports from the above files
  - Do not export or modify any runtime product code from `@hivespace/shared`; test-utils must not change public runtime API or break tree-shaking for production builds (add `"sideEffects": false` to `package.json` if not already set)
  - Do not import from `apps/buyer`, `apps/seller`, or `apps/admin`
  - Acceptance: each utility can be imported in an app test file with `import { createFakeAuthUser } from '@hivespace/shared/test-utils'`; `pnpm --filter @hivespace/shared test` passes

---

## Buyer App Tests

### Update

- [ ] F006 [US2] Verify `apps/buyer/package.json` — confirm test script and Jest config are present
  - File: `../hivespace.web/apps/buyer/package.json`
  - The project uses **Jest** (via `jest.base.cjs` factory). `apps/buyer/jest.config.cjs` already exists.
  - Verify `"test": "jest"` (or `"test": "jest --passWithNoTests"`) exists in `scripts`; add it if absent
  - Do not create a vitest.config.ts; do not add vitest dependency
  - Acceptance: `pnpm --filter buyer test` runs without "test script not found" or Jest config error

### Create

- [ ] F007 [US2] Verify and extend buyer critical-path tests co-located in `apps/buyer/src/`
  - Test files live next to their source files (co-location). All apps use `src/pages/<SubFolder>/` — **not** `src/views/`.
  - TESTING.md coverage policy includes `apps/*/src/pages/**`, `apps/*/src/stores/**`, `apps/*/src/composables/**`, `apps/*/src/router/**`.
  - **Most files already exist** — verify each has adequate coverage per the test cases below; add missing test cases rather than recreating files.
  - Store tests already exist (verify and extend):
    - `../hivespace.web/apps/buyer/src/stores/auth.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/stores/cart.store.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/stores/checkout.store.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/stores/orders.store.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/stores/notification.store.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/router/index.test.ts` ← already exists
  - Page tests already exist (verify and extend):
    - `../hivespace.web/apps/buyer/src/pages/Cart/CartPage.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/pages/Checkout/CheckoutPage.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/pages/Payment/PaymentResultPage.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/pages/Notifications/NotificationsPage.test.ts` ← already exists
    - `../hivespace.web/apps/buyer/src/pages/Profile/OrdersPage.test.ts` ← already exists (covers order list)
    - `../hivespace.web/apps/buyer/src/pages/Product/ProductDetailPage.test.ts` ← already exists
  - Create if absent:
    - `../hivespace.web/apps/buyer/src/pages/Home/HomePage.test.ts` — verify this exists; if missing, add storefront catalog/product listing test
  - `stores/auth.test.ts` (store actions/state only, no component rendering):
    - `initiateLogin_DispatchesOidcRedirect`: login action triggers OIDC redirect
    - `processCallback_StoresAppUserInAuthStore`: successful callback stores `AppUser` in Pinia auth store
    - `routeGuard_WithLoggedInUser_AllowsAccess`: store guard returns true for authenticated user
    - `routeGuard_WithUnauthenticatedUser_RedirectsToLogin`: guard returns redirect for unauthenticated user
    - `logout_ClearsStoredUserAndSession`: logout action clears stored user
  - `stores/cart.test.ts` (store actions/state only):
    - `addItem_IncrementsCartCount`: item added; count increases
    - `updateQuantity_ChangesLineTotal`: quantity change updates subtotal
    - `removeItem_LeavesRemainingItems`: removed item absent; others intact
    - `state_WithNoItems_IsEmptyState`: empty state when no items in store
  - `stores/checkout.test.ts` (store actions/state only):
    - `initiateCheckout_CallsCheckoutEndpointWithCartItems`: checkout action triggers endpoint stub with correct payload
    - `onSuccess_NavigatesToPaymentStep`: successful checkout transitions to payment route
  - `stores/orders.test.ts` (store actions/state only):
    - `fetchOrders_LoadsFromApi`: fetch action populates orders state from stub
    - `fetchOrderDetail_ReturnsLineItemsAndStatus`: detail query returns expected fields
  - `stores/notifications.test.ts` (store actions/state only):
    - `fetchNotifications_LoadsFromApi`: fetch populates notification list from stub
    - `markRead_DecrementsUnreadCount`: read action decrements unread badge count
    - `realtimeNotification_UpdatesStore`: `createFakeSignalRHub().emit('ReceiveNotification', ...)` causes store to refresh
  - `views/CatalogPage.test.ts` (component rendering with store wired up):
    - `renders_ProductListFromStubbedApiResponse`: product list component renders items
    - `searchInput_AppliesFilterViaApi`: search term passed to API stub; filtered results shown
    - `categoryFilter_ScopesResults`: category param applied; filtered products shown
    - `productWithActiveFalse_ShowsUnavailableState`: inactive product shows unavailable state
    - `productDetailPage_Renders_Title_Price_AndVariants`: detail view shows expected fields from stubbed product
  - `views/CartPage.test.ts`:
    - `renders_CartItemsFromStore`: cart items visible from wired store
    - `renders_EmptyState_WhenCartHasNoItems`: empty state when store cart is empty
    - `renders_CorrectSubtotalAfterQuantityChange`: subtotal reflects quantity in store
  - `views/CheckoutPage.test.ts`:
    - `renders_CartItemsInOrderSummary`: checkout page shows items from cart store
    - `invalidForm_ShowsValidationMessages_AsI18nKeys`: missing required field shows i18n error key
  - `views/PaymentResultPage.test.ts`:
    - `renders_OrderConfirmationMessage_OnSuccessStatus`: success status shows confirmation
    - `renders_RetryOrContactOption_OnFailureStatus`: failure shows retry action
    - `renders_WaitingState_OnPendingStatus`: pending shows processing message
    - `readsResultFrom_RouteParamsOrQueryString`: result derived from URL; no hardcoded status
  - `views/OrderListPage.test.ts`:
    - `renders_BuyerOrdersFromStore`: order list renders items
    - `renders_OrderDetail_WithLineItemsAndStatus`: detail view shows line items and status
    - `renders_EmptyState_WhenOrderListIsEmpty`: empty state for zero orders
  - `views/NotificationsPage.test.ts`:
    - `renders_UnreadBadgeCount_FromStore`: badge reflects store unread count
    - `renders_NotificationListFromStore`: notification items shown from store
  - Use `createFakeAuthUser`, `createMockAxios`, `stubApiResponse`, `createFakeSignalRHub`, `createTestI18n` from `@hivespace/shared/test-utils`
  - All text assertions use i18n key lookups, not hardcoded display strings
  - Do not import from `apps/seller`, `apps/admin`, or cross-app paths
  - Acceptance: `pnpm --filter buyer test` exits with code 0

---

## Seller App Tests

### Update

- [ ] F008 [US2] Verify `apps/seller/package.json` — confirm test script and Jest config are present
  - File: `../hivespace.web/apps/seller/package.json`
  - The project uses **Jest** (via `jest.base.cjs` factory). `apps/seller/jest.config.cjs` already exists.
  - Verify `"test": "jest"` (or `"test": "jest --passWithNoTests"`) exists in `scripts`; add it if absent
  - Do not create a vitest.config.ts; do not add vitest dependency
  - Acceptance: `pnpm --filter seller test` runs without "test script not found" or Jest config error

### Create

- [ ] F009 [US2] Verify and extend seller critical-path tests co-located in `apps/seller/src/`
  - Test files live next to their source files (co-location). All apps use `src/pages/<SubFolder>/` — **not** `src/views/`.
  - TESTING.md coverage policy includes `apps/*/src/pages/**`, `apps/*/src/stores/**`, `apps/*/src/composables/**`, `apps/*/src/router/**`.
  - **Most files already exist** — verify each has adequate coverage per the test cases below; add missing test cases rather than recreating files.
  - Store tests already exist (verify and extend):
    - `../hivespace.web/apps/seller/src/stores/auth.test.ts` ← already exists
    - `../hivespace.web/apps/seller/src/stores/product.test.ts` ← already exists
    - `../hivespace.web/apps/seller/src/stores/order.test.ts` ← already exists
  - Page tests already exist (verify and extend):
    - `../hivespace.web/apps/seller/src/pages/DashboardPage.test.ts` ← already exists
    - `../hivespace.web/apps/seller/src/pages/Products/ProductListPage.test.ts` ← already exists
    - `../hivespace.web/apps/seller/src/pages/Products/UpsertProductPage.test.ts` ← already exists (covers create and edit)
    - `../hivespace.web/apps/seller/src/pages/Marketing/CouponListPage.test.ts` ← already exists
    - `../hivespace.web/apps/seller/src/pages/Orders/OrderManagementPage.test.ts` ← already exists (covers list and detail)
  - `stores/auth.test.ts` (store actions/state only):
    - `sellerRole_WithCreateFakeAuthUser_GrantsDashboardAccess`: `createFakeAuthUser({role: 'seller'})` populates seller role in auth store
    - `nonSellerRole_IsRedirectedByRouteGuard`: route guard with buyer role returns redirect from seller routes
    - `storeContext_LoadsOnLogin`: store ID claim populates seller store context in Pinia store
  - `stores/product.test.ts` (store actions/state only):
    - `fetchProducts_LoadsFromSellerScopedApi`: fetch populates product list from seller-scoped stub
    - `createProduct_UpdatesStoreState`: create action adds item to product list state
    - `updateProduct_UpdatesStoreState`: update action reflects changed fields
    - `deactivateProduct_UpdatesStatusInStore`: deactivate action marks product as inactive in state
  - `stores/order.test.ts` (store actions/state only):
    - `fetchOrders_ScopedToSellerStoreId`: fetch action scopes request to seller's store ID
    - `confirmOrder_TransitionsStatusInStore`: confirm action updates order status in state
    - `rejectOrder_StoresReasonAndTransitionsStatus`: reject action stores reason and updates status
  - `pages/DashboardPage.test.ts`:
    - `renders_WithSellerRole`: seller dashboard renders for authenticated seller user
    - `redirects_ForNonSellerUser`: non-seller user is redirected away from seller routes
  - `pages/Products/ProductListPage.test.ts`:
    - `renders_ProductListFromStore`: product list renders items from store
    - `unauthorizedUser_SeesErrorOnProductMutation`: non-seller user sees 403 error on product change attempt
  - `pages/Products/UpsertProductPage.test.ts`:
    - `formSubmit_CallsCreateEndpointWithCorrectPayload`: form submission triggers create endpoint stub with expected fields
    - `formSubmit_CallsUpdateEndpointAndReflectsNewFields`: edit submit triggers update endpoint; updated fields reflected
  - `pages/Marketing/CouponListPage.test.ts`:
    - `renders_CouponListFromApi`: coupon items rendered from stubbed response
    - `createCouponForm_SubmitsToApi`: coupon creation triggers API stub with correct fields
    - `expiredCoupon_ShowsExpiredBadge`: coupon with past `expiresAt` rendered with expired indicator
  - `pages/Orders/OrderManagementPage.test.ts`:
    - `renders_SellerOrders_ScopedToOwnStore`: orders rendered from stub scoped to seller's store ID
    - `renders_BuyerInfoAndLineItems`: detail view renders buyer reference and line items
    - `confirmOrder_UpdatesStatusDisplay`: confirm action triggers confirm endpoint; status updated in view
    - `rejectOrder_WithReason_UpdatesStatusDisplay`: reject action posts reason; status updated in view
  - Use `createFakeAuthUser({role: 'seller'})`, `createMockAxios`, `stubApiResponse`, `createTestI18n`
  - Do not import from `apps/buyer` or `apps/admin`
  - Acceptance: `pnpm --filter seller test` exits with code 0

---

## Admin App Tests

### Update

- [ ] F010 [US2] Verify `apps/admin/package.json` — confirm test script and Jest config are present
  - File: `../hivespace.web/apps/admin/package.json`
  - The project uses **Jest** (via `jest.base.cjs` factory). `apps/admin/jest.config.cjs` already exists.
  - Verify `"test": "jest"` (or `"test": "jest --passWithNoTests"`) exists in `scripts`; add it if absent
  - Do not create a vitest.config.ts; do not add vitest dependency
  - Acceptance: `pnpm --filter admin test` runs without "test script not found" or Jest config error

### Create

- [ ] F011 [US2] Verify and extend admin critical-path tests co-located in `apps/admin/src/`
  - Test files live next to their source files (co-location). All apps use `src/pages/<SubFolder>/` — **not** `src/views/`.
  - TESTING.md coverage policy includes `apps/*/src/pages/**`, `apps/*/src/stores/**`, `apps/*/src/composables/**`, `apps/*/src/router/**`.
  - **All expected files already exist** — verify each has adequate coverage per the test cases below; add missing test cases rather than recreating files.
  - Store tests already exist (verify and extend):
    - `../hivespace.web/apps/admin/src/stores/auth.test.ts` ← already exists
    - `../hivespace.web/apps/admin/src/stores/accounts.test.ts` ← already exists
    - `../hivespace.web/apps/admin/src/stores/notifications.test.ts` ← already exists
  - Page tests already exist (verify and extend):
    - `../hivespace.web/apps/admin/src/pages/DashboardPage.test.ts` ← already exists
    - `../hivespace.web/apps/admin/src/pages/Accounts/AccountsPage.test.ts` ← already exists (covers account list)
    - `../hivespace.web/apps/admin/src/pages/Accounts/AccountDetailPage.test.ts` ← already exists
    - `../hivespace.web/apps/admin/src/pages/Notifications/NotificationsPage.test.ts` ← already exists
    - `../hivespace.web/apps/admin/src/pages/Profile/ProfilePage.test.ts` ← already exists
    - `../hivespace.web/apps/admin/src/pages/Settings/SettingsPage.test.ts` ← already exists
  - `stores/auth.test.ts` (store actions/state only):
    - `adminRole_WithCreateFakeAuthUser_GrantsDashboardAccess`: `createFakeAuthUser({role: 'admin'})` populates admin role in auth store
    - `nonAdminRole_IsRedirectedByRouteGuard`: route guard with seller or buyer role returns redirect from admin routes
  - `stores/accounts.test.ts` (store actions/state only):
    - `fetchAccounts_LoadsFromApi`: fetch populates account list from stub
    - `suspendAccount_UpdatesStatusInStore`: suspend action marks account as suspended in state
    - `activateAccount_UpdatesStatusInStore`: activate action marks account as active in state
  - `stores/notifications.test.ts` (store actions/state only):
    - `fetchNotifications_LoadsForAdminUser`: fetch populates notification list for admin role
    - `realtimeNotification_UpdatesStore`: `createFakeSignalRHub().emit` causes store to refresh
  - `pages/Accounts/AccountsPage.test.ts`:
    - `renders_AccountListFromStore`: account list renders items from store
    - `nonAdminUser_IsRedirected`: route guard with seller or buyer role redirects away
  - `pages/Accounts/AccountDetailPage.test.ts`:
    - `renders_DisplayName_EmailReference_AndStatus`: detail page shows expected fields
    - `suspendAccount_CallsCorrectEndpoint`: suspend action triggers suspend endpoint stub; status updated in view
    - `activateAccount_CallsCorrectEndpoint`: activate action triggers activate endpoint stub; status updated in view
  - `pages/Notifications/NotificationsPage.test.ts`:
    - `renders_NotificationItemsFromStore`: notification items shown for admin user
    - `renders_AdminApplicableChannelsInPreferences`: settings surface shows admin-applicable notification channels
  - `pages/Profile/ProfilePage.test.ts`:
    - `renders_DisplayNameAndEmailReference`: profile page shows display name and email reference
    - `profileUpdate_SubmitsToCorrectEndpoint`: edit form submit triggers update endpoint stub
  - `pages/Settings/SettingsPage.test.ts`:
    - `renders_WithoutError_ForAdminRole`: settings page renders without error for admin role
  - Use `createFakeAuthUser({role: 'admin'})`, `createMockAxios`, `stubApiResponse`, `createFakeSignalRHub`, `createTestI18n`
  - Do not import from `apps/buyer` or `apps/seller`
  - Acceptance: `pnpm --filter admin test` exits with code 0

---

## TDD Guide

### Create

- [ ] F012 Verify and update `TESTING.md` developer guide in `../hivespace.web`
  - File: `../hivespace.web/TESTING.md`
  - **Already exists** — do not recreate. Read the existing file and verify the sections below are present and accurate; add any missing sections.
  - Sections to verify (add if absent or incomplete):
  - **Co-location rule**: test files sit next to the source files they exercise — `stores/cart.test.ts` alongside `stores/cart.ts`; `pages/Cart/CartPage.test.ts` alongside `CartPage.vue`. Never use a top-level `__tests__/` folder. Jest `testMatch` pattern is `src/**/*.test.ts`.
  - **Store tests vs view tests**: store tests use `@pinia/testing` — no component rendering, no DOM, fast; view tests use `@testing-library/vue` — mount component with store wired up, stub HTTP calls via `stubApiResponse`. Explain when to use each.
  - **Shared test-utils reference**: table of each export from `@hivespace/shared/test-utils` (`createFakeAuthUser`, `createFakeAuthState`, `createMockAxios`, `stubApiResponse`, `createFakeSignalRHub`, `createFakePresignResponse`, `createTestI18n`) with one-line descriptions and when to use each.
  - **TDD workflow**: write store test (Red) → implement store action (Green) → write view test (Red) → wire up component (Green).
  - **Naming convention**: store tests use `action_When_Result`; view tests use `renders_What_FromWhere` or `action_CallsEndpoint`. Show concrete examples.
  - **i18n assertion rule**: all text assertions must use i18n key lookups — never assert on hardcoded display strings.
  - **Coverage policy**: 80% policy-scoped target; included paths (`pages/`, `stores/`, `composables/`, `router/`); excluded paths (assets, styles, i18n, types, thin services, config constants, presentational components).
  - Do not add product business logic, CI configuration, or workspace setup scripts
  - Acceptance: file exists with all sections; `pnpm test` results are not affected by this file's presence
