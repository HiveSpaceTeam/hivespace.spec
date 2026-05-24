# PaymentService Workflows

## Online Payment Initiation

```text
OrderService CheckoutSaga
  -> InitiatePayment
  -> PaymentService creates payment
  -> PaymentService requests gateway URL
  -> PaymentInitiatedIntegrationEvent or PaymentInitiationFailedIntegrationEvent
```

## Gateway Return / IPN

```text
Gateway
  -> /api/v1/payments/vnpay/return or /api/v1/payments/webhook/{gateway}
  -> PaymentService validates gateway payload
  -> payment status changes once
  -> PaymentSucceededIntegrationEvent or PaymentFailedIntegrationEvent
```

## Workflow Rules

- Treat gateway callbacks as potentially duplicated.
- Store raw gateway response where supported for diagnostics.
- Publish payment result events once per effective payment outcome.
- Publish payment integration events through the service-owned payment event publisher outside saga consume-context responses.
- Do not mutate OrderService data directly.
