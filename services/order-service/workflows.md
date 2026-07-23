# OrderService Workflows

## Checkout Preview

```text
Buyer app
  -> POST /api/v1/orders/checkout/preview
  -> OrderService reads cart, coupons, product/store refs, and local currency-policy projection
  -> rejects mixed, disabled, missing, or stale currency validation state before returning totals
  -> returns totals, discounts, delivery/payment information with explicit money metadata
```

Preview must not reserve inventory, create payment state, or commit coupon usage.

## Checkout Saga

```text
Initial
  -> validate cart, coupon, and payment currency state against local currency-policy projection
  -> create order records with distinct ORD-{ULID} codes
  -> reserve inventory
  -> if COD:
       initiate one checkout-level offline payment
       wait for PaymentService attempt confirmation
       mark linked orders as COD
       clear/commit checkout side effects
       start fulfillment
  -> if online payment:
       initiate one checkout-level payment for all generated orders
       wait for payment success/failure
       validate current attempt identity
       mark linked orders paid
       commit coupon usage
       start fulfillment
  -> on failure:
       compensate by cancelling order and releasing inventory where needed
```

Checkout success publishes `OrderReadyForFulfillmentIntegrationEvent` as the shared handoff to fulfillment.

Payment initiation and payment outcome messages carry the checkout correlation, linked order set, shared payment reference, and current attempt identity. OrderService ignores stale payment outcomes from older attempts after a newer attempt exists or the payment already succeeded.

Currency validation is a guard inside checkout. Checkout-level payment changes from feature `0011` extend the existing saga payment state and messages without moving payment gateway truth into OrderService.

## Fulfillment Saga

```text
Order ready for fulfillment
  -> notify seller
  -> wait for seller confirmation or timeout
  -> if confirmed:
       confirm inventory
       notify buyer
       complete
  -> if rejected or expired:
       compensate
       notify buyer
       fail/cancel as required
```

Fulfillment uses standardized continuation events including `SellerNewOrderNotifiedIntegrationEvent`, `OrderConfirmedBySellerIntegrationEvent`, `OrderRejectedBySellerIntegrationEvent`, `SellerConfirmationExpiredIntegrationEvent`, `InventoryConfirmedIntegrationEvent`, `InventoryConfirmationFailedIntegrationEvent`, and `BuyerNotifiedIntegrationEvent`.

## Saga Rules

- Use MassTransit state machines for multi-service checkout/fulfillment coordination.
- Every saga step must have a success and failure/timeout path where the external service can fail or not respond.
- Compensation must not assume downstream steps succeeded unless their success event was received.
- State transitions should be logged with correlation ID and actor context where available.
