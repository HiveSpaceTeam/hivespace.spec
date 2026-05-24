# ADR-0002: Standardized Integration Event Contracts

- **Status**: Accepted
- **Date**: 2026-05-24
- **Feature**: `0003-standardize-integration-events`
- **Deciders**: Project maintainers

## Context

HiveSpace uses shared MassTransit contracts across projection events, payment outcomes, checkout, and fulfillment. Current contracts are inconsistent: some event names end with `IntegrationEvent`, some saga events are suffixless, and not every event derives from the shared `IntegrationEvent` base that provides event identity and occurrence metadata.

Checkout and fulfillment workflow contracts are also grouped under checkout-oriented folders, which makes ownership unclear. Some application and background flows publish directly through MassTransit instead of service-owned publisher abstractions.

## Decision

Standardize all shared event contracts so they:

- End with `IntegrationEvent`.
- Derive from `HiveSpace.Infrastructure.Messaging.Events.IntegrationEvent`.
- Keep command and DTO contracts out of the event naming/base-class policy.
- Are grouped by checkout, fulfillment, or shared workflow handoff ownership where saga workflows are involved.

Application and background workflows must publish integration events through service-owned publisher abstractions. MassTransit saga request/response, timeout, schedule, and continuation publishing remains in saga or consume-context orchestration code.

Breaking message-contract renames require a rollout gate: old broker and outbox messages using renamed contract names must be drained or explicitly cleared before deployment.

## Consequences

### Positive

- Maintainers can identify event contracts by name and base type.
- Event metadata is consistent across projection, payment, checkout, and fulfillment events.
- Workflow contract ownership is easier to locate.
- Event construction and publishing rules are centralized per service.

### Negative / Trade-offs

- The rename is breaking for in-flight broker/outbox messages.
- Source references, docs, consumers, publishers, and saga state machines must be updated together.
- Historical feature specs may contain old names and need to be treated as historical context.

### Risks

- Pending messages with old contract names could fail after deployment. Mitigation: require broker/outbox drain or explicit clear before release.
- Renaming commands or DTOs accidentally as events could blur workflow semantics. Mitigation: contract classification and verification searches are required.
- Saga correlation behavior could regress if orchestration publishes are moved into application publishers. Mitigation: keep saga request/response/continuation publishes in MassTransit context.

## Alternatives Considered

| Option | Why rejected |
|---|---|
| Keep mixed event naming and inheritance | Does not satisfy consistency, metadata, or catalog requirements. |
| Rename every saga message with `IntegrationEvent` | Incorrectly treats commands and DTOs as facts. |
| Add backward-compatible old-name aliases | Out of scope and risks duplicate event meanings. |
| Move all publishes, including saga continuation publishes, into service publishers | Risks breaking MassTransit correlation and orchestration behavior. |

## Follow-Up

- Shared message contracts in the backend repo were standardized.
- Affected service docs and `shared/event-catalog.md` were updated.
- `shared/coding-conventions.md` documents the durable policy.
- Deployment evidence must record that old renamed broker/outbox messages were drained or explicitly cleared.
- Implementation evidence must confirm shared event inheritance, old-name source/doc searches, publisher policy searches, and the broker/outbox drain-or-clear rollout gate.
