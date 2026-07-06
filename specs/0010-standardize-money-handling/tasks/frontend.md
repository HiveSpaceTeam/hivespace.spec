# Frontend Tasks

## Shared Package

### Create

- [ ] F001 [US3] Create shared money formatter and input tests
  - File: `../hivespace.web/packages/shared/src/composables/useMoneyFormatter.test.ts`; `../hivespace.web/packages/shared/src/composables/useMoneyInput.test.ts`; `../hivespace.web/packages/shared/src/types/money.types.test.ts`
  - Test: `should render VND without fractional digits`, `should render USD/EUR smallest-unit values as major units by default`, `should return invalid placeholder for missing currency`, and `should convert USD/EUR major-unit input to smallest-unit payloads`
  - Reuse the frontend `should ...` naming style and shared `createTestI18n` helpers where rendering text is asserted
  - Acceptance: tests compile and fail (red) before the shared money helpers are implemented

### Update

- [ ] F002 [US3] Update `@hivespace/shared` money types, formatter/input composables, exports, and shared i18n
  - File: `../hivespace.web/packages/shared/src/types/money.types.ts`; `../hivespace.web/packages/shared/src/types/index.ts`; `../hivespace.web/packages/shared/src/composables/useMoneyFormatter.ts`; `../hivespace.web/packages/shared/src/composables/useMoneyInput.ts`; `../hivespace.web/packages/shared/src/composables/index.ts`; `../hivespace.web/packages/shared/src/internal.ts`; `../hivespace.web/packages/shared/src/i18n/locales/en/common.json`; `../hivespace.web/packages/shared/src/i18n/locales/vi/common.json`
  - Add explicit `MoneyDisplay`, `MoneyIssue`, `PlatformCurrencyConfig`, `PlatformCurrencyConfigItem`, and money-input contracts plus one shared formatter/input rule for `VND`, `USD`, and `EUR`
  - Keep `USD`/`EUR` defaulted to major-unit display/input and preserve an optional explicit raw-smallest-unit mode only when a surface intentionally opts in
  - Do not reuse `useNumberInputFormatter` for `USD`/`EUR` entry or create app-local money helpers that duplicate the shared behavior
  - Acceptance: all three apps can import one shared money formatter/input API and shared i18n includes invalid-money placeholder copy

## Admin App

### Create

- [ ] F003 [US1] Create admin configuration tests for persisted currency configuration on the generic foundation
  - File: `../hivespace.web/apps/admin/src/stores/configuration.store.test.ts`; `../hivespace.web/apps/admin/src/pages/Configuration/ConfigurationPage.test.ts`
  - Test: `should load persisted currency items and default`, `should block disabling the current default until a replacement is selected`, and `should submit the typed item list with versioned config metadata`
  - Mock the configuration service only; keep page tests focused on rendered controls and store interactions
  - Acceptance: tests compile and fail (red) before the admin configuration implementation task runs

### Update

- [ ] F004 [US1] Update admin types, service, store, page, and i18n for generic foundation-backed currency management
  - File: `../hivespace.web/apps/admin/src/types/configuration.types.ts`; `../hivespace.web/apps/admin/src/types/index.ts`; `../hivespace.web/apps/admin/src/services/configuration.service.ts`; `../hivespace.web/apps/admin/src/services/index.ts`; `../hivespace.web/apps/admin/src/stores/configuration.store.ts`; `../hivespace.web/apps/admin/src/stores/index.ts`; `../hivespace.web/apps/admin/src/pages/Configuration/ConfigurationPage.vue`; `../hivespace.web/apps/admin/src/i18n/locales/en/configuration.json`; `../hivespace.web/apps/admin/src/i18n/locales/vi/configuration.json`; `../hivespace.web/apps/admin/src/i18n/index.ts`
  - Replace mock-only currency chips with persisted rows from `GET/PUT /api/v1/admins/configuration/currencies`, carry config-level default/version state, and render explicit enable/disable plus default-selection controls
  - Keep route ownership and admin auth behavior unchanged; do not introduce app-local money formatters in admin
  - Acceptance: the admin configuration page can load, edit, and save the currency configuration using the real backend contract and a UI shape reusable for future config types

- [ ] F012 [US3] Create admin money-surface tests for shared formatter adoption outside the configuration workspace
  - File: `../hivespace.web/apps/admin/src/pages/Buyers/BuyersPage.test.ts`
  - Test: `should render money-bearing values with shared formatter rules` and `should show invalid-money placeholder when backend marks a value invalid`
  - Keep tests focused on existing admin page rendering behavior; do not duplicate seller or buyer formatter coverage here
  - Acceptance: tests compile and fail (red) before the admin money-surface implementation task runs

- [ ] F013 [US3] Update admin money-bearing pages to use the shared money formatter
  - File: `../hivespace.web/apps/admin/src/pages/Buyers/BuyersPage.vue`
  - Replace hard-coded `VND`, `vi-VN`, or locale-only money rendering in shipped admin money-bearing pages with the shared formatter and invalid-money placeholder behavior
  - Keep existing page ownership and data flow intact; do not add an admin-local formatter helper that duplicates `@hivespace/shared`
  - Acceptance: shipped admin money-bearing pages in scope render `VND`, `USD`, and `EUR` consistently with seller and buyer surfaces

## Seller App

### Create

- [ ] F005 [US2] Create seller tests for normalized coupon/product money handling and policy-driven currency options
  - File: `../hivespace.web/apps/seller/src/stores/coupon.store.test.ts`; `../hivespace.web/apps/seller/src/stores/product.test.ts`; `../hivespace.web/apps/seller/src/composables/useCouponValidation.test.ts`; `../hivespace.web/apps/seller/src/pages/Marketing/CouponDetailPage.test.ts`; `../hivespace.web/apps/seller/src/pages/Products/UpsertProductPage.test.ts`
  - Test: `should preserve one coupon currency across money fields`, `should submit USD/EUR major-unit input as smallest-unit payloads`, and `should render invalid-money placeholder when backend marks a value invalid`
  - Mock store services, not components, for store tests; keep page tests focused on shared money-input/formatter wiring
  - Acceptance: tests compile and fail (red) before the seller implementation tasks run

### Update

- [ ] F006 [US2] Update seller types, services, stores, and validation composable for normalized money contracts
  - File: `../hivespace.web/apps/seller/src/types/coupon.types.ts`; `../hivespace.web/apps/seller/src/types/product.types.ts`; `../hivespace.web/apps/seller/src/types/index.ts`; `../hivespace.web/apps/seller/src/services/coupon.service.ts`; `../hivespace.web/apps/seller/src/services/product.service.ts`; `../hivespace.web/apps/seller/src/services/configuration.service.ts`; `../hivespace.web/apps/seller/src/services/index.ts`; `../hivespace.web/apps/seller/src/stores/coupon.store.ts`; `../hivespace.web/apps/seller/src/stores/product.store.ts`; `../hivespace.web/apps/seller/src/composables/useCouponValidation.ts`
  - Normalize coupon/product contracts to explicit money metadata, load the authenticated currency configuration for authoring flows, and remove hard-coded `VND` or symbol assumptions from validation logic
  - Do not bypass the store layer by calling services directly from pages
  - Acceptance: seller stores and composables expose policy-driven currency options plus normalized money DTOs compatible with the shared formatter/input APIs

- [ ] F007 [US2] Update seller coupon and product authoring/detail pages to use shared money formatter and input handling
  - File: `../hivespace.web/apps/seller/src/pages/Marketing/CouponDetailPage.vue`; `../hivespace.web/apps/seller/src/pages/Marketing/CouponListPage.vue`; `../hivespace.web/apps/seller/src/pages/Products/UpsertProductPage.vue`; `../hivespace.web/apps/seller/src/pages/Products/ProductListPage.vue`
  - Replace page-local number/currency formatting and generic integer money entry with the shared formatter/input helpers; ensure coupon/product forms submit smallest-unit values with one canonical currency
  - Keep UX within the existing page structures; do not create app-local duplicate input primitives when shared `Input` plus shared money composables are sufficient
  - Acceptance: seller coupon/product pages render and edit `VND`, `USD`, and `EUR` consistently and submit normalized payloads

- [ ] F008 [US3] Update seller order and translation surfaces to remove hard-coded money rendering
  - File: `../hivespace.web/apps/seller/src/pages/Orders/OrderManagementPage.vue`; `../hivespace.web/apps/seller/src/components/orders/ProductCell.vue`; `../hivespace.web/apps/seller/src/i18n/locales/en/coupon.json`; `../hivespace.web/apps/seller/src/i18n/locales/vi/coupon.json`; `../hivespace.web/apps/seller/src/i18n/locales/en/product.json`; `../hivespace.web/apps/seller/src/i18n/locales/vi/product.json`; `../hivespace.web/apps/seller/src/i18n/locales/en/order.json`; `../hivespace.web/apps/seller/src/i18n/locales/vi/order.json`
  - Replace any remaining `vi-VN`, dong-symbol, or locale-only money formatting in seller order/product/coupon displays and add copy for invalid-money states or disabled-currency errors where surfaced
  - Keep English and Vietnamese files synchronized in the same change
  - Acceptance: seller read-only money surfaces use the shared formatter and show consistent invalid-money placeholders when required

## Buyer App

### Create

- [ ] F009 [US3] Create buyer tests for explicit money metadata and mixed-currency UI handling
  - File: `../hivespace.web/apps/buyer/src/stores/cart.store.test.ts`; `../hivespace.web/apps/buyer/src/stores/checkout.store.test.ts`; `../hivespace.web/apps/buyer/src/stores/orders.store.test.ts`; `../hivespace.web/apps/buyer/src/stores/payment.store.test.ts`; `../hivespace.web/apps/buyer/src/pages/Cart/CartPage.test.ts`; `../hivespace.web/apps/buyer/src/pages/Checkout/CheckoutPage.test.ts`; `../hivespace.web/apps/buyer/src/pages/Payment/PaymentResultPage.test.ts`; `../hivespace.web/apps/buyer/src/pages/Account/OrderDetailPage.test.ts`
  - Test: `should render shared money metadata consistently`, `should show invalid-money placeholder when backend marks a value invalid`, and `should surface mixed-currency checkout errors without guessed totals`
  - Use co-located tests only; do not add a top-level `__tests__` folder
  - Acceptance: tests compile and fail (red) before the buyer implementation tasks run

### Update

- [ ] F010 [US3] Update buyer types, services, and stores for explicit money metadata and mixed-currency error handling
  - File: `../hivespace.web/apps/buyer/src/types/cart.types.ts`; `../hivespace.web/apps/buyer/src/types/checkout.types.ts`; `../hivespace.web/apps/buyer/src/types/order.types.ts`; `../hivespace.web/apps/buyer/src/types/payment.types.ts`; `../hivespace.web/apps/buyer/src/types/product.types.ts`; `../hivespace.web/apps/buyer/src/types/index.ts`; `../hivespace.web/apps/buyer/src/services/cart.service.ts`; `../hivespace.web/apps/buyer/src/services/checkout.service.ts`; `../hivespace.web/apps/buyer/src/services/order.service.ts`; `../hivespace.web/apps/buyer/src/services/payment.service.ts`; `../hivespace.web/apps/buyer/src/services/product.service.ts`; `../hivespace.web/apps/buyer/src/stores/cart.store.ts`; `../hivespace.web/apps/buyer/src/stores/checkout.store.ts`; `../hivespace.web/apps/buyer/src/stores/orders.store.ts`; `../hivespace.web/apps/buyer/src/stores/payment.store.ts`; `../hivespace.web/apps/buyer/src/stores/product.store.ts`
  - Preserve backend money metadata and error codes end to end, remove numeric-enum or implicit-`VND` assumptions, and surface mixed-currency failures through existing store-driven UX
  - Keep HTTP in services and orchestration/state in stores; do not move API calls into pages/components
  - Acceptance: buyer stores expose explicit money metadata and error state needed for consistent rendering across cart, checkout, order, payment, and product flows

- [ ] F011 [US3] Update buyer pages and components to use the shared money formatter everywhere in scope
  - File: `../hivespace.web/apps/buyer/src/pages/Product/ProductDetailPage.vue`; `../hivespace.web/apps/buyer/src/pages/Cart/CartPage.vue`; `../hivespace.web/apps/buyer/src/pages/Checkout/CheckoutPage.vue`; `../hivespace.web/apps/buyer/src/pages/Payment/PaymentResultPage.vue`; `../hivespace.web/apps/buyer/src/pages/Profile/OrdersPage.vue`; `../hivespace.web/apps/buyer/src/pages/Account/OrderDetailPage.vue`; `../hivespace.web/apps/buyer/src/components/common/AvailableCouponPopover.vue`; `../hivespace.web/apps/buyer/src/components/home/ProductCard.vue`; `../hivespace.web/apps/buyer/src/components/home/FlashSale.vue`
  - Replace all hard-coded `VND`/`vi-VN` money formatting in the scoped buyer surfaces with the shared formatter and display invalid-money placeholders where backend metadata requires it
  - Keep component ownership unchanged; do not create page-local money utilities or duplicate shared formatting rules
  - Acceptance: the currently shipped buyer money surfaces in scope render `VND`, `USD`, and `EUR` consistently with the same symbol/code and precision rules
