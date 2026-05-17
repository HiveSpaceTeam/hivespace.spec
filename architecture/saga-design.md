# Saga Design: [SagaName]

**Type:** Orchestration — MassTransit StateMachine
**Service:** [service-name]
**State store:** EF Core / PostgreSQL — `[table_name]` table
**Related spec:** [spec.md](./spec.md)

---

## State diagram

```
Initial
└─► [State1]
├─► [State2] ──► Completed (terminal ✓)
└─► [FailState] ──► Failed (terminal ✗)
```

## States

| State       | Meaning                          |
| ----------- | -------------------------------- |
| `Initial`   | Saga started, no steps taken     |
| `Completed` | Terminal — happy path            |
| `Failed`    | Terminal — compensation complete |

## Happy path

1. Trigger: `[Event]` received
2. Step: publish `[Command]` to [service]
3. On `[ResponseEvent]`: [what happens]
4. Terminal: saga → Completed, publish `[FinalEvent]`

## Compensation flows

### [Failure scenario name]

1. `[FailureEvent]` received
2. [Compensation step 1]
3. [Compensation step 2]
4. Saga → Failed
5. User sees: "[Error message]"

### Timeout ([duration])

1. Scheduled timeout fires
2. [What gets rolled back]
3. Saga → Failed

## Idempotency

- Saga keyed on: `CorrelationId = [FieldName]` ([type])
- Duplicate trigger event → existing saga state returned, no reprocessing
- All consumers: Redis dedup key (TTL 24h)

## Sync bridge (if applicable)

[Describe if the API must return something synchronously while the saga runs async]
Pattern: TaskCompletionSource keyed by CorrelationId
Timeout: [duration] before API returns timeout error

## Observability

Every state transition logs: CorrelationId, FromState, ToState, Timestamp, ActorId
Satisfies NFR-OBS-003.

## Integration test coverage

| Scenario           | Expected terminal state |
| ------------------ | ----------------------- |
| Happy path         | Completed               |
| [Failure scenario] | Failed                  |
| Timeout            | Failed                  |
