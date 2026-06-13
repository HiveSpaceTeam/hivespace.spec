# Data Model: System Testing Quality Gate

No persisted product data, domain aggregates, database tables, or migrations are introduced by this feature. The following planning entities define the quality-gate concepts that implementation must represent in tests, reports, or workflow metadata.

## QualityGate

Represents a named decision gate used before merge or release.

| Field | Description |
| ----- | ----------- |
| `scope` | The evaluated scope: docs-only, backend service, frontend app, shared package, broad shared change, or release readiness. |
| `requiredChecks` | The checks required for the selected scope, including shared baseline checks for runtime changes. |
| `result` | Overall result: pass, fail, not applicable, accepted risk, environment failure, missing data, or unstable check. |
| `startedAt` / `completedAt` | Timestamps used to evaluate duration goals. |

## CriticalJourney

Represents a buyer, seller, admin, or service capability that must be protected.

| Field | Description |
| ----- | ----------- |
| `audience` | Buyer, seller, admin, contributor, maintainer, stakeholder, or service capability. |
| `name` | Canonical journey name from the spec critical-path list. |
| `ownerSurface` | Backend service, frontend app, shared package, or cross-service workflow covered by the journey. |
| `blockingLevel` | Whether failure blocks merge, release, or both. |

## CheckResult

Represents the outcome of one quality-gate check.

| Field | Description |
| ----- | ----------- |
| `checkId` | Stable identifier for the check within the source repo quality-gate implementation. |
| `journey` | Critical journey or service capability protected by the check. |
| `status` | pass, fail, not applicable, accepted risk, environment failure, missing data, or unstable check. |
| `failureCategory` | Product behavior, environment readiness, missing test data, unstable check, or accepted coverage gap. |
| `blockingDecision` | none, merge-blocking, release-blocking, or blocked-until-risk-accepted. |
| `rerunCount` | Number of confirmation reruns used for inconsistent results; max one before unstable-check blocking. |

## CoverageGap

Represents a required critical journey or capability that lacks reliable verification.

| Field | Description |
| ----- | ----------- |
| `journey` | The uncovered or partially covered critical journey. |
| `reason` | Why coverage is absent or incomplete. |
| `riskLevel` | Merge risk, release risk, or informational. |
| `resolution` | Add check, mark not applicable, or accept risk. |

## AcceptedRisk

Represents an explicit approval to proceed with a known failure or gap.

| Field | Description |
| ----- | ----------- |
| `scope` | Merge or release. |
| `approvingRole` | Maintainer for merge risk; release owner for release risk. |
| `reason` | Short justification for proceeding. |
| `expiresWhen` | The change, release, or review point where the acceptance must be revisited. |

## State Transitions

```text
pending
  -> running
  -> pass
  -> fail
  -> not_applicable
  -> environment_failure
  -> missing_data
  -> accepted_risk
  -> unstable_check

inconsistent
  -> rerun_once
  -> pass | fail | unstable_check

coverage_gap
  -> add_check | not_applicable | accepted_risk
```

Repeated inconsistency after one rerun must transition to `unstable_check` unless the accepted-risk process is used.
