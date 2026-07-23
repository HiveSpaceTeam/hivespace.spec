# Contract: Checkout Payment Saga Messages

## Changed Command: `InitiatePayment`

Producer: OrderService checkout saga

Consumer: PaymentService

Purpose: create one checkout-level payment for all generated orders.

```csharp
public sealed record InitiatePayment(
    Guid CorrelationId,
    Guid BuyerId,
    string MethodCode,
    long Amount,
    string CurrencyCode,
    string IdempotencyKey,
    IReadOnlyList<CheckoutPaymentOrderDto> Orders);

public sealed record CheckoutPaymentOrderDto(
    Guid OrderId,
    string OrderCode,
    Guid StoreId,
    long Amount,
    string CurrencyCode);
```

Validation:

- `Orders` must contain at least one order.
- Linked order IDs must be unique.
- Sum of linked order amounts must equal `Amount`.
- `CurrencyCode` must be enabled in PaymentService local currency-policy projection.
- `MethodCode` must be PaymentService-recognized and checkout-selectable, except historical reads outside initiation.

## Changed Event: `PaymentInitiatedIntegrationEvent`

Producer: PaymentService

Consumer: OrderService checkout saga

```csharp
public sealed record PaymentInitiatedIntegrationEvent(
    Guid CorrelationId,
    Guid PaymentId,
    Guid PaymentAttemptId,
    int AttemptNo,
    string ReferenceNo,
    Guid BuyerId,
    string MethodCode,
    long Amount,
    string CurrencyCode,
    string? RedirectUrl,
    IReadOnlyList<CheckoutPaymentOrderDto> Orders);
```

Rules:

- COD returns no `RedirectUrl`.
- VNPay returns a gateway redirect URL and uses `ReferenceNo` as merchant transaction reference.

## Changed Event: `PaymentInitiationFailedIntegrationEvent`

Producer: PaymentService

Consumer: OrderService checkout saga

```csharp
public sealed record PaymentInitiationFailedIntegrationEvent(
    Guid CorrelationId,
    Guid? PaymentId,
    Guid? PaymentAttemptId,
    int? AttemptNo,
    Guid BuyerId,
    string MethodCode,
    long Amount,
    string CurrencyCode,
    string ReasonCode,
    string? Reason,
    IReadOnlyList<CheckoutPaymentOrderDto> Orders);
```

## Changed Event: `PaymentSucceededIntegrationEvent`

Producer: PaymentService

Consumer: OrderService checkout saga

```csharp
public sealed record PaymentSucceededIntegrationEvent(
    Guid CorrelationId,
    Guid PaymentId,
    Guid PaymentAttemptId,
    int AttemptNo,
    string ReferenceNo,
    string MethodCode,
    long Amount,
    string CurrencyCode,
    string? GatewayTransactionId,
    IReadOnlyList<CheckoutPaymentOrderDto> Orders);
```

## Changed Event: `PaymentFailedIntegrationEvent`

Producer: PaymentService

Consumer: OrderService checkout saga

```csharp
public sealed record PaymentFailedIntegrationEvent(
    Guid CorrelationId,
    Guid PaymentId,
    Guid PaymentAttemptId,
    int AttemptNo,
    string ReferenceNo,
    string MethodCode,
    long Amount,
    string CurrencyCode,
    string FailureType,
    string? Reason,
    IReadOnlyList<CheckoutPaymentOrderDto> Orders);
```

Rules:

- Duplicate success/failure events for the same payment must not duplicate order transitions.
- OrderService applies the outcome only to orders in the event's linked order set and validates correlation plus attempt identity against saga state.
- Outcomes from stale attempts are recorded by PaymentService for diagnostics but must not publish order-changing events after a newer current attempt exists or the payment is already succeeded.
