# Backend Tasks

## PaymentService

### Create

- [ ] B001 [US1] Create `CheckoutPaymentDomainTests` - AC1.1: one checkout payment links multiple orders
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/Domain/Payments/CheckoutPaymentDomainTests.cs`
  - Test: `Create_WithMultipleLinkedOrders_CreatesCheckoutPaymentWithReferenceAndAttempt`
  - Assert `Payment.CreateCheckout(...)` requires one or more linked orders, stores a unique `ReferenceNo`, creates attempt `1`, and sums linked order amounts to the payment amount.
  - Do not use a live database or gateway adapter; construct value objects and linked order DTOs in memory.
  - Acceptance: test compiles and fails red before B007 and B008 implement the aggregate behavior.

- [ ] B002 [US3] Create `PaymentReferenceNoGeneratorTests` - AC3.2: public payment reference uniqueness
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/Application/Payments/PaymentReferenceNoGeneratorTests.cs`
  - Test: `GenerateAsync_WhenCollisionOccurs_RetriesUntilUniquePayUlid`
  - Mock `IPaymentRepository.ReferenceNoExistsAsync` to return collision then success; assert format `PAY-{26-character uppercase ULID}` and no idempotency key reuse.
  - Do not expose database IDs or idempotency keys as public references.
  - Acceptance: test compiles and fails red before B009 implements generation.

- [ ] B003 [US1] Create `InitiatePaymentConsumerTests` - AC1.1/AC1.3: initiation validation
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/Application/Messaging/InitiatePaymentConsumerTests.cs`
  - Test: `Consume_MismatchedLinkedOrderTotal_PublishesPaymentInitiationFailed`
  - Given `InitiatePayment` with linked order totals not equal to `Amount`, assert `PaymentInitiationFailedIntegrationEvent` contains correlation, method, amount, currency, reason, and original linked orders.
  - Do not create a partial payment or gateway request on validation failure.
  - Acceptance: test compiles and fails red before B010 and B012 implement validation and failure publishing.

- [ ] B004 [US1] Create `PaymentAttemptOutcomeTests` - AC1.2/AC1.5: duplicate and stale callbacks
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/Application/Payments/PaymentAttemptOutcomeTests.cs`
  - Test: `HandleGatewayOutcome_StaleAttemptAfterNewerSuccess_DoesNotPublishOrderChangingEvent`
  - Arrange a checkout payment with attempt 1 expired and attempt 2 succeeded; assert duplicate/stale callback records diagnostics but does not publish a new `PaymentSucceededIntegrationEvent` or `PaymentFailedIntegrationEvent`.
  - Do not silently swallow invalid gateway data; failures must remain observable through result or log assertions.
  - Acceptance: test compiles and fails red before B013 implements stale outcome handling.

- [ ] B005 [US1] Create `CreatePaymentAttemptCommandHandlerTests` - AC1.4: retry under same reference
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/Application/Payments/CreatePaymentAttemptCommandHandlerTests.cs`
  - Test: `Handle_FailedVnPayPayment_CreatesNextAttemptWithSameReferenceNo`
  - Assert VNPay retry increments `AttemptNo`, keeps parent `ReferenceNo`, returns redirect URL, and rejects retry after any succeeded attempt.
  - Include COD switch test `Handle_FailedVnPayBeforeFulfillment_CreatesOfflineCodAttempt`.
  - Acceptance: test compiles and fails red before B014 implements retry command handling.

- [ ] B006 [US2] Create `GetPaymentMethodsQueryHandlerTests` - AC2.1/AC2.2: canonical method metadata
  - File: `../hivespace.microservice/tests/HiveSpace.PaymentService.Tests/Application/Payments/GetPaymentMethodsQueryHandlerTests.cs`
  - Test: `Handle_DefaultConfiguration_ReturnsCodVnpayAndFutureStripeInSortOrder`
  - Assert COD and VNPay are enabled and checkout-selectable, Stripe is not checkout-selectable and has `Future` or `Unavailable` availability.
  - Do not hardcode app-specific labels or checkout behavior outside PaymentService.
  - Acceptance: test compiles and fails red before B015 implements canonical method provider/query.

### Update

- [ ] B007 [US1] Update `Payment` aggregate for checkout-level payments
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Domain/Payments/Payment.cs`
  - Add fields/properties: `ReferenceNo`, `BuyerId`, `CheckoutCorrelationId`, `Amount`, `CurrencyCode`, `CurrentAttemptId`, aggregate `Status`, `IdempotencyKey`, `LinkedOrders`, and `Attempts`.
  - Add factory and behavior: `CreateCheckout(...)`, `AddAttempt(...)`, `MarkAttemptSucceeded(...)`, `MarkAttemptFailedOrExpired(...)`, and guards for no linked orders, mismatched totals, duplicate linked order IDs, and post-success retries.
  - Preserve historical reads for older one-order payment data and unknown method values; do not migrate order lifecycle decisions into PaymentService.
  - Acceptance: B001 domain test passes and the aggregate compiles with private setters/protected EF constructor patterns.

- [ ] B008 [US1] Create `PaymentAttempt` and `PaymentLinkedOrder` models
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Domain/Payments/PaymentAttempt.cs`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Domain/Payments/PaymentLinkedOrder.cs`
  - Include attempt fields: `Id`, `PaymentId`, `AttemptNo`, `MethodCode`, `GatewayCode`, `Amount`, `CurrencyCode`, `Status`, `GatewayTransactionId`, `GatewayResponse`, `RedirectUrl`, `FailureReasonCode`, `FailureReason`, `CreatedAt`, `ExpiresAt`, `CompletedAt`.
  - Include linked order fields: `PaymentId`, `OrderId`, `OrderCode`, `StoreId`, `Amount`, `CurrencyCode`, optional `StatusSnapshot`.
  - Do not store OrderService mutable lifecycle truth beyond allowed snapshots.
  - Acceptance: B001 and B004 compile against explicit child entities/value objects.

- [ ] B009 [US3] Create `PaymentReferenceNo` generator and repository uniqueness contract
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/IPaymentReferenceNoGenerator.cs`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/PaymentReferenceNoGenerator.cs`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Domain/Payments/IPaymentRepository.cs`
  - Add `GetByReferenceNoAsync`, `GetByOrderIdAsync`, and `ReferenceNoExistsAsync` to `IPaymentRepository`.
  - Generate `PAY-{26-character uppercase ULID}` and retry collisions before persistence.
  - Do not use idempotency keys, database IDs, or gateway transaction IDs as `ReferenceNo`.
  - Acceptance: B002 passes and repository implementations must compile with the new interface.

- [ ] B010 [US1] Update `InitiatePayment` consumer/handler for linked order validation
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/InitiatePayment/InitiatePaymentConsumer.cs`
  - Accept changed `InitiatePayment` contract fields: `CorrelationId`, `BuyerId`, `MethodCode`, `Amount`, `CurrencyCode`, `IdempotencyKey`, and `Orders`.
  - Validate method, amount, currency, linked order uniqueness, linked order sum, idempotency repeated request shape, and local currency-policy projection before payment creation.
  - Do not request VNPay or persist partial payments when validation fails.
  - Acceptance: B003 passes and failure paths publish `PaymentInitiationFailedIntegrationEvent`.

- [ ] B011 [US1] Update `shared checkout payment messaging contracts`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Payments/*.cs`; `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Orders/*.cs`
  - Update `InitiatePayment`, `PaymentInitiatedIntegrationEvent`, `PaymentInitiationFailedIntegrationEvent`, `PaymentSucceededIntegrationEvent`, `PaymentFailedIntegrationEvent`, and create/update `CheckoutPaymentOrderDto` exactly as `contracts/checkout-payment-messages.md`.
  - Ensure integration events still derive from the shared integration event base and commands/DTOs do not.
  - Do not rename contracts or add duplicate events with equivalent meaning.
  - Acceptance: PaymentService and OrderService projects compile against the changed payloads.

- [ ] B012 [US1] Update `PaymentService integration event publishing`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/PaymentIntegrationEventPublisher.cs`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/InitiatePayment/InitiatePaymentConsumer.cs`
  - Publish initiation, initiation failure, success, and failure events with `CorrelationId`, `PaymentId`, `PaymentAttemptId`, `AttemptNo`, `ReferenceNo`, method, amount, currency, redirect/failure details, and linked orders.
  - Use the existing service-owned publisher/outbox pattern; saga participant responses may remain consume-context workflow messages where established.
  - Do not publish order-changing outcome events for stale non-current attempts.
  - Acceptance: B003 and B004 pass and outbox/integration publisher tests remain green.

- [ ] B013 [US1] Update `VNPay return/IPN handling`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/VnPay/*.cs`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Infrastructure/Gateways/VnPay/*.cs`
  - Send `Payment.ReferenceNo` as VNPay merchant transaction reference; store gateway transaction IDs on `PaymentAttempt`.
  - Correlate return/IPN to the payment and current attempt, handle duplicate callbacks idempotently, record stale callbacks for diagnostics, and preserve latest successful attempt state.
  - Do not overwrite `ReferenceNo` with gateway transaction identifiers.
  - Acceptance: B004 passes and VNPay adapter tests verify merchant reference mapping.

- [ ] B014 [US1] Create `payment retry command and endpoint`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/CreatePaymentAttempt/*`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api/Endpoints/PaymentEndpoints.cs`
  - Implement `POST /api/v1/payments/{paymentId}/attempts` with request `methodCode` and `idempotencyKey`, response containing `paymentId`, `referenceNo`, and attempt details.
  - Allow VNPay retry and COD switch only after failed/expired/cancelled attempts, before fulfillment eligibility is lost, and before any attempt succeeds; reject Stripe until implemented.
  - Do not create a new checkout payment or new `ReferenceNo` for retries.
  - Acceptance: B005 passes and endpoint returns `201 Created` for new attempts.

- [ ] B015 [US2] Create `canonical payment methods query and endpoint`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/GetPaymentMethods/*`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api/Endpoints/PaymentEndpoints.cs`
  - Add `GET /api/v1/payments/methods` returning COD, VNPay, and Stripe metadata with `code`, `displayName`, `kind`, `gatewayCode`, `isEnabled`, `isCheckoutSelectable`, `availability`, and `sortOrder`.
  - Keep Stripe visible for non-checkout metadata but disabled/not checkout-selectable.
  - Do not let frontend apps become the source of truth for payment method lists.
  - Acceptance: B006 passes and API response matches `contracts/payment-methods-api.md`.

- [ ] B016 [US3] Update `PaymentService read queries and DTOs`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/GetPaymentDetail/*`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/GetPaymentByOrder/*`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Payments/GetPaymentByReference/*`
  - Extend payment detail with `referenceNo`, method metadata, explicit money, gateway merchant reference, latest attempt, full attempt history for privileged callers, and linked orders.
  - Add `GET /api/v1/payments/by-reference/{referenceNo}` and change by-order lookup to return shared checkout payment for any linked order.
  - Do not leak other buyers' linked order details to unauthorized callers.
  - Acceptance: payment API tests cover buyer-owned read, privileged read with attempt history, by-reference lookup, and by-order linked payment lookup.

- [ ] B017 [US3] Update `PaymentService EF mapping and migration`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Infrastructure/Persistence/Configurations/PaymentConfiguration.cs`; `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Infrastructure/Persistence/Migrations/*`
  - Map unique `reference_no`, checkout correlation, current attempt, linked order rows/owned collection, payment attempts, gateway response diagnostics, and indexes for `reference_no`, linked `order_id`, and gateway transaction ID when present.
  - Preserve historical payment columns/data readability and add migration defaults/nullability intentionally.
  - Do not hard-delete existing payment data or require data from OrderService database.
  - Acceptance: PaymentService DbContext migration scaffolds/applies in local test database and repository queries can load attempts and linked orders.

- [ ] B018 [US1] Create `CheckoutSagaPaymentStateTests` - AC1.2/AC1.5: current-attempt saga handling
  - File: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/Application/Checkout/CheckoutSagaPaymentStateTests.cs`
  - Test: `WhenPaymentSucceededForCurrentAttempt_MarksEveryLinkedOrderPaidOnce`
  - Include stale callback test `WhenPaymentFailedForOlderAttemptAfterSuccess_DoesNotChangeOrders`.
  - Use MassTransit test harness or existing saga fixture; do not call PaymentService directly.
  - Acceptance: test compiles and fails red before B024 and B025 implement saga state/outcome behavior.

- [ ] B019 [US3] Create `OrderCodeGeneratorTests` - AC3.1: unique order code format
  - File: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/Application/Orders/OrderCodeGeneratorTests.cs`
  - Test: `GenerateAsync_WhenCollisionOccurs_RetriesUntilUniqueOrdUlid`
  - Mock `IOrderRepository.OrderCodeExistsAsync` to collide once; assert `ORD-{26-character uppercase ULID}` format.
  - Do not rewrite historical order identifiers.
  - Acceptance: test compiles and fails red before B021 implements generation.

- [ ] B020 [US3] Create `OrderDetailPaymentSummaryTests` - AC3.1/AC3.3: linked payment summary
  - File: `../hivespace.microservice/tests/HiveSpace.OrderService.Tests/Application/Orders/GetOrderDetailQueryHandlerTests.cs`
  - Test: `Handle_OrderWithLinkedCheckoutPayment_ReturnsOrderCodeAndPaymentReferenceSummary`
  - Assert buyer/seller/admin order detail DTOs include `OrderCode`, shared payment ID/reference, method code/label field, status summary, and latest attempt identity when known.
  - Do not call PaymentService synchronously from OrderService read handlers.
  - Acceptance: test compiles and fails red before B027 implements DTO/read updates.

- [ ] B021 [US3] Update `OrderCode` generation and repository uniqueness contract
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Orders/IOrderCodeGenerator.cs`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Orders/OrderCodeGenerator.cs`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Domain/Orders/IOrderRepository.cs`
  - Add `OrderCodeExistsAsync` and generate `ORD-{26-character uppercase ULID}` with collision retry before persistence.
  - Use new generator for newly created orders only; historical order code reads/searches must remain valid.
  - Do not accept client-supplied order codes.
  - Acceptance: B019 passes and order creation compiles with generator dependency.

- [ ] B022 [US1] Update `checkout order creation`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Checkout/CreateOrder/*`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Checkout/CheckoutSagaState.cs`
  - Store generated order ID, `OrderCode`, store ID, amount, and currency in saga state after order creation.
  - Ensure single-order checkout uses the same linked-order shape as multi-order checkout.
  - Do not compute payment amount separately from final checkout totals already accepted by the checkout flow.
  - Acceptance: saga state contains a non-empty linked order set before payment initiation.

- [ ] B023 [US1] Update `OrderService payment initiation publishing`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Checkout/CheckoutStateMachine.cs`
  - Publish changed `InitiatePayment` with checkout correlation, buyer ID, canonical method code, final amount, currency, idempotency key, and full linked order set.
  - Reject mismatched amount, currency, method, or linked order set before sending payment command where OrderService can detect the mismatch.
  - Do not create one payment command per generated order.
  - Acceptance: B018 setup receives one `InitiatePayment` message for a multi-order checkout.

- [ ] B024 [US1] Update `checkout saga state`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Checkout/CheckoutSagaState.cs`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Infrastructure/Persistence/Configurations/CheckoutSagaStateConfiguration.cs`
  - Add `PaymentId`, `PaymentReferenceNo`, `CurrentPaymentAttemptId`, `CurrentPaymentAttemptNo`, `PaymentOutcomeAppliedAt`, and linked order snapshot persistence.
  - Map new saga fields in EF/MassTransit saga persistence without losing existing state data.
  - Do not add a new saga for this feature; update the existing checkout saga.
  - Acceptance: saga persistence tests or build confirm new fields are mapped.

- [ ] B025 [US1] Update `payment outcome saga transitions`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Checkout/CheckoutStateMachine.cs`
  - On `PaymentInitiatedIntegrationEvent`, store shared payment ID/reference and current attempt identity; COD proceeds to COD marking and VNPay waits for outcome/redirect handling.
  - On success/failure, validate correlation, linked order set, and current attempt identity before applying outcome to every linked order exactly once.
  - Ignore stale older-attempt outcomes after a newer attempt exists or payment already succeeded.
  - Acceptance: B018 passes for current attempt success and stale callback scenarios.

- [ ] B026 [US1] Update `order payment and COD marking commands`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Orders/MarkOrderAsPaid/*`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Orders/MarkOrderAsCOD/*`
  - Apply paid/COD status to all linked orders idempotently and persist shared payment summary fields on each order.
  - Include current attempt ID/number and `PaymentReferenceNo` in order payment summary updates.
  - Do not mark orders outside the event linked order set.
  - Acceptance: batch marking tests confirm every linked order is updated once and unrelated orders remain unchanged.

- [ ] B027 [US3] Update `order detail and list DTOs`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Orders/Queries/*`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/Endpoints/OrderEndpoints.cs`
  - Include `OrderCode`, shared `PaymentId`, `PaymentReferenceNo`, `PaymentMethodCode`, method display field if locally available, `PaymentStatus`, `PaymentAttemptId`, and `PaymentAttemptNo` in buyer/seller/admin order reads.
  - Preserve explicit money metadata and invalid-money diagnostics from existing order reads.
  - Do not synchronously query PaymentService from OrderService reads.
  - Acceptance: B020 passes and API contract remains backward-compatible for historical orders with nullable payment summary.

- [ ] B028 [US1] Update `OrderService EF mapping and migration`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Infrastructure/Persistence/Configurations/OrderConfiguration.cs`; `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Infrastructure/Persistence/Migrations/*`
  - Map new order columns for `order_code` uniqueness if not already unique, shared payment ID/reference, method code, payment status, payment attempt ID, and attempt number.
  - Add indexes for `order_code` and `payment_reference_no` to support support/admin lookup and display.
  - Do not require foreign keys to PaymentService database.
  - Acceptance: OrderService DbContext migration scaffolds/applies in local test database.

## ApiGateway

### Verify

- [ ] B029 [US2] Verify `PaymentService` route coverage for new payment endpoints
  - File: `../hivespace.microservice/src/HiveSpace.ApiGateway/HiveSpace.YarpApiGateway/appsettings*.json`
  - Confirm existing `/api/v1/payments/**` route forwards `GET /api/v1/payments/methods`, `GET /api/v1/payments/by-reference/{referenceNo}`, and `POST /api/v1/payments/{paymentId}/attempts` to PaymentService.
  - Update only gateway route config if the existing prefix route does not cover these paths; keep ApiGateway free of business validation.
  - Do not edit `../hivespace.config`.
  - Acceptance: route table covers all PaymentService endpoints through the gateway without adding business logic.
