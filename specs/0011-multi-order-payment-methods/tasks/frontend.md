# Frontend Tasks

## Shared Package

### Update

- [ ] F003 [US1] Update `shared payment and order types`
  - File: `../hivespace.web/packages/shared/src/types/payment.types.ts`; `../hivespace.web/packages/shared/src/types/order.types.ts`
  - Add payment method, payment detail, payment attempt, linked order, retry request/response, and order payment summary types matching `contracts/payment-api.md` and `contracts/payment-methods-api.md`.
  - Add order fields: `orderCode`, `paymentId`, `paymentReferenceNo`, `paymentMethodCode`, `paymentStatus`, `paymentAttemptId`, and `paymentAttemptNo` where order detail/list DTOs are typed.
  - Do not introduce app-local duplicate type definitions.
  - Acceptance: shared package type-checks; do not add type-only tests because `types/**` is outside policy-scoped coverage.

- [ ] F004 [US2] Update `shared payment service APIs`
  - File: `../hivespace.web/packages/shared/src/services/payment.service.ts`
  - Add `getPaymentMethods`, `getPaymentByReference`, `getPaymentByOrder`, `getPaymentDetail`, and `createPaymentAttempt` methods using shared `ApiService`.
  - Preserve bearer/correlation/error handling through shared HTTP infrastructure.
  - Do not bypass `ApiService` or use absolute backend service URLs.
  - Acceptance: methods return typed responses from F003 and are exercised indirectly by covered store/page tests; do not add thin service-wrapper tests unless the service owns branching behavior.

- [ ] F005 [US3] Update `shared order service consumers`
  - File: `../hivespace.web/packages/shared/src/services/order.service.ts`; `../hivespace.web/packages/shared/src/stores/orders*.ts`
  - Thread new order code and linked payment summary fields through existing order detail/list service responses and stores/factories.
  - Keep existing money metadata handling intact.
  - Do not fetch PaymentService from component bodies to fill order summaries.
  - Acceptance: existing shared order consumers compile with new optional fields.

- [ ] F006 [US2] Update `shared i18n resources`
  - File: `../hivespace.web/packages/shared/src/i18n/en*.ts`; `../hivespace.web/packages/shared/src/i18n/vi*.ts`
  - Add keys for `payments.methods.cod`, `payments.methods.vnpay`, `payments.methods.stripe`, `payments.methods.unavailable`, `payments.methods.future`, `payments.referenceNo`, `payments.linkedOrders`, `payments.attempt`, `payments.retryPayment`, `payments.checkoutPayment`, `orders.orderCode`, and `orders.paymentReference`.
  - Update English and Vietnamese resources together using the existing namespace style.
  - Do not reuse one i18n key as both string and object namespace.
  - Acceptance: i18n type checks pass and no user-facing text is hardcoded for these labels.

## Buyer App

### Create

- [ ] F007 [US1] Create `buyer checkout payment method tests`
  - File: `../hivespace.web/apps/buyer/src/stores/checkout.store.test.ts`; `../hivespace.web/apps/buyer/src/pages/Checkout/CheckoutPage.test.ts`
  - Test: should load checkout-selectable methods and hide future Stripe in checkout
  - Mock `paymentService.getPaymentMethods` to return COD, VNPay, and Stripe; assert checkout state includes only enabled checkout-selectable methods for selection.
  - Test: should create retry attempt under same payment reference after failed VNPay
  - Acceptance: tests compile and fail red before F008 and F009 implement buyer checkout changes.

### Update

- [ ] F008 [US1] Update `buyer checkout store`
  - File: `../hivespace.web/apps/buyer/src/stores/checkout.store.ts`
  - Load methods from `getPaymentMethods`, store checkout-selectable methods, use canonical `methodCode` in checkout request, and call `createPaymentAttempt` for retry after failed/expired VNPay.
  - Preserve the same `paymentReferenceNo` in retry state and store latest attempt redirect/status.
  - Do not hardcode a local method list or allow Stripe checkout selection before backend marks it selectable.
  - Acceptance: F007 store assertions pass.

- [ ] F009 [US1] Update `buyer checkout UI`
  - File: `../hivespace.web/apps/buyer/src/pages/Checkout/CheckoutPage.vue`; `../hivespace.web/apps/buyer/src/components/checkout/*.vue`
  - Render payment methods from checkout store metadata, allow COD and VNPay, hide or disable Stripe based on `isCheckoutSelectable`, and show retry action for failed/expired VNPay attempts.
  - Use i18n keys from F006 for method labels, unavailable/future copy, reference number, and retry text.
  - Do not place domain HTTP calls in page/component bodies.
  - Acceptance: F007 page assertions pass and UI has no hardcoded payment labels.

- [ ] F010 [US3] Update `buyer order views`
  - File: `../hivespace.web/apps/buyer/src/pages/orders/*.vue`; `../hivespace.web/apps/buyer/src/components/orders/*.vue`
  - Display `OrderCode`, shared `PaymentReferenceNo`, method label, payment status, and latest attempt state when available.
  - Preserve historical orders with missing reference or older code formats.
  - Do not infer payment truth from order status alone when payment summary fields exist.
  - Acceptance: buyer order detail can show distinct `ORD-*` and shared `PAY-*` values for linked orders.

- [ ] F011 [US1] Update `buyer checkout redirect handling`
  - File: `../hivespace.web/apps/buyer/src/router/*`; `../hivespace.web/apps/buyer/src/pages/payment*.vue`; `../hivespace.web/apps/buyer/src/stores/checkout.store.ts`
  - Ensure VNPay redirect state uses latest attempt redirect URL and checkout-level payment reference, not a per-order payment assumption.
  - Keep COD path offline with no gateway redirect.
  - Do not create frontend-only payment IDs or references.
  - Acceptance: retry after failed/expired VNPay redirects through the new attempt while keeping same payment reference in state.

## Seller App

### Update

- [ ] F012 [US2] Update `seller order payment display`
  - File: `../hivespace.web/apps/seller/src/pages/orders/*.vue`; `../hivespace.web/apps/seller/src/stores/orders*.ts`
  - Load or consume PaymentService canonical method metadata for order/payment labels and availability meaning.
  - Show COD, VNPay, and future/unavailable Stripe consistently where payment methods are displayed.
  - Do not maintain seller-local payment label constants.
  - Acceptance: seller order views display canonical labels and compile without hardcoded method text.

- [ ] F013 [US3] Update `seller order views`
  - File: `../hivespace.web/apps/seller/src/pages/orders/*.vue`; `../hivespace.web/apps/seller/src/components/orders/*.vue`
  - Display shared `PaymentReferenceNo`, payment method, payment status, and buyer-visible `OrderCode` for seller-owned orders.
  - Preserve seller authorization boundaries; do not expose linked orders from other stores unless the backend response explicitly includes them for the seller.
  - Do not fetch admin payment detail from seller views.
  - Acceptance: seller can identify the shared payment reference for an order without cross-store leakage.

## Admin App

### Update

- [ ] F014 [US2] Update `admin payment method metadata`
  - File: `../hivespace.web/apps/admin/src/stores/payments*.ts`; `../hivespace.web/apps/admin/src/pages/payments*.vue`
  - Load PaymentService method metadata for admin payment/order surfaces and show Stripe as future/unavailable until selectable.
  - Use shared payment service and i18n keys.
  - Do not define admin-only COD/VNPay/Stripe label constants.
  - Acceptance: admin app displays the same canonical method labels as buyer and seller.

- [ ] F015 [US3] Update `admin payment reference lookup`
  - File: `../hivespace.web/apps/admin/src/stores/payments*.ts`; `../hivespace.web/apps/admin/src/pages/payments*.vue`; `../hivespace.web/apps/admin/src/components/payments*.vue`
  - Add by-reference lookup using `GET /api/v1/payments/by-reference/{referenceNo}` and render linked orders with `orderId`, `orderCode`, `storeId`, amount, and currency.
  - Render latest attempt and full attempt history when backend returns it for privileged users.
  - Do not treat `ReferenceNo` as an idempotency key or gateway transaction ID.
  - Acceptance: admin can search a `PAY-*` reference and see covered `ORD-*` orders and attempt history.

- [ ] F016 [US3] Update `admin order views`
  - File: `../hivespace.web/apps/admin/src/pages/orders*.vue`; `../hivespace.web/apps/admin/src/stores/orders*.ts`
  - Display `OrderCode`, shared `PaymentReferenceNo`, method metadata, payment status, and latest attempt summary on order detail/list surfaces.
  - Link to payment reference lookup/detail using shared routing patterns if present.
  - Do not duplicate payment detail state when shared payment store already owns it.
  - Acceptance: admin can navigate from an order to its shared checkout payment reference.

### Verify

- [ ] F017 [US2] Verify `app-local payment method constants`
  - File: `../hivespace.web/apps/*/src/**/*.{ts,vue}`; `../hivespace.web/packages/shared/src/**/*.{ts,vue}`
  - Search for hardcoded COD/VNPay/Stripe method option arrays outside backend-response fixtures, i18n resources, and tests.
  - Replace app-local constants with PaymentService metadata consumption where found.
  - Do not remove legitimate display translations or test fixtures.
  - Acceptance: final search finds no checkout/payment method source-of-truth outside PaymentService API consumption.
