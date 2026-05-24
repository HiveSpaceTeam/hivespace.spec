# Implementation Plan: Standardize Integration Event Contracts

**Branch:** `0003-standardize-integration-events`
**Date:** 2026-05-24
**Spec:** [spec.md](spec.md)

> Planning context read: `.specify/memory/constitution.md`, `architecture/overview.md`, `services/_inventory.md`, `shared/api-catalog.md`, `shared/event-catalog.md`, `shared/coding-conventions.md`, `shared/glossary.md`, and affected service docs.

---

## Phase 0 - Research

### Existing context

- [x] All affected `services/<service>/README.md` files read for service boundaries and message ownership.
- [x] `shared/event-catalog.md` checked for existing event names and saga message groupings.
- [x] `shared/api-catalog.md` checked; no public endpoint change is required.
- [x] Backend shared message contract inventory inspected under `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared`.
- [x] Current publisher usage inspected in affected backend services.

### Technical unknowns

- None unresolved. Rollout condition was clarified: old broker and outbox messages using renamed contract names must be drained or explicitly cleared before deployment.

### Research notes

See [research.md](research.md). Key decisions:

- All shared event contracts derive from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`.
- Event contract type names end with `IntegrationEvent`; command and DTO contracts keep command/DTO names.
- Checkout and fulfillment workflow contracts are separated by workflow ownership, with shared handoff events in a shared workflow group.
- Application/background event publishing goes through service-owned publisher abstractions; saga request/response/schedule/continuation behavior remains MassTransit context-driven.

---

## Phase 1 - Architecture & Data Model

### Service placement

This feature is a shared messaging standardization across backend services. OrderService owns checkout and fulfillment saga workflow grouping. Each producing/consuming service owns its local publisher, consumer references, and service documentation updates.

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| IdentityService | Changed supporting service | Produces identity/email verification integration events and consumes store-created projection event. | Update docs and source references for event inheritance and publisher policy. |
| UserService | Changed supporting service | Produces profile/store events and consumes identity/media events. | Update docs and source references for standardized event contracts. |
| CatalogService | Changed supporting service | Produces product/SKU events and consumes store/media events; participates in checkout/fulfillment inventory commands/events. | Update docs and source references for standardized events and workflow grouping. |
| MediaService | Changed supporting service | Background image processing publishes `MediaAssetProcessedIntegrationEvent` directly today. | Add service-owned media event publisher and update docs. |
| OrderService | Owning service | Owns checkout and fulfillment sagas and the workflow contract folder split. | Update docs, saga source references, and event catalog workflow groupings. |
| PaymentService | Changed supporting service | Produces payment outcome events used by checkout saga. | Update docs and source references for standardized namespace/grouping. |
| NotificationService | Changed supporting service | Consumes identity/user events and publishes saga notification continuation events. | Update docs and source references for fulfillment event names. |
| ApiGateway | Reused supporting service | No public HTTP/WebSocket route changes. | Verification-only; no gateway doc or API catalog update. |

### New aggregates

None. This feature changes shared message contracts, namespaces/folders, publishers, consumers, and documentation only. It does not add business data, database tables, or ownership boundaries.

### New data repository interfaces

None. This feature adds service-owned event publisher interfaces only; it does not add data repository interfaces.

### Contract model

See [data-model.md](data-model.md) for the contract classification model and validation rules.

### Integration events

All existing event meanings are retained. No new business events are introduced, but existing suffixless saga events are renamed to the standardized `IntegrationEvent` form and current event contracts that do not inherit from `IntegrationEvent` are updated.

| Event group | Current examples | Target standard |
| ----------- | ---------------- | --------------- |
| Identity/User/Store/Product/Media/Payment events | `UserCreatedIntegrationEvent`, `PaymentSucceededIntegrationEvent` | Keep names; ensure every event derives from `IntegrationEvent`. |
| Checkout workflow events | `OrderCreated`, `InventoryReserved`, `PaymentInitiated`, `OrderMarkedAsPaid` | Rename to `OrderCreatedIntegrationEvent`, `InventoryReservedIntegrationEvent`, `PaymentInitiatedIntegrationEvent`, `OrderMarkedAsPaidIntegrationEvent`, etc. |
| Fulfillment workflow events | `SellerNewOrderNotified`, `BuyerNotified`, `SellerConfirmationExpired` | Rename to `SellerNewOrderNotifiedIntegrationEvent`, `BuyerNotifiedIntegrationEvent`, `SellerConfirmationExpiredIntegrationEvent`, etc. |
| Shared handoff event | `OrderReadyForFulfillment` | Rename to `OrderReadyForFulfillmentIntegrationEvent` and place in shared workflow handoff grouping. |

Detailed contract map: [contracts/standardized-message-contracts.md](contracts/standardized-message-contracts.md).

All outgoing integration events must go through the transactional outbox pattern. Existing common contract meanings are reused; catalog updates are required because names, inheritance, folders, and workflow groupings change. No API catalog update is required.

### API endpoints

No public HTTP or SignalR endpoints are added or changed. `shared/api-catalog.md` requires no update for this feature.

---

## Phase 2 - Implementation Plan

### Backend shared contract layer

- [ ] Read `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md` before source changes.
- [ ] Classify all files in `libs/HiveSpace.Infrastructure.Messaging.Shared` as event, command, or DTO.
- [ ] Make every event contract inherit from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`.
- [ ] Keep command contracts action-named and DTO contracts DTO-named.
- [ ] Split workflow contracts into checkout, fulfillment, and shared handoff folders/namespaces.
- [ ] Rename suffixless saga events to `*IntegrationEvent`.
- [ ] Update all service imports, consumers, sagas, and publishers for new names/namespaces.

### Service publisher policy

- [ ] IdentityService: introduce/register identity-owned event publisher for account and email verification application flows.
- [ ] MediaService: introduce/register media-owned event publisher for background image-processing publication.
- [ ] UserService/CatalogService/PaymentService/OrderService: keep or align existing service-owned publisher abstractions with the new event base class and namespaces.
- [ ] NotificationService: keep saga continuation publishes inside consume context where they are saga responses, but update event names.
- [ ] Do not route MassTransit saga request/response, scheduled, timeout, or continuation mechanics through service application publishers.

### Documentation layer

- [ ] Update `shared/event-catalog.md` with current standardized names, owners, consumers, and checkout/fulfillment/shared handoff groupings.
- [ ] Update affected `services/*/data-and-events.md`, workflow docs, and README planning notes where they mention changed event names.
- [ ] Update `shared/coding-conventions.md` with the durable event contract and publisher policy.
- [ ] Leave `shared/api-catalog.md` unchanged except for a verification note in task evidence if needed.
- [ ] Historical completed feature specs may retain old names as historical context; current catalogs and service docs must not present old names as current contracts.

### Verification

- [ ] Run backend build and source verification after source changes.
- [ ] Search current source and durable docs for suffixless current event names.
- [ ] Verify 100% of event contracts derive from `IntegrationEvent`.
- [ ] Verify no command/DTO contract was renamed as an event.
- [ ] Verify old broker/outbox messages are drained or explicitly cleared before deployment.

### Saga design

No saga design artifact is required. This feature renames and re-groups saga message contracts but preserves the existing checkout and fulfillment MassTransit state-machine behavior, states, timeouts, compensation paths, and business outcomes.

### Architecture decision

ADR required for the messaging standardization and rollout policy:

- [ADR-0002: Standardized Integration Event Contracts](../../architecture/decisions/ADR-0002-standardized-integration-event-contracts.md)

---

## Phase 3 - Frontend Plan

No frontend implementation is planned. The feature adds no browser-facing API, route, UI, i18n, or app behavior changes.

---

## Constitution Compliance Check

- [x] No hard-deletes planned.
- [x] No new money values planned.
- [x] No new persisted IDs planned.
- [x] All outgoing integration events remain MassTransit/outbox-backed.
- [x] No NuGet dependency changes planned; no `Version=` attributes.
- [x] No frontend text or i18n changes planned.
- [x] No public API endpoint changes planned.
- [x] Service boundaries remain unchanged.
- [x] Documentation/catalog changes are scoped to actual message contract and workflow grouping changes.

Post-design check: no constitution violations remain. The only breaking behavior is message type/name compatibility, mitigated by the rollout gate requiring old broker/outbox messages to be drained or explicitly cleared before deployment.
