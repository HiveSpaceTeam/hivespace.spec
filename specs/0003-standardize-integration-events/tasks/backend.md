# Backend Tasks

## Repository: `../hivespace.microservice`

### Source Orientation

#### Verify

- [ ] B001 [US1] Verify `backend source instructions and refactor blast radius`
  - File: `../hivespace.microservice/AGENTS.md`, `../hivespace.microservice/CLAUDE.md`, `.gitnexus/`
  - Read both backend repo instruction files before editing source.
  - Run GitNexus impact or rename previews before changing shared contract symbols, especially suffixless saga event records that are referenced by multiple consumers and state machines.
  - Do not edit backend symbols until the direct callers/importers are identified and listed in implementation notes.
  - Acceptance: implementation notes identify direct dependents for renamed contract symbols and confirm backend repo guardrails were read.

## Library: `HiveSpace.Infrastructure.Messaging.Shared`

### Update

- [ ] B002 [US1] Update `non-workflow integration event base classes`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Users/*.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Products/*.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Media/*.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/Events/Stores/*.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/IntegrationEvents/*.cs`
  - Ensure these event records derive from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`: `IdentityUserCreatedIntegrationEvent`, `UserCreatedIntegrationEvent`, `UserUpdatedIntegrationEvent`, `UserEmailVerificationRequestedIntegrationEvent`, `UserEmailVerifiedIntegrationEvent`, `StoreCreatedIntegrationEvent`, `StoreUpdatedIntegrationEvent`, `ProductCreatedIntegrationEvent`, `ProductUpdatedIntegrationEvent`, `ProductDeletedIntegrationEvent`, `ProductSkuUpdatedIntegrationEvent`, `MediaAssetProcessedIntegrationEvent`, `PaymentSucceededIntegrationEvent`, `PaymentFailedIntegrationEvent`.
  - Keep existing payload fields and constructors; only add the common base where missing and required `using` directives.
  - Do not rename commands, DTOs, or already-suffixed event types.
  - Acceptance: all listed records compile and expose `EventId` and `OccurredOn` through the `IntegrationEvent` base.

- [ ] B003 [US1] Update `checkout workflow event contract names`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/CheckoutSaga/Events/*.cs`
  - Rename checkout-owned event records and files to: `OrderCreatedIntegrationEvent`, `OrderCreationFailedIntegrationEvent`, `InventoryReservedIntegrationEvent`, `InventoryReservationFailedIntegrationEvent`, `InventoryReleasedIntegrationEvent`, `PaymentInitiatedIntegrationEvent`, `PaymentInitiationFailedIntegrationEvent`, `PaymentTimeoutIntegrationEvent`, `OrderMarkedAsPaidIntegrationEvent`, `MarkOrderAsPaidFailedIntegrationEvent`, `OrderMarkedAsCODIntegrationEvent`, `MarkOrderAsCODFailedIntegrationEvent`, `CouponUsageCommittedIntegrationEvent`, `CommitCouponUsageFailedIntegrationEvent`, `OrderCancelledIntegrationEvent`, `SagaStepExpiredIntegrationEvent`.
  - Make each renamed event derive from `IntegrationEvent` and preserve existing payload fields, including `StockFailureDto` as a DTO-only type.
  - Do not add `IntegrationEvent` suffix to checkout commands such as `ReserveInventory`, `InitiatePayment`, `CancelOrder`, or `CommitCouponUsage`.
  - Acceptance: every checkout event file name and record name ends in `IntegrationEvent`, while checkout command files keep action names.

- [ ] B004 [US2] Update `fulfillment workflow event contract names`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/FulfillmentSaga/Events/*.cs`
  - Rename fulfillment-owned event records and files to: `SellerNewOrderNotifiedIntegrationEvent`, `BuyerNotifiedIntegrationEvent`, `OrderConfirmedBySellerIntegrationEvent`, `OrderRejectedBySellerIntegrationEvent`, `SellerConfirmationExpiredIntegrationEvent`, `InventoryConfirmedIntegrationEvent`, `InventoryConfirmationFailedIntegrationEvent`.
  - Move these renamed records out of the current checkout-owned events folder and into the fulfillment saga/workflow event folder.
  - Make each renamed event derive from `IntegrationEvent` and preserve existing payload fields and correlation fields.
  - Do not rename fulfillment commands `NotifySellerNewOrder`, `NotifyBuyerOrderConfirmed`, `NotifyBuyerOrderCancelled`, or `ConfirmInventory`.
  - Acceptance: fulfillment event records live under the fulfillment saga/workflow event folder, are distinguishable as integration events, and command/DTO semantics remain unchanged.

- [ ] B005 [US2] Move/Rename `workflow contract folders and namespaces`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/CheckoutSaga/**`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/FulfillmentSaga/**`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/WorkflowHandoff/**`
  - Split contracts into exact ownership folders and matching namespaces: checkout commands/events/DTOs under `CheckoutSaga`, fulfillment commands/events under `FulfillmentSaga`, and shared handoff contracts under `WorkflowHandoff`.
  - Place `OrderReadyForFulfillmentIntegrationEvent` in `WorkflowHandoff` and make it derive from `IntegrationEvent`.
  - Move fulfillment commands `NotifySellerNewOrder`, `NotifyBuyerOrderConfirmed`, `NotifyBuyerOrderCancelled`, and `ConfirmInventory` out of checkout-owned folders if they currently live there.
  - Update namespace declarations to match the new folders and keep the project file SDK-style globbing behavior unchanged.
  - Do not move unrelated projection/payment/user/store/product/media events into workflow folders.
  - Acceptance: maintainers can locate checkout-owned, fulfillment-owned, and shared handoff contracts from the folder tree without scanning all saga events.

- [ ] B006 [US2] Update `DTO contract placement`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/CheckoutSaga/Dtos/*.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/**/Dtos/*.cs`
  - Keep `CheckoutCouponSelectionDto`, `StoreCouponSelectionDto`, `DeliveryAddressDto`, `OrderItemDto`, and `StockFailureDto` named as DTOs and outside event naming/base-class policy.
  - If `StockFailureDto` is embedded in a renamed event file today, move it to the appropriate DTO file only if that matches local one-type-per-file practice; otherwise leave it as a DTO record without event inheritance.
  - Do not document DTOs as standalone events or commands.
  - Acceptance: DTO records compile, keep DTO names, and do not derive from `IntegrationEvent`.

- [ ] B007 [US1] Update `shared contract namespace imports`
  - File: `../hivespace.microservice/src/**/*.cs`, `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/**/*.cs`
  - Replace old shared contract namespaces and type names after B003-B006, including all suffixless event references in consumers, state machines, and publisher implementations.
  - Prefer `using` directives over fully qualified type names unless there is a real ambiguity.
  - Do not introduce compatibility aliases or duplicate old-name contracts.
  - Acceptance: `rg -n "IConsumer<|Publish<|RespondAsync|Request<|Schedule"` finds only current contract names for renamed events.

## Service: IdentityService

### Create

- [ ] B008 [US3] Create `identity event publisher abstraction`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Interfaces/Messaging/IIdentityEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Messaging/IdentityEventPublisher.cs`
  - Add a service-owned publisher interface for account-created and email-verification integration events.
  - Implement the publisher using the existing outbox-backed MassTransit publishing infrastructure available in IdentityService.
  - Include publish methods for `IdentityUserCreatedIntegrationEvent`, `UserEmailVerificationRequestedIntegrationEvent`, and `UserEmailVerifiedIntegrationEvent`.
  - Do not move StoreCreated consumer role-propagation behavior into the publisher.
  - Acceptance: application/page handlers can publish IdentityService-owned events without injecting `IPublishEndpoint` directly.

### Update

- [ ] B009 [US3] Update `IdentityService publisher registration and callers`
  - File: `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Extensions/ServiceCollectionExtensions.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/Account/Register/Index.cshtml.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Pages/ExternalLogin/Callback.cshtml.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/AdminIdentity/Commands/CreateAdmin/CreateAdminCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/Commands/SendEmailVerification/SendEmailVerificationCommandHandler.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Core/Features/EmailVerification/Commands/ConfirmEmailVerification/ConfirmEmailVerificationCommandHandler.cs`
  - Register `IIdentityEventPublisher` with the service DI container.
  - Replace direct `IPublishEndpoint` event publishes in listed handlers/pages with the new service-owned publisher.
  - Preserve event payloads, correlation behavior, and existing command/page outcomes.
  - Do not route MassTransit consumers or saga mechanics through this publisher.
  - Acceptance: no IdentityService application/page flow injects `IPublishEndpoint` solely to publish identity-owned integration events.

## Service: MediaService

### Create

- [ ] B010 [US3] Create `media event publisher abstraction`
  - File: `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Core/Interfaces/Messaging/IMediaEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Infrastructure/Messaging/Publishers/MediaEventPublisher.cs`
  - Add a service-owned publisher interface and implementation for `MediaAssetProcessedIntegrationEvent`.
  - Use the existing outbox-backed MassTransit infrastructure available to MediaService Api/Func projects.
  - Keep media processing result fields unchanged: file ID, owner/entity type, URLs, processing status, and timestamps as currently published.
  - Do not move file processing logic into the publisher.
  - Acceptance: media processing code can publish completion through `IMediaEventPublisher`.

### Update

- [ ] B011 [US3] Update `MediaService publisher registration and background caller`
  - File: `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Api/Extensions/ServiceCollectionExtensions.cs`, `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Func/Program.cs`, `../hivespace.microservice/src/HiveSpace.MediaService/HiveSpace.MediaService.Func/Functions/Queue/ImageProcessingFunction.cs`
  - Register the media event publisher in both API and function hosts where dependencies are built.
  - Replace direct `IPublishEndpoint.Publish(new MediaAssetProcessedIntegrationEvent(...))` in `ImageProcessingFunction` with `IMediaEventPublisher`.
  - Preserve retry/observability behavior for failed processing and do not suppress existing exceptions.
  - Acceptance: background image processing publishes media completion through the service-owned publisher.

## Services: Existing Publisher Abstractions

### Update

- [ ] B012 [US3] Update `existing service event publishers`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Application/Interfaces/Messaging/IStoreEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Infrastructure/Messaging/Publishers/StoreEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Application/Interfaces/Messaging/IProductEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Infrastructure/Messaging/Publishers/ProductEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Application/Interfaces/Messaging/IPaymentEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Infrastructure/Messaging/Publishers/PaymentEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Application/Interfaces/Messaging/IOrderEventPublisher.cs`, `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Infrastructure/Messaging/Publishers/OrderEventPublisher.cs`
  - Align imports and event construction with standardized event namespaces and `IntegrationEvent` inheritance.
  - Keep service-owned publisher interfaces as the application/background publication boundary.
  - Do not reroute saga request/response/schedule/continuation messages through these publishers.
  - Acceptance: existing publishers compile and publish only standardized event types.

- [ ] B013 [US3] Update `application/background publish call sites`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/**/*.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/**/*.cs`, `../hivespace.microservice/src/HiveSpace.PaymentService/**/*.cs`, `../hivespace.microservice/src/HiveSpace.OrderService/**/*.cs`
  - Confirm application handlers and background workflows use service-owned publishers for integration events.
  - Replace any direct application/background `IPublishEndpoint.Publish(new ...IntegrationEvent)` remaining outside saga consumers/state machines.
  - Preserve direct MassTransit context publish/respond calls only where they are saga orchestration messages.
  - Acceptance: source search distinguishes service-owned application/background publishing from saga orchestration publishing.

## Services: Consumers, Sagas, and Workflow Participants

### Update

- [ ] B014 [US1] Update `projection and notification event consumers`
  - File: `../hivespace.microservice/src/HiveSpace.UserService/HiveSpace.UserService.Api/Consumers/*.cs`, `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Consumers/**/*.cs`, `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/Consumers/Sync/*.cs`, `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Consumers/**/*.cs`, `../hivespace.microservice/src/HiveSpace.IdentityService/HiveSpace.IdentityService.Api/Consumers/*.cs`
  - Update `IConsumer<T>` generic arguments and imports to standardized event namespaces.
  - Preserve idempotency, not-found observability, and existing projection update behavior.
  - Do not silently return for missing required data while updating consumers.
  - Acceptance: projection and notification consumers compile against standardized event types and retain existing behavior.

- [ ] B015 [US2] Update `CatalogService checkout/fulfillment saga participants`
  - File: `../hivespace.microservice/src/HiveSpace.CatalogService/HiveSpace.CatalogService.Api/Consumers/Saga/Checkout/*.cs`
  - Update `ReserveInventoryConsumer`, `ReleaseInventoryConsumer`, and `ConfirmInventoryConsumer` to publish/respond with standardized event types: `InventoryReservedIntegrationEvent`, `InventoryReservationFailedIntegrationEvent`, `InventoryReleasedIntegrationEvent`, `InventoryConfirmedIntegrationEvent`, and `InventoryConfirmationFailedIntegrationEvent`.
  - Keep command consumers on `ReserveInventory`, `ReleaseInventory`, and `ConfirmInventory`.
  - Do not rename command queue semantics or change inventory reservation/confirmation business logic.
  - Acceptance: CatalogService saga participants emit standardized success/failure events while consuming unchanged commands.

- [ ] B016 [US2] Update `PaymentService checkout saga participant`
  - File: `../hivespace.microservice/src/HiveSpace.PaymentService/HiveSpace.PaymentService.Api/Consumers/Saga/CheckoutSaga/InitiatePaymentConsumer.cs`
  - Update responses/publications from `PaymentInitiated` and `PaymentInitiationFailed` to `PaymentInitiatedIntegrationEvent` and `PaymentInitiationFailedIntegrationEvent`.
  - Keep the consumed command as `InitiatePayment`.
  - Preserve idempotency and gateway URL behavior.
  - Acceptance: PaymentService checkout participant compiles and returns standardized payment initiation events.

- [ ] B017 [US2] Update `OrderService checkout saga state machine`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/Sagas/CheckoutSaga/CheckoutSagaStateMachine.cs`
  - Replace suffixless checkout event references with standardized checkout event names and updated namespaces.
  - Publish `OrderReadyForFulfillmentIntegrationEvent` from the shared handoff group instead of `OrderReadyForFulfillment`.
  - Keep requests/commands such as `CreateOrder`, `ReserveInventory`, `InitiatePayment`, `CancelOrder`, and `ReleaseInventory` as commands.
  - Do not change states, transitions, timeouts, compensation paths, or payment/COD business outcomes.
  - Acceptance: checkout saga transition graph is behaviorally equivalent and compiles with standardized event names.

- [ ] B018 [US2] Update `OrderService fulfillment saga state machine`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/Sagas/FulfillmentSaga/FulfillmentSagaStateMachine.cs`
  - Start/continue fulfillment from `OrderReadyForFulfillmentIntegrationEvent`.
  - Replace suffixless fulfillment event references with standardized names: seller notification, buyer notification, seller confirmation/rejection, seller confirmation timeout, inventory confirmation success/failure.
  - Keep commands `NotifySellerNewOrder`, `NotifyBuyerOrderConfirmed`, `NotifyBuyerOrderCancelled`, and `ConfirmInventory` unchanged.
  - Do not change fulfillment states, seller timeout behavior, compensation behavior, or notification decision flow.
  - Acceptance: fulfillment saga compiles and uses the shared handoff plus fulfillment event names without business behavior changes.

- [ ] B019 [US2] Update `OrderService saga participant consumers`
  - File: `../hivespace.microservice/src/HiveSpace.OrderService/HiveSpace.OrderService.Api/Consumers/Saga/CheckoutSaga/*.cs`
  - Update consumer publishes/responds in `CreateOrderConsumer`, `MarkOrderAsPaidConsumer`, `MarkOrderAsCODConsumer`, `CommitCouponUsageConsumer`, `CancelOrderConsumer`, and related definitions to standardized event names.
  - Keep consumed commands unchanged and preserve queue endpoint definitions.
  - Do not alter order creation, mark-paid, COD, coupon usage, or cancellation rules.
  - Acceptance: OrderService saga participant consumers emit standardized checkout events and compile.

- [ ] B020 [US2] Update `NotificationService fulfillment participants`
  - File: `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Consumers/NotifySellerNewOrderConsumer.cs`, `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Consumers/NotifyBuyerOrderConfirmedConsumer.cs`, `../hivespace.microservice/src/HiveSpace.NotificationService/HiveSpace.NotificationService.Api/Consumers/NotifyBuyerOrderCancelledConsumer.cs`
  - Update continuation publishes from `SellerNewOrderNotified` and `BuyerNotified` to `SellerNewOrderNotifiedIntegrationEvent` and `BuyerNotifiedIntegrationEvent`.
  - Keep consumed notification commands unchanged.
  - Preserve consume-context publishing because these messages are saga continuations, not application/background integration event publication.
  - Acceptance: NotificationService fulfillment participants compile and still publish continuation events through MassTransit consume context.

- [ ] B021 [US1] Remove `old suffixless event contracts and stale imports`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/**/*.cs`, `../hivespace.microservice/src/**/*.cs`
  - Remove old suffixless event files/types after all references are migrated.
  - Remove stale namespace imports for old `CheckoutSaga.Events` locations where the folder split changed.
  - Do not remove command or DTO contracts that intentionally keep non-event names.
  - Acceptance: source search finds no current `public record` declarations for old suffixless event names and no code references to those event types.
