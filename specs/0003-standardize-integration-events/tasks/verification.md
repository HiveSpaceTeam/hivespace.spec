# Verification Tasks

## Backend Build And Checks

### Verify

- [ ] V001 [US1] Verify `backend solution build`
  - File: `../hivespace.microservice`
  - Run `dotnet build` from the backend repo root after source changes.
  - Record any baseline warnings separately from new errors; backend instructions currently allow known development warnings but not compile failures.
  - Do not run unrelated formatters or broad cleanup during verification.
  - Acceptance: backend solution builds successfully with standardized contracts.

- [ ] V002 [US1] Verify `shared event inheritance inventory`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/**/*.cs`
  - Search all shared message contract records and confirm every event record ends with `IntegrationEvent`.
  - Confirm every event record derives from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`.
  - Confirm command records and DTO records do not derive from `IntegrationEvent`.
  - Acceptance: 100% of current event contracts meet name and base-class requirements, with 0 command/DTO false positives.

- [ ] V003 [US3] Verify `publisher policy source search`
  - File: `../hivespace.microservice/src/**/*.cs`
  - Search for `IPublishEndpoint`, `Publish<`, and `.Publish(`.
  - Confirm application/background integration event publishing uses service-owned publishers in IdentityService, MediaService, UserService, CatalogService, PaymentService, and OrderService.
  - Confirm remaining direct MassTransit context publishes/responds are saga request/response/timeout/continuation mechanics.
  - Acceptance: publisher-policy evidence lists allowed direct saga orchestration publishes and no disallowed application/background direct event publishes.

- [ ] V004 [US2] Verify `old suffixless saga event names are absent from current source`
  - File: `../hivespace.microservice/libs/HiveSpace.Infrastructure.Messaging.Shared/**/*.cs`, `../hivespace.microservice/src/**/*.cs`
  - Search for old event type declarations and references: `OrderCreated`, `OrderCreationFailed`, `InventoryReserved`, `InventoryReservationFailed`, `InventoryReleased`, `PaymentInitiated`, `PaymentInitiationFailed`, `PaymentTimeout`, `OrderMarkedAsPaid`, `MarkOrderAsPaidFailed`, `OrderMarkedAsCOD`, `MarkOrderAsCODFailed`, `CouponUsageCommitted`, `CommitCouponUsageFailed`, `OrderCancelled`, `SagaStepExpired`, `OrderReadyForFulfillment`, `SellerNewOrderNotified`, `BuyerNotified`, `OrderConfirmedBySeller`, `OrderRejectedBySeller`, `SellerConfirmationExpired`, `InventoryConfirmed`, `InventoryConfirmationFailed`.
  - Treat command names and DTO fields separately so commands like `ReserveInventory` and `NotifySellerNewOrder` remain valid.
  - Do not fail the check for renamed target names that contain the old text plus `IntegrationEvent`.
  - Acceptance: no current source declares or references suffixless event types after the migration.

## Documentation And Catalog Checks

### Verify

- [ ] V005 [US4] Verify `durable docs use standardized event names`
  - File: `shared/event-catalog.md`, `shared/coding-conventions.md`, `services/*/*.md`, `architecture/decisions/ADR-0002-standardized-integration-event-contracts.md`
  - Search durable docs for old suffixless saga event names.
  - Confirm current docs use standardized event names and command/DTO names correctly.
  - Exclude historical completed feature specs from this check unless they are referenced as current documentation.
  - Acceptance: durable planning docs contain no old suffixless names as current contracts.

- [ ] V006 [US4] Verify `API catalog and ApiGateway unchanged`
  - File: `shared/api-catalog.md`, `services/api-gateway/`
  - Confirm no public HTTP or SignalR endpoint changes were introduced for this feature.
  - Confirm `shared/api-catalog.md` did not receive unnecessary endpoint edits.
  - Do not add ApiGateway docs/tasks unless source changes actually changed gateway routing.
  - Acceptance: API catalog remains valid and this feature has no browser-facing contract changes.

- [ ] V007 [US1] Verify `quickstart acceptance scenarios`
  - File: `specs/0003-standardize-integration-events/quickstart.md`, `../hivespace.microservice`
  - Execute or document the quickstart verification searches for shared contract names, old suffixless event names, and publisher policy.
  - Confirm checkout and fulfillment flows are behaviorally equivalent from code review and build evidence.
  - Do not treat compile-only success as enough if searches reveal old current contract names.
  - Acceptance: quickstart evidence satisfies US1-US4 independent test criteria.

## Deployment Readiness

### Verify

- [ ] V008 [US4] Verify `old broker and outbox message rollout gate`
  - File: `specs/0003-standardize-integration-events/quickstart.md`, deployment/runbook evidence for target environment
  - Before deployment, record that old broker and outbox messages using renamed contract names were drained or explicitly cleared.
  - Include RabbitMQ queue/outbox evidence appropriate to the target environment.
  - Do not deploy this breaking contract rename while old messages remain pending.
  - Acceptance: release notes or deployment evidence explicitly confirm the drain/clear gate was satisfied before release.
