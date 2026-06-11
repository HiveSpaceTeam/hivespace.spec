# Implementation Plan: [FEATURE_NAME]

**Branch:** [NNNN-feature-name]
**Date:** [DATE]
**Spec:** [link to spec.md]

> Before writing anything in this file, read `.specify/memory/constitution.md`

---

## Phase 0 — Research

### Existing context (read before planning)

- [ ] `services/<owner-or-supporting>/README.md` — existing aggregates, events, endpoints
- [ ] `shared/event-catalog.md` — verify no event name conflicts
- [ ] `shared/api-catalog.md` — verify no endpoint conflicts
- [ ] Existing saga code (if this feature extends a saga)

### Technical unknowns

- [List anything that needs investigation before implementation]

### Research notes

[Agent fills this during /speckit.plan]

---

## Phase 1 — Architecture & Data Model

### Service placement

Which service owns this feature and why?
Reference constitution Article I (service boundaries).

Classify every service mentioned by the feature:

| Service | Classification | Reason | Documentation/catalog action |
| ------- | -------------- | ------ | ---------------------------- |
| [Service] | Owning service / Changed supporting service / Reused supporting service | [Why this service is involved] | [Update docs/catalogs, or verification-only] |

Rules:

- Owning services have feature-owned domain, data, workflow, API, or event changes and may need service doc updates.
- Changed supporting services may need docs/catalog updates only for actual API, event, validation, workflow, ownership, or behavior changes.
- Reused supporting services are context only. Do not update their docs or shared catalog rows when existing contracts are reused unchanged.
- Shared catalogs change only for new contracts or actual changes to existing endpoint/message contract, owner, auth, semantics, or consumer set.

### New aggregates

```csharp
// Example structure
public class Dispute : AggregateRoot<DisputeId>
{
    public OrderId OrderId { get; private set; }
    public DisputeStatus Status { get; private set; }
    // ...
}
```

EF Core table: `disputes` (snake_case)
Migration name: `AddDispute`

### New repository interfaces

```csharp
public interface IDisputeRepository : IRepository<Dispute, DisputeId>
{
    Task<Dispute?> GetByOrderIdAsync(OrderId orderId, CancellationToken ct);
}
```

### Integration events (must go through Outbox)

| Event                           | Topic/Exchange   | Producer      | Consumer(s)          |
| ------------------------------- | ---------------- | ------------- | -------------------- |
| `DisputeOpenedIntegrationEvent` | `dispute.opened` | order-service | notification-service |

If this feature reuses an existing common event unchanged, list it as reused and state that no catalog update is required.

### API endpoints

| Method | Path                        | Auth  | Handler            |
| ------ | --------------------------- | ----- | ------------------ |
| POST   | /api/v1/orders/{id}/dispute | Buyer | OpenDisputeHandler |

If this feature reuses an existing common endpoint unchanged, list it as reused and state that no catalog update is required.

---

## Phase 2 — Implementation Plan

### Layer order (always domain → application → infrastructure → api)

**Domain layer**

- [ ] `Dispute.cs` — aggregate with private setters, static Create(), AddDomainEvent()
- [ ] `DisputeId.cs` — record DisputeId(Guid Value) with ULID factory
- [ ] `DisputeStatus.cs` — enum
- [ ] `DisputeReason.cs` — enum
- [ ] `DisputeOpenedDomainEvent.cs` — domain event
- [ ] `IDisputeRepository.cs` — interface

**Application layer**

- [ ] `OpenDisputeCommand.cs` + `OpenDisputeCommandHandler.cs`
- [ ] `GetDisputeQuery.cs` + `GetDisputeQueryHandler.cs`

**Infrastructure layer**

- [ ] `DisputeConfiguration.cs` — EF Core config (snake_case table + columns)
- [ ] `DisputeRepository.cs` — extends BaseRepository<Dispute, DisputeId>
- [ ] `DisputeOpenedConsumer.cs` — MassTransit consumer with Redis dedup
- [ ] Migration: `AddDispute`

**API layer**

- [ ] `DisputeEndpoints.cs` — static MapDisputeEndpoints() extension
- [ ] `OpenDisputeRequest.cs`, `DisputeDto.cs` — DTOs
- [ ] Registration in Program.cs

### Saga design (if applicable)

Create `saga-design.md` alongside this file from `.specify/templates/saga-design-template.md` only when this feature introduces or changes a MassTransit saga state machine.

Do not create `saga-design.md` for ordinary cross-service events, direct-upload flows, simple async consumers, or REST workflows that do not add/change saga state.

If a saga is needed, the plan must link to `saga-design.md`, and the saga design must define owner service, participants, states, messages, compensation, timeouts, idempotency, and observability.

If no saga is needed: state "No saga required" and explain why the feature does not add or change MassTransit saga state.

### Architecture decision (if applicable)

Create `architecture/decisions/ADR-[NNNN]-[short-slug].md` from `.specify/templates/architecture-decision-template.md` when this feature makes a non-obvious architecture, service-boundary, data-ownership, messaging, or cross-repo decision with meaningful alternatives.

Number ADRs sequentially by checking `architecture/decisions/`, keep status `Draft` during planning, and link the ADR from this plan.

---

## Phase 3 — Frontend Plan

### Surface(s)

- [ ] buyer
- [ ] seller
- [ ] admin

### Files to create (mandatory order: types → service → store → components → view → route → i18n)

| File                          | Notes                            |
| ----------------------------- | -------------------------------- |
| `types/dispute.ts`            | Interfaces + enums               |
| `services/dispute-service.ts` | API calls via shared HTTP client |
| `stores/dispute-store.ts`     | Pinia store                      |
| `components/DisputeForm.vue`  | Check @hivespace/shared first    |
| `views/DisputeView.vue`       | Loading + error states required  |

### i18n keys to add (both en.json and vi.json)

```json
{
  "dispute": {
    "title": "",
    "reason": {
      "damagedItem": "",
      "wrongItem": "",
      "notReceived": "",
      "other": ""
    },
    "status": {
      "open": "",
      "sellerResponded": "",
      "resolved": ""
    }
  }
}
```

---

## Constitution Compliance Check

Before implementation starts, verify all of these:

- [ ] No hard-deletes planned
- [ ] All money values are long
- [ ] All IDs use ULID
- [ ] All events go through MassTransit Outbox
- [ ] No Version= in any new .csproj dependencies
- [ ] Both en.json and vi.json will be updated
- [ ] Frontend text uses $t() keys only
