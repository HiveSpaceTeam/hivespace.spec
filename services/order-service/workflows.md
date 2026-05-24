# OrderService Workflows

## Checkout Preview

```text
Buyer app
  -> POST /api/v1/orders/checkout/preview
  -> OrderService reads cart, coupons, projections
  -> returns totals, discounts, delivery/payment information
```

Preview must not reserve inventory, create payment state, or commit coupon usage.

## Checkout Saga

```text
Initial
  -> create order records
  -> reserve inventory
  -> if COD:
       mark order as COD
       clear/commit checkout side effects
       start fulfillment
  -> if online payment:
       initiate payment
       wait for payment success/failure
       mark paid
       commit coupon usage
       start fulfillment
  -> on failure:
       compensate by cancelling order and releasing inventory where needed
```

Checkout success publishes `OrderReadyForFulfillmentIntegrationEvent` as the shared handoff to fulfillment.

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
