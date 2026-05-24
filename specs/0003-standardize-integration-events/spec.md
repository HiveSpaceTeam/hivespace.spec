# Feature Specification: Standardize Integration Event Contracts

- **Feature Branch**: `0003-standardize-integration-events`
- **Created**: 2026-05-24
- **Status**: Implemented
- **Implemented**: 2026-05-24
- **Input**: User description: "Standardize all integration event contracts so shared events derive from IntegrationEvent, event names end with IntegrationEvent, checkout and fulfillment saga contracts are split into separate folders, application and background publishes use service-specific event publishers, and durable documentation/catalogs are updated."

## Clarifications

### Session 2026-05-24

- Q: What rollout condition should the spec require before deploying the breaking event-contract renames? -> A: Drain or explicitly clear old broker/outbox messages before deployment.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Consistent Event Contracts (Priority: P1)

As a backend maintainer, I need all cross-service event contracts to follow one naming and base-contract standard so producers, consumers, and documentation use the same language.

**Why this priority**: Contract consistency is the foundation for safe message publishing, consumer discovery, catalog updates, and future workflow changes.

**Independent Test**: Review the shared message contract inventory and confirm every event contract is identifiable as an integration event by name and common event metadata.

**Acceptance Scenarios**:

1. **Given** the shared message contract inventory, **When** a maintainer reviews event contracts, **Then** every event contract uses the `IntegrationEvent` suffix.
2. **Given** any event contract in the shared inventory, **When** it is inspected for common event behavior, **Then** it exposes the standard integration event identity and occurrence metadata.
3. **Given** a non-event command or data transfer contract, **When** naming standards are applied, **Then** the contract is not incorrectly renamed as an event.

---

### User Story 2 - Clear Saga Contract Ownership (Priority: P2)

As a workflow maintainer, I need checkout and fulfillment saga messages separated by workflow responsibility so each saga's commands and events are easy to locate and reason about.

**Why this priority**: The checkout and fulfillment workflows currently share contract groupings, which makes ownership unclear and increases the risk of changing the wrong workflow message.

**Independent Test**: Review the workflow message catalog and confirm checkout-owned, fulfillment-owned, and shared handoff messages are grouped by responsibility without changing business behavior.

**Acceptance Scenarios**:

1. **Given** checkout workflow messages, **When** a maintainer searches for checkout commands and events, **Then** checkout-owned contracts appear in the checkout workflow group.
2. **Given** fulfillment workflow messages, **When** a maintainer searches for fulfillment commands and events, **Then** fulfillment-owned contracts appear in the fulfillment workflow group.
3. **Given** a message that hands work from checkout to fulfillment, **When** a maintainer reviews ownership, **Then** the message appears in a shared workflow group rather than being hidden inside only one saga.

---

### User Story 3 - Standardized Event Publishing (Priority: P3)

As a service maintainer, I need event publishing from application and background workflows to go through service-owned publisher abstractions so publishing rules are centralized and consistent.

**Why this priority**: Centralized publishing keeps event construction, message metadata, and outbox expectations consistent while preserving saga orchestration mechanics.

**Independent Test**: Review event-producing workflows and confirm application/background event publication uses service-owned publishers, while saga request/response orchestration remains unchanged.

**Acceptance Scenarios**:

1. **Given** an application workflow that publishes an integration event, **When** the workflow is reviewed, **Then** event publication is delegated to a service-owned event publisher.
2. **Given** a background workflow that publishes an integration event, **When** the workflow is reviewed, **Then** event publication is delegated to a service-owned event publisher.
3. **Given** a saga step that depends on request, response, schedule, or continuation behavior, **When** publishing rules are applied, **Then** the saga orchestration behavior remains intact.

---

### User Story 4 - Updated Documentation and Catalogs (Priority: P4)

As a planner or implementer, I need the event catalog and affected service documentation to reflect the standardized contract names and workflow ownership so future work starts from accurate references.

**Why this priority**: Documentation is the planning source of truth for HiveSpace message contracts and must not drift from implementation.

**Independent Test**: Compare the event catalog and affected service documentation against the standardized message inventory and confirm names, owners, consumers, and workflow groupings match.

**Acceptance Scenarios**:

1. **Given** the event catalog, **When** a maintainer searches for any standardized event name, **Then** the catalog contains the current name and purpose.
2. **Given** affected service documentation, **When** a maintainer reviews service-owned or consumed events, **Then** the documented names and responsibilities match the standardized contracts.
3. **Given** older suffixless saga event names, **When** durable documentation is searched, **Then** those names no longer appear as current contract names.

### Edge Cases

- Existing command contracts must not be renamed as events merely because they participate in a saga.
- Data transfer objects shared by workflows must remain identifiable as data shapes, not events or commands.
- Cross-saga handoff events must remain discoverable by both the producing and consuming workflow owners.
- The standardization must avoid introducing duplicate event meanings under different names.
- Documentation must distinguish current contract names from historical references in older completed feature specs when those historical files are intentionally left unchanged.
- Deployment must not proceed while old broker or outbox messages using renamed contract names remain pending; they must be drained or explicitly cleared.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST classify shared message contracts into events, commands, and data transfer contracts before applying naming changes.
- **FR-002**: System MUST ensure every shared event contract uses a name ending with `IntegrationEvent`.
- **FR-003**: System MUST ensure every shared event contract includes the standard integration event identity and occurrence metadata.
- **FR-004**: System MUST keep command contracts named as commands or command-like actions and MUST NOT add the `IntegrationEvent` suffix to command contracts.
- **FR-005**: System MUST separate checkout-owned workflow contracts from fulfillment-owned workflow contracts.
- **FR-006**: System MUST place cross-workflow handoff events in a shared workflow grouping that is discoverable by both checkout and fulfillment maintainers.
- **FR-007**: System MUST preserve existing checkout and fulfillment business outcomes while standardizing names and groupings.
- **FR-008**: System MUST route application and background integration event publication through service-owned event publisher abstractions.
- **FR-009**: System MUST preserve saga orchestration behavior for request, response, timeout, schedule, and continuation messages.
- **FR-010**: System MUST update the event catalog with current event names, owners, consumers, and workflow meanings.
- **FR-011**: System MUST update affected service documentation so each service lists current event names for produced and consumed messages.
- **FR-012**: System MUST update backend coding rules so future event contracts and publishers follow the standardized naming, inheritance, and publishing policy.
- **FR-013**: System MUST require a rollout gate that drains or explicitly clears broker and outbox messages using old renamed contract names before deployment.
- **FR-014**: System MUST not introduce new browser-facing capabilities, public endpoints, or business workflow decisions as part of this standardization.
- **FR-015**: System MUST provide a verification record showing that standardized event names are used consistently across current source and durable documentation.

### Key Entities

- **Integration Event Contract**: A cross-service fact published after a state change or workflow transition; identified by standard event metadata and an `IntegrationEvent` name suffix.
- **Command Contract**: A request for a service or workflow participant to perform an action; remains named as an imperative action and is not treated as an integration event.
- **Workflow Handoff Event**: An event that transfers responsibility between checkout and fulfillment workflows and must be visible to both workflow owners.
- **Event Publisher Policy**: The standard that application and background workflows use service-owned publishers for integration event publication while saga orchestration keeps its request/response behavior.
- **Event Catalog Entry**: Durable documentation of a message contract's name, owner, consumers, and purpose.
- **Coding Rule**: Durable backend implementation guidance that prevents future contracts and publishing paths from drifting from the event standard.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of current shared event contracts use the `IntegrationEvent` suffix after the standardization.
- **SC-002**: 100% of current shared event contracts expose standard event identity and occurrence metadata after the standardization.
- **SC-003**: A maintainer can identify whether a workflow message belongs to checkout, fulfillment, or a shared handoff group in under 2 minutes using the current catalog or contract inventory.
- **SC-004**: 0 current command or data transfer contracts are incorrectly documented as integration events.
- **SC-005**: 100% of affected current service documentation pages reference the standardized event names.
- **SC-006**: Backend coding rules describe the standardized event contract and publishing policy in a way a maintainer can find in under 2 minutes.
- **SC-007**: Deployment readiness evidence confirms old renamed contract messages are drained or explicitly cleared before release.
- **SC-008**: Existing checkout and fulfillment acceptance flows remain demonstrably equivalent after the standardization.

## Assumptions

- The standardization is allowed to make breaking message-contract name changes because it targets current source and planning documentation before deployment coordination.
- Compatibility aliases for old suffixless saga event names are out of scope.
- Existing in-flight broker or outbox messages using old contract names will be drained or explicitly cleared before deployment.
- Command names should remain concise action names because workflow grouping provides ownership context.
- No new public HTTP APIs, user-facing behavior, database ownership, or service boundary changes are required.
- Historical completed feature specs may retain old names when they describe previous planning context, but current catalogs and service docs must use standardized names.
