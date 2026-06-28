# HiveSpace Event Catalog

## Purpose

This catalog records cross-service commands and events. Check it before adding any integration event, command, saga message, or local projection sync.

Implementation source of truth:

```text
../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared
```

Rules:

- Do not create a second event with the same meaning under a different name.
- Integration contracts must not expose domain entities directly.
- Consumers must be idempotent.
- Outgoing integration messages must participate in the transactional outbox.
- If a new event is added in code, add it here in the same change.

## Naming

| Suffix | Meaning |
|---|---|
| `Command` or command-like imperative name | Requests another service or saga participant to perform work |
| `IntegrationEvent` | Cross-service fact published after a domain change |
| `*FailedIntegrationEvent` | Step failure fact that the orchestrating saga must handle |
| `*ExpiredIntegrationEvent` or `*TimeoutIntegrationEvent` | Time-based transition fact |

## Identity, User, and Store Events

| Contract | Owner | Consumers / Purpose |
|---|---|---|
| `IdentityUserReadyIntegrationEvent` | IdentityService | UserService creates the matching profile with the same public user ID once the account is usable |
| `UserCreatedIntegrationEvent` | UserService | Build profile/display user projections in downstream services after profile creation |
| `UserUpdatedIntegrationEvent` | UserService | Refresh profile/display user projections |
| `UserEmailVerificationRequestedIntegrationEvent` | IdentityService | Notification/email delivery |
| `UserEmailVerifiedIntegrationEvent` | IdentityService | Update email-verification projections or notification state |
| `UserOtpChallengeRequestedIntegrationEvent` | IdentityService | NotificationService sends the OTP sign-in email; fields: RecipientEmail, OtpCode, ExpiresAt, Purpose (`SignIn` in v1) |
| `StoreCreatedIntegrationEvent` | UserService | Catalog/Order/Notification store reference projection; IdentityService seller onboarding |
| `StoreUpdatedIntegrationEvent` | UserService | Refresh store reference projections |

## Product and Media Events

| Contract | Owner | Consumers / Purpose |
|---|---|---|
| `ProductCreatedIntegrationEvent` | CatalogService | OrderService product/SKU references and storefront integrations |
| `ProductUpdatedIntegrationEvent` | CatalogService | Refresh product projections |
| `ProductDeletedIntegrationEvent` | CatalogService | Deactivate product projections |
| `ProductSkuUpdatedIntegrationEvent` | CatalogService | Refresh SKU price/availability projections |
| `MediaAssetProcessedIntegrationEvent` | MediaService | Notify owning domain that media processing completed |

## Payment Events

| Contract | Owner | Consumers / Purpose |
|---|---|---|
| `PaymentSucceededIntegrationEvent` | PaymentService | OrderService checkout saga marks payment success |
| `PaymentFailedIntegrationEvent` | PaymentService | OrderService checkout saga fails or compensates payment path |

## Checkout Saga Commands

These contracts coordinate the checkout workflow. OrderService owns the saga, but individual services own the side effects.

| Contract | Requested action | Expected responder |
|---|---|---|
| `CheckoutInitiated` | Start checkout orchestration | OrderService saga |
| `CreateOrder` | Create order records for checkout | OrderService |
| `ReserveInventory` | Reserve SKU inventory | CatalogService |
| `ReleaseInventory` | Release reserved inventory during compensation | CatalogService |
| `InitiatePayment` | Create payment and redirect URL | PaymentService |
| `MarkOrderAsPaid` | Mark order payment status paid | OrderService |
| `MarkOrderAsCOD` | Mark order as cash-on-delivery path | OrderService |
| `CommitCouponUsage` | Commit coupon usage after payment/order success | OrderService |
| `CancelOrder` | Cancel order during compensation | OrderService |

## Checkout Saga Success Events

| Contract | Meaning |
|---|---|
| `OrderCreatedIntegrationEvent` | Order records were created |
| `InventoryReservedIntegrationEvent` | Inventory reservation succeeded |
| `InventoryReleasedIntegrationEvent` | Inventory was released during compensation |
| `PaymentInitiatedIntegrationEvent` | Payment record/link was created |
| `OrderMarkedAsPaidIntegrationEvent` | Order payment status was marked paid |
| `OrderMarkedAsCODIntegrationEvent` | Order was marked for COD processing |
| `CouponUsageCommittedIntegrationEvent` | Coupon usage was committed |
| `OrderCancelledIntegrationEvent` | Order cancellation completed |

## Checkout Saga Failure and Timeout Events

| Contract | Meaning |
|---|---|
| `OrderCreationFailedIntegrationEvent` | Order creation failed |
| `InventoryReservationFailedIntegrationEvent` | Inventory could not be reserved |
| `PaymentInitiationFailedIntegrationEvent` | Payment initiation failed |
| `MarkOrderAsPaidFailedIntegrationEvent` | Mark-paid step failed |
| `MarkOrderAsCODFailedIntegrationEvent` | COD marking failed |
| `CommitCouponUsageFailedIntegrationEvent` | Coupon usage commit failed |
| `PaymentTimeoutIntegrationEvent` | Buyer did not complete payment in time |
| `SagaStepExpiredIntegrationEvent` | A saga request step timed out |

## Shared Workflow Handoff Events

| Contract | Producer | Consumer | Meaning |
|---|---|---|---|
| `OrderReadyForFulfillmentIntegrationEvent` | OrderService checkout saga | OrderService fulfillment saga | Checkout completed and fulfillment can begin |

## Fulfillment Saga Commands

These contracts coordinate fulfillment after checkout has handed off a ready order.

| Contract | Requested action | Expected responder |
|---|---|---|
| `NotifySellerNewOrder` | Notify seller of new order | NotificationService |
| `NotifyBuyerOrderConfirmed` | Notify buyer order confirmed | NotificationService |
| `NotifyBuyerOrderCancelled` | Notify buyer order cancelled | NotificationService |
| `ConfirmInventory` | Finalize inventory after seller confirmation | CatalogService |

## Fulfillment Saga Success Events

| Contract | Meaning |
|---|---|
| `SellerNewOrderNotifiedIntegrationEvent` | Seller notification was sent |
| `BuyerNotifiedIntegrationEvent` | Buyer notification was sent |
| `OrderConfirmedBySellerIntegrationEvent` | Seller accepted fulfillment |
| `OrderRejectedBySellerIntegrationEvent` | Seller rejected fulfillment |
| `InventoryConfirmedIntegrationEvent` | Inventory was finalized |

## Fulfillment Saga Failure and Timeout Events

| Contract | Meaning |
|---|---|
| `SellerConfirmationExpiredIntegrationEvent` | Seller confirmation window expired |
| `InventoryConfirmationFailedIntegrationEvent` | Final inventory confirmation failed |

## Workflow Data Transfer Objects

| Contract | Purpose |
|---|---|
| `CheckoutCouponSelectionDto` | Coupon selections passed through checkout |
| `StoreCouponSelectionDto` | Store-level coupon selection entry passed through checkout |
| `DeliveryAddressDto` | Delivery address snapshot passed through checkout |
| `OrderItemDto` | Item snapshot passed through checkout |
| `StockFailureDto` | Inventory failure details passed in reservation or confirmation failure events |

## Projection Events

Use projection events to copy minimal reference data into another service. Do not make synchronous calls or direct database reads for data a service should own locally.

| Projection | Source event examples | Owned by |
|---|---|---|
| `store_refs` | `StoreCreatedIntegrationEvent`, `StoreUpdatedIntegrationEvent` | CatalogService, OrderService, NotificationService |
| `product_refs` | `ProductCreatedIntegrationEvent`, `ProductUpdatedIntegrationEvent`, `ProductDeletedIntegrationEvent` | OrderService |
| `sku_refs` | `ProductSkuUpdatedIntegrationEvent` | OrderService |
| `user_refs` | `UserCreatedIntegrationEvent`, `UserUpdatedIntegrationEvent` | NotificationService |

## When To Add A New Event

Add an event only when at least one is true:

- Another service needs a durable fact after a successful state change.
- A local projection in another service must be updated.
- A long-running workflow needs an async step, timeout, or compensation path.
- NotificationService needs to deliver a user-visible message caused by another service.

Do not add an event for simple in-process behavior, frontend-only state, or a one-service CRUD operation.
