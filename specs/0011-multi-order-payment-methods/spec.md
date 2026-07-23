# Feature Specification: Multi-Order Payment Methods

- **Feature Branch**: `0012-multi-order-payment-methods`
- **Created**: 2026-07-10
- **Status**: Implemented
- **Implemented**: 2026-07-19
- **Input**: User description: "I want to add a spec to refactor payment service to handle one payment for multiple orders to meet checkout flow, also config payment method in all app to be cod, vnpay and stripe(will be implemented later)"

## Implementation Notes

- Feature shipped with PaymentService-owned checkout-level payments, canonical payment method metadata, `PAY-{ULID}` payment references, and child payment attempts.
- OrderService shipped with `ORD-{ULID}` order codes, linked payment summaries, and checkout saga current-attempt validation for shared payment outcomes.
- User-owned E2E task `V009` remains an explicit external validation follow-up unless separately confirmed by the user; it was not marked complete by the agent.

## Clarifications

### Session 2026-07-11

- Q: How should COD be represented for multi-order checkout payment tracking? -> A: COD creates a checkout-level offline payment record linked to all generated orders, with no gateway transaction.
- Q: Where should the canonical payment method set be owned? -> A: Backend owns canonical payment method configuration and exposes it to all apps.
- Q: Which service owns canonical payment method configuration? -> A: PaymentService owns and exposes canonical payment method configuration.
- Q: How should Stripe appear before gateway implementation is enabled? -> A: Show Stripe in non-checkout displays/configuration, but hide or disable it in buyer checkout selection.
- Q: How should order codes and payment reference numbers relate in a multi-order checkout? -> A: Each order gets its own unique `OrderCode`, and the shared checkout payment gets one separate `ReferenceNo` linked to all orders.
- Q: Which visible format should new order codes and payment reference numbers use? -> A: Use prefixed uppercase ULIDs: `ORD-{ULID}` for orders and `PAY-{ULID}` for payments.
- Q: Where should payment `ReferenceNo` be used? -> A: Use `ReferenceNo` in APIs/UI and as the merchant transaction reference sent to VNPay for reconciliation.
- Q: How should buyer payment retry be modeled after a failed or expired checkout payment attempt? -> A: Keep one checkout-level payment and one stable `ReferenceNo`, then create child `PaymentAttempt` records for each retry.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Pay Once For Multi-Order Checkout (Priority: P1)

As a buyer checking out items that become multiple orders, I want to complete one payment action for the whole checkout so I do not need to pay separately for each seller or order.

**Why this priority**: This is the core checkout gap. Multi-order checkout cannot feel complete if each generated order requires a separate payment attempt.

**Independent Test**: Can be tested by checking out selected cart items that produce more than one order, choosing an online payment method, and confirming that the buyer receives one payment request covering all generated orders.

**Acceptance Scenarios**:

1. **Given** a buyer has selected cart items that generate multiple orders, **When** the buyer chooses VNPay and starts checkout, **Then** the system creates one payment for the checkout and associates it with every generated order.
2. **Given** a buyer completes the single online payment successfully, **When** the payment result is confirmed, **Then** every linked order reflects the successful payment outcome without requiring additional buyer payment actions.
3. **Given** a buyer's single online payment fails or expires, **When** the failure is confirmed, **Then** every linked order remains unpaid or failed according to the checkout outcome and the buyer sees one clear checkout-level failure.
4. **Given** a buyer's VNPay attempt fails or expires while the linked orders remain eligible for payment retry, **When** the buyer retries payment, **Then** the system creates a new payment attempt under the same checkout payment `ReferenceNo` instead of creating a second checkout payment.
5. **Given** a buyer retries payment and the new attempt succeeds, **When** an older failed or expired attempt later sends a gateway callback, **Then** the older attempt outcome is ignored for order transitions and the linked orders remain paid exactly once.

---

### User Story 2 - Use Consistent Payment Methods Across Apps (Priority: P2)

As a buyer, seller, or admin, I want every app to use the same payment method set so checkout, order views, and configuration screens describe payment choices consistently.

**Why this priority**: Inconsistent method names or availability across apps creates checkout errors, support confusion, and incorrect order expectations.

**Independent Test**: Can be tested by reviewing payment method choices and order/payment labels in buyer, seller, and admin surfaces and confirming they use the same canonical set: COD, VNPay, and Stripe as a future unavailable method.

**Acceptance Scenarios**:

1. **Given** payment methods are shown in buyer checkout, seller order views, or admin payment-related views, **When** the app displays method names or statuses, **Then** the labels come from PaymentService-owned canonical payment method configuration containing COD, VNPay, and Stripe.
2. **Given** Stripe is planned but not yet available for real checkout payment, **When** a buyer chooses a payment method, **Then** Stripe is hidden or disabled in checkout selection while remaining visible as unavailable or future metadata in non-checkout displays and configuration.
3. **Given** an existing order or payment uses COD or VNPay, **When** any app displays its payment method, **Then** the same method label and availability meaning is shown across apps.

---

### User Story 3 - Track One Payment Across Linked Orders (Priority: P3)

As a buyer, seller, or admin reviewing checkout history, I want to understand which orders belong to the same payment so payment support and reconciliation are clear.

**Why this priority**: Once one payment covers multiple orders, users need traceability from each order back to the shared payment without changing the underlying order ownership model.

**Independent Test**: Can be tested by opening each order generated from a multi-order checkout and confirming each order references the same checkout-level payment where applicable.

**Acceptance Scenarios**:

1. **Given** a checkout produced multiple orders paid by one online payment, **When** a user views any linked order detail, **Then** the order shows that it belongs to the shared payment for that checkout.
2. **Given** a support or admin user reviews the shared payment, **When** they inspect its covered orders, **Then** the payment exposes one payment `ReferenceNo`, the linked order list, each linked order's distinct `OrderCode`, and the total amount represented by those orders.
3. **Given** a checkout uses COD, **When** users review the generated orders, **Then** the orders reference the same checkout-level offline payment record and clearly show COD as the checkout payment method without an online gateway transaction.
4. **Given** a support or admin user reviews a payment that had retries, **When** they inspect the shared payment, **Then** the payment exposes the stable `ReferenceNo`, linked orders, current/latest attempt, and the full attempt history.

### Edge Cases

- A checkout contains items from multiple sellers and produces more than one order.
- A checkout contains only one order and still uses the same payment flow without special handling.
- The summed order totals do not match the requested payment total because cart, coupon, delivery, or item state changed.
- A single online payment succeeds after some linked orders have already moved into a failed or cancelled checkout state.
- A gateway return or notification is received more than once for the same shared payment.
- A buyer retries after a failed or expired VNPay attempt and receives a new gateway redirect while the payment keeps the same `ReferenceNo`.
- A stale gateway success or failure arrives for an older attempt after a newer attempt has already been created or succeeded.
- A buyer switches from failed/expired VNPay to COD before fulfillment starts.
- Stripe appears in configuration or labels before the gateway is implemented.
- A historical payment or order references a method outside COD, VNPay, or Stripe.
- A newly generated `OrderCode` or payment `ReferenceNo` collides with an existing record and must be regenerated before save.
- A historical order or payment uses an older identifier format and must remain readable and searchable.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST support one checkout-level payment covering one or more generated orders.
- **FR-002**: The system MUST associate every order generated from the same paid checkout with the same payment when an online payment method is used.
- **FR-003**: The system MUST calculate the shared payment amount from the final checkout total across all linked orders, including item totals, discounts, shipping, and fees shown to the buyer.
- **FR-004**: The system MUST reject payment creation when the requested payment amount, currency, method, or linked order set does not match the final checkout state.
- **FR-005**: The system MUST apply a successful shared online payment outcome to every linked order exactly once.
- **FR-006**: The system MUST apply a failed, expired, or cancelled shared online payment outcome consistently to every linked order that depends on that payment.
- **FR-007**: The system MUST keep order ownership and fulfillment separated by order while allowing payment tracking at the checkout level.
- **FR-008**: The system MUST provide a canonical payment method set containing COD, VNPay, and Stripe.
- **FR-009**: The system MUST treat COD as a non-gateway checkout method that creates a checkout-level offline payment record linked to all generated orders without creating an online gateway transaction.
- **FR-010**: The system MUST treat VNPay as an available online payment method for checkout-level payment.
- **FR-011**: The system MUST include Stripe in PaymentService-owned payment method configuration and non-checkout display metadata while hiding or disabling Stripe in buyer checkout selection until Stripe implementation is explicitly enabled later.
- **FR-012**: All buyer, seller, and admin app surfaces that display or select payment methods MUST load method names, availability states, and ordering from PaymentService-owned canonical payment method configuration.
- **FR-013**: The system MUST expose enough payment-to-order traceability for buyer support, seller order review, and admin reconciliation.
- **FR-014**: The system MUST preserve readable historical order and payment records whose method is unknown, disabled, or no longer available.
- **FR-015**: Duplicate payment confirmations for the same shared payment MUST NOT duplicate order payment transitions or user-visible payment success actions.
- **FR-016**: OrderService MUST generate a distinct server-side `OrderCode` for every generated order using the format `ORD-{26-character uppercase ULID}`.
- **FR-017**: PaymentService MUST generate a distinct server-side payment `ReferenceNo` for every checkout-level payment using the format `PAY-{26-character uppercase ULID}`.
- **FR-018**: The system MUST keep payment `ReferenceNo` separate from payment idempotency keys; idempotency keys prevent duplicate command processing and MUST NOT be used as public payment references.
- **FR-019**: A multi-order checkout MUST preserve distinct order codes per order while linking all generated orders to the same checkout-level payment `ReferenceNo`.
- **FR-020**: VNPay payment initiation MUST use payment `ReferenceNo` as the merchant transaction reference, while storing gateway transaction identifiers separately after gateway return or notification.
- **FR-021**: New order code and payment reference generation MUST enforce uniqueness before persistence and retry generation on collision.
- **FR-022**: A checkout-level payment MUST support multiple child payment attempts while keeping the same payment `ReferenceNo` across the checkout.
- **FR-023**: PaymentService MUST create a new `PaymentAttempt` for each buyer retry after a failed, expired, or cancelled VNPay attempt when the linked orders remain eligible and no attempt has succeeded.
- **FR-024**: PaymentService MUST allow switching from a failed, expired, or cancelled VNPay attempt to a COD attempt only before fulfillment starts and before any online attempt succeeds.
- **FR-025**: PaymentService MUST record attempt number, method, gateway, amount, currency, status, gateway transaction identifier, redirect URL when applicable, failure reason, and lifecycle timestamps for every payment attempt.
- **FR-026**: PaymentService MUST include payment attempt identity in payment initiation and payment outcome messages so OrderService can ignore stale outcomes from older attempts.
- **FR-027**: Once any attempt succeeds, the checkout-level payment MUST become succeeded, further retries MUST be rejected, and later callbacks from older attempts MUST NOT change linked order state.
- **FR-028**: Payment read APIs MUST expose current/latest attempt information for buyer-facing displays and full attempt history for support/admin reconciliation.

### Key Entities *(include if feature involves data)*

- **Checkout Payment**: A payment record for one checkout; may cover one or more generated orders and records one stable `ReferenceNo`, amount, currency, status, linked orders, and current/latest attempt.
- **Payment Attempt**: A child record under a checkout payment representing one COD or VNPay payment try, including attempt number, method, gateway, amount, currency, status, gateway transaction data, redirect URL, failure reason, and timestamps.
- **Linked Order Payment Reference**: The relationship between a generated order and the checkout payment that covers it, preserving the order's distinct `OrderCode` and the shared payment `ReferenceNo`.
- **Payment Method**: PaymentService-owned canonical method metadata exposed to apps, including method code, display name, online/offline type, availability, ordering, and future implementation state.
- **Payment Outcome**: The final or intermediate result of a specific payment attempt, including successful, failed, expired, cancelled, or pending states.
- **Order Code**: OrderService-owned public order identifier generated as `ORD-{26-character uppercase ULID}` for support, search, and display.
- **Payment Reference Number**: PaymentService-owned public payment identifier generated as `PAY-{26-character uppercase ULID}` for support, reconciliation, UI/API display, and VNPay merchant transaction reference.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Buyers can complete a checkout that produces two or more orders with one VNPay payment action in 95% of valid test attempts.
- **SC-002**: 100% of orders generated by the same successful online checkout show the same payment outcome within the normal checkout completion flow.
- **SC-003**: Duplicate gateway confirmations for the same payment produce no duplicate paid-order transitions in repeated validation tests.
- **SC-004**: Buyer, seller, and admin apps display COD, VNPay, and Stripe from the same PaymentService-owned payment method configuration in all payment-related surfaces reviewed for the feature.
- **SC-005**: Stripe appears as unavailable or future metadata outside checkout but cannot be selected to complete checkout before the later Stripe implementation is enabled.
- **SC-006**: Support or admin users can identify all orders covered by a shared payment from payment or order detail views in under 30 seconds.
- **SC-007**: Every new order created during repeated checkout tests has a unique `ORD-{ULID}` order code, including all orders created from the same checkout.
- **SC-008**: Every new checkout-level payment created during repeated checkout tests has one unique `PAY-{ULID}` reference number that is returned consistently for idempotent retries.
- **SC-009**: VNPay reconciliation data contains the payment `ReferenceNo` used by HiveSpace payment detail views.
- **SC-010**: A buyer can retry a failed or expired VNPay payment at least twice in validation tests while all attempts remain under the same payment `ReferenceNo`.
- **SC-011**: A stale gateway callback from an older attempt never changes orders after a newer attempt has succeeded in repeated validation tests.
- **SC-012**: Admin/support payment detail shows the complete attempt history for a retried payment, including each attempt status and gateway transaction identifier when present.

## Assumptions

- Existing checkout can already split a cart checkout into multiple orders, commonly by seller or store.
- This feature changes payment grouping and method consistency only; it does not implement the Stripe gateway.
- VNPay remains the only available online gateway method in this feature.
- COD remains available for checkout without immediate online payment capture.
- Payment retry is in scope for VNPay and COD only; Stripe remains future/unavailable and has no retry behavior in this feature.
- The stable payment `ReferenceNo` is reused across attempts; gateway transaction identifiers and attempt numbers distinguish individual tries.
- Buyer, seller, and admin apps all need consistent payment method labels wherever payment method data is displayed, sourced from PaymentService-owned payment method configuration.
- Currency handling follows the existing standardized money policy from feature `0010-standardize-money-handling`.
- Existing historical order codes and payments remain valid; this feature changes generation for new records and does not require rewriting old identifiers.
- Clients never submit order codes or payment reference numbers; both are generated server-side by their owning services.
