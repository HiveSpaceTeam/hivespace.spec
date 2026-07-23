# Data Model: Multi-Order Payment Methods

## PaymentService

### Checkout Payment

Aggregate root representing one checkout-level payment that may contain multiple payment attempts.

| Field | Type | Rules |
| --- | --- | --- |
| `Id` | `PaymentId` / `Guid` | Internal identifier. |
| `ReferenceNo` | `string` | Required, unique, generated as `PAY-{26-character uppercase ULID}`. |
| `BuyerId` | `Guid` | Required. |
| `CheckoutCorrelationId` | `Guid` | Required; matches checkout saga correlation. |
| `Amount` | `long` | Required, positive, smallest currency unit. |
| `CurrencyCode` | `string` | Required; validated against PaymentService local currency-policy projection. |
| `CurrentAttemptId` | `Guid?` | Points to the active/latest attempt when one exists. |
| `Status` | `PaymentStatus` | Aggregate status derived from attempts: `Pending`, `Processing`, `Succeeded`, `Failed`, `Expired`, `Cancelled`, plus existing compatible states. |
| `IdempotencyKey` | `string` | Required for creation; not exposed as public reference. |
| `LinkedOrders` | collection | One or more linked order rows. |
| `Attempts` | collection | One or more payment attempt rows for COD/VNPay tries. |

Validation:

- Payment amount and currency must match the final checkout total across linked orders.
- Online `VNPAY` attempts require gateway URL/transaction initiation before `Processing`.
- COD attempts are offline, have no gateway transaction, and can complete the checkout COD path without gateway redirect.
- Duplicate initiation with the same idempotency key returns the existing checkout payment when the request shape matches.
- Gateway return/IPN processing is idempotent by payment reference, attempt identity, and gateway transaction identity.
- Retry after `Failed`, `Expired`, or `Cancelled` creates a new attempt under the same `ReferenceNo` when no attempt has succeeded and linked orders are still eligible.
- Once any attempt succeeds, further attempts are rejected and stale callbacks for older attempts cannot change payment or order state.

### Payment Attempt

Child record representing one try to pay a checkout-level payment.

| Field | Type | Rules |
| --- | --- | --- |
| `Id` | `Guid` | Required attempt identifier used in events and callback correlation. |
| `PaymentId` | `Guid` | Required parent checkout payment. |
| `AttemptNo` | `int` | Required, starts at `1`, increments by one per retry. |
| `MethodCode` | `PaymentMethodCode` | Required; `COD` or `VNPAY` in this feature. |
| `GatewayCode` | `string?` | `VNPAY` for VNPay, null for COD. |
| `Amount` | `long` | Required, equals parent payment amount. |
| `CurrencyCode` | `string` | Required, equals parent payment currency. |
| `Status` | `PaymentAttemptStatus` | `Pending`, `Processing`, `Succeeded`, `Failed`, `Expired`, or `Cancelled`. |
| `GatewayTransactionId` | `string?` | Stored separately from payment `ReferenceNo`; unique when present. |
| `GatewayResponse` | value object / JSON | Stored for diagnostics when available. |
| `RedirectUrl` | `string?` | Present for active VNPay attempts when gateway initiation succeeds. |
| `FailureReasonCode` | `string?` | Machine-readable failure reason when failed/cancelled. |
| `FailureReason` | `string?` | Human-readable diagnostics where safe to expose. |
| `CreatedAt` | `DateTimeOffset` | Required. |
| `ExpiresAt` | `DateTimeOffset?` | Required for online pending/processing attempts. |
| `CompletedAt` | `DateTimeOffset?` | Set when final. |

Validation:

- A payment attempt belongs to exactly one checkout payment.
- Only the current attempt can drive current payment status unless the parent payment is already succeeded.
- Stale callbacks for non-current attempts are recorded for diagnostics but do not publish order-changing outcomes.
- Switching from VNPay to COD is allowed only after a failed, expired, or cancelled attempt and before fulfillment starts.

### Payment Linked Order

Relationship between one checkout payment and one generated order.

| Field | Type | Rules |
| --- | --- | --- |
| `PaymentId` | `Guid` | Required. |
| `OrderId` | `Guid` | Required. |
| `OrderCode` | `string` | Required snapshot from OrderService. |
| `StoreId` | `Guid` | Required for seller/admin traceability. |
| `Amount` | `long` | Required, smallest currency unit. |
| `CurrencyCode` | `string` | Required; must match payment currency for this feature. |
| `StatusSnapshot` | `string?` | Optional read snapshot; OrderService remains order lifecycle truth. |

Validation:

- A payment must have at least one linked order.
- Linked order IDs must be unique within a payment.
- Summed linked order amounts must equal the payment amount.

### Payment Method

Canonical metadata owned by PaymentService and consumed by all apps.

| Field | Type | Rules |
| --- | --- | --- |
| `Code` | `string` | `COD`, `VNPAY`, `STRIPE`; preserve unknown historical values in reads. |
| `DisplayNameKey` | `string` | Stable key or label source for clients. |
| `Kind` | enum | `Offline` or `Online`. |
| `GatewayCode` | `string?` | Null for COD, `VNPAY` for VNPay, `STRIPE` when implemented later. |
| `IsEnabled` | `bool` | True for COD and VNPay in this feature. |
| `IsCheckoutSelectable` | `bool` | True for COD and VNPay; false for Stripe. |
| `Availability` | enum | `Available`, `Unavailable`, `Future`. |
| `SortOrder` | `int` | Required for consistent app ordering. |

## OrderService

### Order Code

Public identifier generated by OrderService for new orders.

| Field | Type | Rules |
| --- | --- | --- |
| `OrderCode` | `string` | Required for new orders, unique, generated as `ORD-{26-character uppercase ULID}`. |

Validation:

- Generated server-side only.
- Check uniqueness before persistence and retry on collision.
- Historical order codes remain readable/searchable.

### Order Payment Summary

Order-owned read/display summary of linked payment state. PaymentService remains the source of truth.

| Field | Type | Rules |
| --- | --- | --- |
| `PaymentId` | `Guid?` | Shared checkout payment ID when known. |
| `PaymentReferenceNo` | `string?` | Shared `PAY-{ULID}` reference when known. |
| `PaymentMethodCode` | `string` | Canonical code used for checkout. |
| `PaymentStatus` | `string?` | Latest outcome known from payment events. |
| `PaymentAttemptId` | `Guid?` | Latest attempt identity accepted by the checkout saga. |
| `PaymentAttemptNo` | `int?` | Latest attempt number accepted by the checkout saga. |

Validation:

- Every order generated by a paid/COD checkout links to the same checkout-level payment.
- Success/failure events update each linked order exactly once and only when the attempt identity matches the saga's current attempt.
- Order lifecycle remains separated by individual order/store.

## State Transitions

### VNPay

```text
Checkout creates orders
  -> PaymentService creates checkout payment with ReferenceNo and attempt 1
  -> VNPay return/IPN confirms attempt 1 Succeeded or Failed/Expired
  -> if failed/expired and still eligible, retry creates attempt 2 under the same ReferenceNo
  -> OrderService applies outcome to all linked orders idempotently
```

### COD

```text
Checkout creates orders
  -> PaymentService creates offline COD attempt under payment ReferenceNo
  -> OrderService marks all linked orders COD idempotently
  -> Fulfillment continues per order/store
```
