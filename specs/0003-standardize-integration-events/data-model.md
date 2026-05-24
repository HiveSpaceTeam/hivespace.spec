# Data Model: Standardize Integration Event Contracts

This feature has no persisted domain data model. The model below describes message contract categories and validation rules.

## Integration Event Contract

Fields:

- `EventId: Guid` from `IntegrationEvent`
- `OccurredOn: DateTimeOffset` from `IntegrationEvent`
- Event-specific payload fields already present in the current contract

Validation rules:

- Type name ends with `IntegrationEvent`.
- Type derives from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`.
- Represents a fact that has happened, not a requested action.
- Published through MassTransit with transactional outbox support.
- Listed in `shared/event-catalog.md` with owner, consumers, and purpose.

## Command Contract

Fields:

- Existing command payload fields only.

Validation rules:

- Name remains imperative or command-like, for example `ReserveInventory`, `InitiatePayment`, or `NotifySellerNewOrder`.
- Does not use the `IntegrationEvent` suffix.
- Does not derive from `IntegrationEvent`.
- Used for saga requests/actions, not durable facts.

## Data Transfer Contract

Fields:

- Shape-only payload fields used by commands or events.

Validation rules:

- Name remains DTO-like, for example `DeliveryAddressDto`, `OrderItemDto`, or `CheckoutCouponSelectionDto`.
- Does not derive from `IntegrationEvent`.
- Is not documented as a standalone event or command.

## Workflow Handoff Event

Fields:

- `EventId: Guid`
- `OccurredOn: DateTimeOffset`
- `CorrelationId`
- Existing order/store/payment payload required by the fulfillment saga start.

Validation rules:

- Name is `OrderReadyForFulfillmentIntegrationEvent`.
- Lives in a shared workflow handoff namespace/folder.
- Documented as produced by checkout and consumed by fulfillment.

## Event Publisher Policy

Fields:

- Service-owned publisher interface, for example `IIdentityEventPublisher` or `IMediaEventPublisher`.
- Implementation that constructs standardized integration events.
- Registration in the owning service dependency injection setup.

Validation rules:

- Application and background workflows publish integration events through the service-owned publisher.
- Saga state machine and consume-context continuation publishing remains in MassTransit orchestration code.
- Publisher implementations use the existing outbox-backed MassTransit infrastructure.
