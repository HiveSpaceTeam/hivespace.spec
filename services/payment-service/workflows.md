# PaymentService Workflows

## Online Payment Initiation

```text
OrderService CheckoutSaga
  -> InitiatePayment
  -> PaymentService creates payment
  -> PaymentService requests gateway URL
  -> PaymentInitiated or PaymentInitiationFailed
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
- Do not mutate OrderService data directly.
