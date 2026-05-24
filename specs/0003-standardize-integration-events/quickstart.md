# Quickstart: Standardize Integration Event Contracts

## Scope

Use this quickstart after tasks are generated and before source implementation.

## Prerequisites

1. Read `../hivespace.microservice/AGENTS.md` and `../hivespace.microservice/CLAUDE.md`.
2. Review [contracts/standardized-message-contracts.md](contracts/standardized-message-contracts.md).
3. Confirm there are no pending old broker/outbox messages in the deployment environment, or plan an explicit clear before release.

## Implementation Order

1. Update shared message contracts and namespaces/folders.
2. Update publishers and consumers service by service.
3. Update checkout and fulfillment saga references.
4. Update affected service docs and `shared/event-catalog.md`.
5. Update `shared/coding-conventions.md` with the event/publisher policy.
6. Run backend verification.

## Verification Commands

From `../hivespace.microservice`:

```powershell
dotnet build
```

Useful searches:

```powershell
rg -n "public record .*IntegrationEvent" libs/HiveSpace.Infrastructure.Messaging.Shared -g "*.cs"
rg -n "public record (OrderCreated|InventoryReserved|BuyerNotified|SellerNewOrderNotified|OrderReadyForFulfillment)" libs/HiveSpace.Infrastructure.Messaging.Shared src -g "*.cs"
rg -n "IPublishEndpoint|Publish<|\\.Publish\\(" src -g "*.cs"
```

Expected result:

- All event records end with `IntegrationEvent`.
- All event records derive from `IntegrationEvent`.
- Commands and DTOs keep non-event names.
- Application/background event publishing goes through service-owned publishers.
- Saga request/response/timeout/continuation publishing remains behaviorally equivalent.
- Durable docs and catalogs use current standardized names.
