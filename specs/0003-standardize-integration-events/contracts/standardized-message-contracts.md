# Contract Design: Standardized Message Contracts

## Non-Workflow Integration Events

These event names already use the target suffix. Implementation must ensure each derives from `IntegrationEvent` and update namespaces/usages only where needed.

| Contract | Owner | Required action |
|---|---|---|
| `IdentityUserCreatedIntegrationEvent` | IdentityService | Keep name; ensure derives from `IntegrationEvent`. |
| `UserCreatedIntegrationEvent` | UserService | Keep name; add `IntegrationEvent` base. |
| `UserUpdatedIntegrationEvent` | UserService | Keep name; add `IntegrationEvent` base. |
| `UserEmailVerificationRequestedIntegrationEvent` | IdentityService | Keep name; add `IntegrationEvent` base. |
| `UserEmailVerifiedIntegrationEvent` | IdentityService | Keep name; add `IntegrationEvent` base. |
| `StoreCreatedIntegrationEvent` | UserService | Keep name; ensure derives from `IntegrationEvent`. |
| `StoreUpdatedIntegrationEvent` | UserService | Keep name; ensure derives from `IntegrationEvent`. |
| `ProductCreatedIntegrationEvent` | CatalogService | Keep name; ensure derives from `IntegrationEvent`. |
| `ProductUpdatedIntegrationEvent` | CatalogService | Keep name; ensure derives from `IntegrationEvent`. |
| `ProductDeletedIntegrationEvent` | CatalogService | Keep name; ensure derives from `IntegrationEvent`. |
| `ProductSkuUpdatedIntegrationEvent` | CatalogService | Keep name; ensure derives from `IntegrationEvent`. |
| `MediaAssetProcessedIntegrationEvent` | MediaService | Keep name; ensure derives from `IntegrationEvent`. |
| `PaymentSucceededIntegrationEvent` | PaymentService | Keep name; ensure derives from `IntegrationEvent`. |
| `PaymentFailedIntegrationEvent` | PaymentService | Keep name; ensure derives from `IntegrationEvent`. |

## Checkout Commands

Commands keep their current action names and do not derive from `IntegrationEvent`.

| Contract | Target participant |
|---|---|
| `CheckoutInitiated` | OrderService saga start command/message |
| `CreateOrder` | OrderService |
| `ReserveInventory` | CatalogService |
| `ReleaseInventory` | CatalogService |
| `InitiatePayment` | PaymentService |
| `MarkOrderAsPaid` | OrderService |
| `MarkOrderAsCOD` | OrderService |
| `CommitCouponUsage` | OrderService |
| `CancelOrder` | OrderService |

## Checkout Events

| Current contract | Target contract |
|---|---|
| `OrderCreated` | `OrderCreatedIntegrationEvent` |
| `OrderCreationFailed` | `OrderCreationFailedIntegrationEvent` |
| `InventoryReserved` | `InventoryReservedIntegrationEvent` |
| `InventoryReservationFailed` | `InventoryReservationFailedIntegrationEvent` |
| `InventoryReleased` | `InventoryReleasedIntegrationEvent` |
| `PaymentInitiated` | `PaymentInitiatedIntegrationEvent` |
| `PaymentInitiationFailed` | `PaymentInitiationFailedIntegrationEvent` |
| `PaymentTimeout` | `PaymentTimeoutIntegrationEvent` |
| `OrderMarkedAsPaid` | `OrderMarkedAsPaidIntegrationEvent` |
| `MarkOrderAsPaidFailed` | `MarkOrderAsPaidFailedIntegrationEvent` |
| `OrderMarkedAsCOD` | `OrderMarkedAsCODIntegrationEvent` |
| `MarkOrderAsCODFailed` | `MarkOrderAsCODFailedIntegrationEvent` |
| `CouponUsageCommitted` | `CouponUsageCommittedIntegrationEvent` |
| `CommitCouponUsageFailed` | `CommitCouponUsageFailedIntegrationEvent` |
| `OrderCancelled` | `OrderCancelledIntegrationEvent` |
| `SagaStepExpired` | `SagaStepExpiredIntegrationEvent` |

## Shared Workflow Handoff Events

| Current contract | Target contract | Producer | Consumer |
|---|---|---|---|
| `OrderReadyForFulfillment` | `OrderReadyForFulfillmentIntegrationEvent` | Checkout saga | Fulfillment saga |

## Fulfillment Commands

Commands keep their current action names and do not derive from `IntegrationEvent`.

| Contract | Target participant |
|---|---|
| `NotifySellerNewOrder` | NotificationService |
| `NotifyBuyerOrderConfirmed` | NotificationService |
| `NotifyBuyerOrderCancelled` | NotificationService |
| `ConfirmInventory` | CatalogService |

## Fulfillment Events

| Current contract | Target contract |
|---|---|
| `SellerNewOrderNotified` | `SellerNewOrderNotifiedIntegrationEvent` |
| `BuyerNotified` | `BuyerNotifiedIntegrationEvent` |
| `OrderConfirmedBySeller` | `OrderConfirmedBySellerIntegrationEvent` |
| `OrderRejectedBySeller` | `OrderRejectedBySellerIntegrationEvent` |
| `SellerConfirmationExpired` | `SellerConfirmationExpiredIntegrationEvent` |
| `InventoryConfirmed` | `InventoryConfirmedIntegrationEvent` |
| `InventoryConfirmationFailed` | `InventoryConfirmationFailedIntegrationEvent` |

## DTO Contracts

DTOs keep their DTO names and do not derive from `IntegrationEvent`.

| Contract | Usage |
|---|---|
| `CheckoutCouponSelectionDto` | Checkout coupon selections |
| `StoreCouponSelectionDto` | Store coupon selection entry |
| `DeliveryAddressDto` | Checkout address snapshot |
| `OrderItemDto` | Checkout/order item snapshot |
| `StockFailureDto` | Inventory failure details |
