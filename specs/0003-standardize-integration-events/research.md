# Research: Standardize Integration Event Contracts

## Decision: Use `IntegrationEvent` as the required base for all shared events

Rationale: `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent` already provides `EventId` and `OccurredOn`, matching the feature requirement for standard event identity and occurrence metadata. Some current events already derive from it, while several user/profile/product events and all suffixless saga events do not.

Alternatives considered:

| Alternative | Why rejected |
|---|---|
| Keep mixed inheritance | Fails the metadata consistency requirement and makes publisher constraints weaker. |
| Add only an interface | Does not guarantee common serialized metadata unless every event repeats the properties. |
| Create service-specific base events | Duplicates a shared messaging primitive already present in the backend. |

## Decision: Rename only events to `*IntegrationEvent`

Rationale: The suffix cleanly identifies cross-service facts while preserving command and DTO semantics. Current checkout saga command names such as `ReserveInventory` and `InitiatePayment` are imperative requests and should not become events.

Alternatives considered:

| Alternative | Why rejected |
|---|---|
| Rename all saga contracts with `IntegrationEvent` | Incorrectly labels commands and DTOs as facts. |
| Keep saga events suffixless | Fails the standard and keeps workflow events visually inconsistent with projection/payment events. |
| Add aliases for old names | Out of scope; rollout uses broker/outbox drain or explicit clear instead. |

## Decision: Split workflow contracts by checkout, fulfillment, and shared handoff ownership

Rationale: Checkout creates/reserves/pays/coupons; fulfillment starts after order-ready handoff, notifies sellers, waits for seller action, confirms inventory, and notifies buyers. `OrderReadyForFulfillmentIntegrationEvent` must be discoverable by both workflow owners.

Alternatives considered:

| Alternative | Why rejected |
|---|---|
| Keep all saga contracts under `CheckoutSaga` | Hides fulfillment ownership and fails the spec requirement. |
| Put handoff under only checkout | Makes the fulfillment entry contract harder to find. |
| Put handoff under only fulfillment | Hides the checkout producer side. |

## Decision: Use service-owned publishers for application/background event publication

Rationale: CatalogService, UserService, PaymentService, and OrderService already have publisher abstractions. IdentityService and MediaService currently publish directly through MassTransit in application/page/background paths, so those flows should get service-owned publisher abstractions. Saga response/continuation publishing stays in MassTransit consume/state-machine context because it is orchestration, not application event publication.

Alternatives considered:

| Alternative | Why rejected |
|---|---|
| Let all code inject `IPublishEndpoint` directly | Spreads event construction and weakens consistency. |
| Force saga publishes through application publishers | Risks breaking MassTransit correlation, request/response, and continuation behavior. |
| One shared generic publisher per service only | Useful as infrastructure, but service-owned interfaces still document service event intent. |

## Decision: Treat this as a breaking message-contract rename with a deployment gate

Rationale: Contract type/name and namespace changes can strand old broker/outbox messages. The clarified requirement explicitly gates deployment on draining or explicitly clearing old messages.

Alternatives considered:

| Alternative | Why rejected |
|---|---|
| Backward-compatible aliases | Out of scope for the feature. |
| Ignore pending messages | Risks failed consumers and lost workflow continuity. |
| Dual-publish old and new events | Introduces duplicate event meanings, which the catalog rules prohibit. |
