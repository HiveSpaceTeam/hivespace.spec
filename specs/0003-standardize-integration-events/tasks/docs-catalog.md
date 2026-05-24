# Docs And Catalog Tasks

## Shared Catalogs

### Update

- [ ] D001 [US4] Update `event catalog standardized names`
  - File: `shared/event-catalog.md`
  - Replace current suffixless checkout and fulfillment saga event rows with standardized `*IntegrationEvent` names from `specs/0003-standardize-integration-events/contracts/standardized-message-contracts.md`.
  - Keep command rows action-named and DTO rows DTO-named.
  - Split workflow documentation into checkout commands, checkout events, shared handoff events, fulfillment commands, fulfillment events, failure/timeout events, and DTOs as needed for discoverability.
  - Do not add duplicate rows with old and new names for the same event meaning.
  - Acceptance: every current event row uses the standardized name and every command/DTO row keeps non-event naming.

- [ ] D002 [US3] Update `coding conventions messaging policy`
  - File: `shared/coding-conventions.md`
  - Add durable backend messaging rules requiring shared event contracts to end with `IntegrationEvent` and derive from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`.
  - Document that commands and DTOs must not derive from `IntegrationEvent` or use event suffixes.
  - Document that application/background integration event publication goes through service-owned publishers, while saga request/response/timeout/schedule/continuation behavior stays in MassTransit orchestration context.
  - Include the rollout gate requiring old broker/outbox messages using renamed contracts to be drained or explicitly cleared before deployment.
  - Acceptance: a maintainer can find event naming, inheritance, publisher policy, and rollout gate rules in under 2 minutes.

## Service Docs

### Update

- [ ] D003 [US3] Update `IdentityService event docs`
  - File: `services/identity-service/README.md`, `services/identity-service/data-and-events.md`
  - Document that IdentityService publishes `IdentityUserCreatedIntegrationEvent`, `UserEmailVerificationRequestedIntegrationEvent`, and `UserEmailVerifiedIntegrationEvent` through a service-owned identity event publisher.
  - Keep `StoreCreatedIntegrationEvent` listed as consumed for seller role/claim propagation.
  - Do not move profile/store ownership into IdentityService docs.
  - Acceptance: IdentityService docs list current event names and the service-owned publisher policy without boundary drift.

- [ ] D004 [US3] Update `MediaService event docs`
  - File: `services/media-service/README.md`, `services/media-service/data-and-events.md`
  - Document that media background processing publishes `MediaAssetProcessedIntegrationEvent` through a service-owned media event publisher.
  - Keep MediaService ownership limited to upload/media processing state and URLs.
  - Do not describe product/user business association decisions as MediaService-owned.
  - Acceptance: MediaService docs identify the current event name and publisher policy.

- [ ] D005 [US2] Update `CatalogService event docs`
  - File: `services/catalog-service/README.md`, `services/catalog-service/data-and-events.md`
  - Update checkout and fulfillment participation rows to standardized event names: `InventoryReservedIntegrationEvent`, `InventoryReservationFailedIntegrationEvent`, `InventoryReleasedIntegrationEvent`, `InventoryConfirmedIntegrationEvent`, and `InventoryConfirmationFailedIntegrationEvent`.
  - Keep commands `ReserveInventory`, `ReleaseInventory`, and `ConfirmInventory` unchanged.
  - Keep product/store projection event names current and document that product events derive from `IntegrationEvent`.
  - Acceptance: CatalogService docs distinguish inventory commands from inventory integration events.

- [ ] D006 [US3] Update `UserService event docs`
  - File: `services/user-service/README.md`, `services/user-service/data-and-events.md`
  - Confirm published user/store event names remain current and derive from the standard event base.
  - Document that application publishing uses `IStoreEventPublisher` or equivalent service-owned publisher abstraction.
  - Keep `IdentityUserCreatedIntegrationEvent` and `MediaAssetProcessedIntegrationEvent` as consumed events with current purposes.
  - Do not document IdentityService-owned account/email verification state as UserService-owned.
  - Acceptance: UserService docs show current event names and publisher boundary without service-boundary drift.

- [ ] D007 [US2] Update `PaymentService event docs`
  - File: `services/payment-service/README.md`, `services/payment-service/data-and-events.md`, `services/payment-service/workflows.md`
  - Rename payment initiation saga events to `PaymentInitiatedIntegrationEvent` and `PaymentInitiationFailedIntegrationEvent`.
  - Keep command `InitiatePayment` unchanged.
  - Confirm payment result events remain `PaymentSucceededIntegrationEvent` and `PaymentFailedIntegrationEvent`.
  - Do not document PaymentService as mutating OrderService state directly.
  - Acceptance: PaymentService docs distinguish the initiate-payment command from initiation/result events.

- [ ] D008 [US2] Update `OrderService saga docs`
  - File: `services/order-service/README.md`, `services/order-service/data-and-events.md`, `services/order-service/workflows.md`
  - Update consumed and published saga event names to standardized checkout, fulfillment, and shared handoff event names.
  - Document `OrderReadyForFulfillmentIntegrationEvent` as shared handoff from checkout to fulfillment.
  - Keep saga command names unchanged and preserve existing checkout/fulfillment workflow descriptions.
  - Do not introduce new saga states, compensation paths, endpoint changes, or business decisions in docs.
  - Acceptance: OrderService docs describe the same saga behavior using current standardized event names.

- [ ] D009 [US2] Update `NotificationService fulfillment docs`
  - File: `services/notification-service/README.md`, `services/notification-service/data-and-events.md`, `services/notification-service/workflows.md`
  - Update published saga events to `SellerNewOrderNotifiedIntegrationEvent` and `BuyerNotifiedIntegrationEvent`.
  - Keep consumed commands `NotifySellerNewOrder`, `NotifyBuyerOrderConfirmed`, and `NotifyBuyerOrderCancelled` unchanged.
  - Document that fulfillment continuation publishes remain MassTransit consume-context orchestration messages.
  - Do not describe NotificationService as owning order, payment, catalog, or identity business decisions.
  - Acceptance: NotificationService docs distinguish notification commands from standardized saga continuation events.

## Architecture Decisions

### Update

- [ ] D010 [US4] Update `ADR-0002 follow-up`
  - File: `architecture/decisions/ADR-0002-standardized-integration-event-contracts.md`
  - Keep ADR status as `Draft` for this task; do not infer maintainer approval or mark it accepted.
  - Verify the ADR still documents the final decision, rollout gate, consequences, rejected alternatives, and follow-up evidence expectations after backend and docs/catalog changes are complete.
  - Keep the rollout gate and rejected alternatives visible.
  - Acceptance: ADR remains `Draft` and accurately documents the standardized event contract decision, rollout gate, and follow-up evidence expectations.
