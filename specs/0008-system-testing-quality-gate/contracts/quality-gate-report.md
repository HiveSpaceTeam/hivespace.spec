# Contract: Quality Gate Report

This document defines the quality-gate result schema used by backend and frontend quality-gate runners. Both `quality-gate.ps1` (backend) and `scripts/quality-gate.mjs` (frontend) must emit output conforming to this contract. Reviewers and release owners use this schema to make merge and release decisions. This is not a public product API and does not change `shared/api-catalog.md`.

---

## Top-Level Report Shape

```json
{
  "gate": {
    "scope": "backend:OrderService",
    "result": "fail",
    "startedAt": "2026-06-12T08:00:00Z",
    "completedAt": "2026-06-12T08:03:47Z",
    "durationSeconds": 227
  },
  "checkResults": [],
  "coverageGaps": [],
  "acceptedRisks": []
}
```

### `gate` object

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `scope` | string | yes | One of: `docs-only`, `backend:<serviceName>`, `frontend:<appName>`, `shared`, `release` |
| `result` | string | yes | Overall gate result; see allowed values below |
| `startedAt` | ISO-8601 string | yes | When the gate run started |
| `completedAt` | ISO-8601 string | yes | When the gate run completed |
| `durationSeconds` | number | yes | Elapsed seconds; used for merge-readiness duration goal (target: under 1800s) |

**Allowed `gate.result` values**: `pass`, `fail`, `not_applicable`, `accepted_risk`, `environment_failure`, `missing_data`, `unstable_check`

Overall result is the most severe individual check result. Priority order (highest severity first):
`fail` → `unstable_check` → `environment_failure` → `missing_data` → `accepted_risk` → `not_applicable` → `pass`

---

## CheckResult

Each entry in `checkResults` represents the outcome of one quality-gate check.

```json
{
  "checkId": "backend.identity.session-continuity",
  "name": "IdentityService — session continuity",
  "journey": "buyer-authentication-session",
  "audience": "buyer",
  "ownerSurface": "IdentityService",
  "status": "fail",
  "failureCategory": "product_behavior",
  "blockingDecision": "merge_blocking",
  "rerunCount": 0,
  "summary": "SessionTests.LoginWithUnverifiedAccount: expected 401 but received 200"
}
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `checkId` | string | yes | Stable dot-namespaced identifier: `<area>.<service-or-app>.<check-slug>` |
| `name` | string | yes | Human-readable check name for reviewers |
| `journey` | string | yes | Canonical critical journey or service capability name |
| `audience` | string | yes | One of: `buyer`, `seller`, `admin`, `service`, `contributor`, `maintainer` |
| `ownerSurface` | string | yes | Backend service, frontend app, or shared package owning the check |
| `status` | string | yes | See allowed values below |
| `failureCategory` | string | conditional | Required when `status` is `fail`, `environment_failure`, `missing_data`, or `unstable_check` |
| `blockingDecision` | string | yes | See allowed values below |
| `rerunCount` | integer | yes | 0 = first run; 1 = already rerun once; must not exceed 1 |
| `summary` | string | yes | Short explanation of result suitable for review output |

**Allowed `status` values**:
- `pass` — check ran and succeeded
- `fail` — check ran and detected a product behavior regression
- `not_applicable` — check does not apply to the evaluated scope
- `accepted_risk` — check is skipped with recorded approval; see AcceptedRisk entries
- `environment_failure` — check could not run due to missing or misconfigured infrastructure
- `missing_data` — check could not run because required test data was absent
- `unstable_check` — check produced inconsistent results after one rerun

**Allowed `failureCategory` values**:
- `product_behavior` — service or UI logic does not meet its expected outcome
- `environment_readiness` — a required dependency (database, broker, container) is unavailable or misconfigured
- `missing_test_data` — required seed data, fixtures, or provider stubs are absent
- `unstable_check` — check result varies across runs without a deterministic cause
- `accepted_coverage_gap` — the journey has no reliable check yet; decision is recorded in AcceptedRisk

**Allowed `blockingDecision` values**:
- `none` — result does not affect merge or release
- `merge_blocking` — change must not be merged until resolved or accepted
- `release_blocking` — change must not be released until resolved or accepted
- `blocked_until_risk_accepted` — gate is blocked; unblocked only when an AcceptedRisk entry covers this `checkId`

---

## CoverageGap

Represents a critical journey or service capability that lacks reliable verification.

```json
{
  "journey": "seller-coupon-promotion-management",
  "reason": "Coupon/promotion surface not yet implemented in seller app",
  "riskLevel": "merge_risk",
  "resolution": "accept_risk"
}
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `journey` | string | yes | The uncovered or partially covered critical journey name |
| `reason` | string | yes | Why coverage is absent or incomplete |
| `riskLevel` | string | yes | One of: `merge_risk`, `release_risk`, `informational` |
| `resolution` | string | yes | One of: `add_check`, `not_applicable`, `accept_risk` |

---

## AcceptedRisk

Represents an explicit approval to proceed with a known failure or gap.

```json
{
  "checkId": "backend.order.fulfillment-saga",
  "scope": "merge",
  "approvingRole": "maintainer",
  "reason": "Fulfillment saga check is incomplete; buyer-visible checkout flow is unaffected",
  "expiresWhen": "Before 0008 release cut"
}
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `checkId` | string | yes | The `checkId` this approval covers; must match a CheckResult or CoverageGap entry |
| `scope` | string | yes | `merge` (maintainer may approve) or `release` (release owner must approve) |
| `approvingRole` | string | yes | `maintainer` for merge scope; `release_owner` for release scope |
| `reason` | string | yes | Short justification for proceeding despite the known gap or failure |
| `expiresWhen` | string | yes | The change, release, or review point where this acceptance must be revisited |

---

## Status Rules

- `pass` is allowed only when all required checks for the selected scope pass or are `not_applicable`.
- `accepted_risk` is allowed only when every blocking failure or gap has a valid AcceptedRisk entry.
- `environment_failure` must be distinct from `product_behavior` failure and must include a `summary` that identifies the missing or misconfigured dependency.
- `missing_data` must be distinct from `product_behavior` failure and must identify the absent fixture or seed.
- `unstable_check` is blocking after one inconsistent-result rerun unless an AcceptedRisk entry covers the `checkId`.
- Runtime changes always include shared baseline checks regardless of scope.
- `release` scope requires full critical-path coverage or explicit release-owner accepted risk for every gap.

---

## Rerun Policy

1. A check that produces an inconsistent result (different outcome on re-invocation) may be rerun **once** by the runner.
2. `rerunCount` must be incremented to `1` when a rerun is triggered.
3. If the second run also produces an inconsistent result, `status` must be set to `unstable_check` and `blockingDecision` must be `merge_blocking` or `release_blocking` unless an AcceptedRisk entry exists for the `checkId`.
4. A check that consistently fails (same failure on both runs) is never `unstable_check` — it is `fail`.

---

## Sample: Passing Release Gate

```json
{
  "gate": {
    "scope": "release",
    "result": "pass",
    "startedAt": "2026-06-12T10:00:00Z",
    "completedAt": "2026-06-12T10:22:14Z",
    "durationSeconds": 1334
  },
  "checkResults": [
    {
      "checkId": "backend.identity.session-continuity",
      "name": "IdentityService — session continuity",
      "journey": "buyer-authentication-session",
      "audience": "buyer",
      "ownerSurface": "IdentityService",
      "status": "pass",
      "failureCategory": null,
      "blockingDecision": "none",
      "rerunCount": 0,
      "summary": "All SessionTests passed"
    }
  ],
  "coverageGaps": [],
  "acceptedRisks": []
}
```

## Sample: Failing Merge Check with Accepted Gap

```json
{
  "gate": {
    "scope": "backend:OrderService",
    "result": "fail",
    "startedAt": "2026-06-12T11:00:00Z",
    "completedAt": "2026-06-12T11:04:30Z",
    "durationSeconds": 270
  },
  "checkResults": [
    {
      "checkId": "backend.order.checkout-saga",
      "name": "OrderService — checkout saga state transitions",
      "journey": "cart-and-checkout",
      "audience": "buyer",
      "ownerSurface": "OrderService",
      "status": "fail",
      "failureCategory": "product_behavior",
      "blockingDecision": "merge_blocking",
      "rerunCount": 0,
      "summary": "CheckoutSagaTests.PaymentFailedTransitionsToCancelled: expected OrderStatus.Cancelled but received OrderStatus.Pending"
    },
    {
      "checkId": "backend.order.fulfillment-saga",
      "name": "OrderService — fulfillment saga outcomes",
      "journey": "fulfillment-workflow",
      "audience": "service",
      "ownerSurface": "OrderService",
      "status": "accepted_risk",
      "failureCategory": "accepted_coverage_gap",
      "blockingDecision": "blocked_until_risk_accepted",
      "rerunCount": 0,
      "summary": "Coverage gap accepted by maintainer; see acceptedRisks"
    }
  ],
  "coverageGaps": [
    {
      "journey": "fulfillment-workflow",
      "reason": "FulfillmentSaga consumer test stubs not yet wired",
      "riskLevel": "merge_risk",
      "resolution": "accept_risk"
    }
  ],
  "acceptedRisks": [
    {
      "checkId": "backend.order.fulfillment-saga",
      "scope": "merge",
      "approvingRole": "maintainer",
      "reason": "Fulfillment saga check is incomplete; buyer-visible checkout flow is unaffected by this change",
      "expiresWhen": "Before 0008 release cut"
    }
  ]
}
```
