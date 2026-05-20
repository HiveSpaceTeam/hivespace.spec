# Saga Design: [FEATURE / WORKFLOW NAME]

- **Feature**: `[NNNN-feature-name]`
- **Date**: [DATE]
- **Owner service**: [e.g., OrderService]
- **Saga type**: MassTransit StateMachine
- **Status**: Draft

## Purpose

[Describe the long-running business workflow this saga coordinates and why a saga is required.]

## Trigger Conditions

| Trigger | Source | Message/API | Notes |
|---|---|---|---|
| [Initial trigger] | [Service/app] | [Command/event/API] | [Notes] |

## Participants

| Participant | Responsibility | Messages handled |
|---|---|---|
| [Service] | [What it owns in this workflow] | [Commands/events] |

## State Machine

```text
Initial
  -> [State]
  -> [State]
  -> Completed

Failure:
  -> Compensating
  -> Failed
```

## Messages

### Commands Published

| Command | Target | Purpose | Success event | Failure event |
|---|---|---|---|---|
| [Command] | [Service] | [Purpose] | [Event] | [Event] |

### Events Consumed

| Event | Source | Effect |
|---|---|---|
| [Event] | [Service] | [Transition/effect] |

## State Data

| Field | Type | Purpose |
|---|---|---|
| `CorrelationId` | `Guid` | Saga correlation key |

## Compensation

| Failed step | Compensation action | Owner | Notes |
|---|---|---|---|
| [Step] | [Action] | [Service] | [Notes] |

## Timeouts And Retries

| Step/state | Timeout | Retry policy | Timeout event |
|---|---|---|---|
| [Step] | [Duration] | [Policy] | [Event] |

## Idempotency And Consistency

- [How duplicate commands/events are handled]
- [What is committed atomically with outbox messages]
- [Which operations are safe to retry]

## Observability

- Correlation ID: [source]
- Required logs: [state transitions, failures, compensation]
- Health/diagnostics: [queues, dead-letter, metrics]

## Open Questions

- [Question or N/A]
