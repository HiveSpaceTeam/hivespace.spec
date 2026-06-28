# Docs/Catalog Tasks: System Testing Quality Gate

**Repo**: `hivespace.spec`
**Scope**: Planning artifacts and ADR only. No service doc updates (all services are reused supporting services — verification only). No `shared/api-catalog.md` or `shared/event-catalog.md` changes (no new or changed public contracts).

---

## Spec — contracts/

### Update

- [ ] D001 [US1] Update `contracts/quality-gate-report.md` — verify and finalize schema
  - File: `specs/0008-system-testing-quality-gate/contracts/quality-gate-report.md`
  - Verify that schema fields match `data-model.md` entities: `QualityGate` (scope, result, startedAt, completedAt, durationSeconds), `CheckResult` (checkId, name, journey, audience, ownerSurface, status, failureCategory, blockingDecision, rerunCount, summary), `CoverageGap` (journey, reason, riskLevel, resolution), `AcceptedRisk` (checkId, scope, approvingRole, reason, expiresWhen)
  - Verify allowed `status` values: `pass`, `fail`, `not_applicable`, `accepted_risk`, `environment_failure`, `missing_data`, `unstable_check`
  - Verify allowed `failureCategory` values: `product_behavior`, `environment_readiness`, `missing_test_data`, `unstable_check`, `accepted_coverage_gap`
  - Verify allowed `blockingDecision` values: `none`, `merge_blocking`, `release_blocking`, `blocked_until_risk_accepted`
  - Verify sample JSON sections: one passing release gate sample and one failing merge check with accepted risk sample
  - Verify rerun policy section states: max one rerun; inconsistent after rerun → `unstable_check`; consistent failure → `fail`
  - Do not change `shared/api-catalog.md` — this is not a public product API
  - Acceptance: file exists at path; all entity fields from `data-model.md` are represented; both JSON samples are valid JSON

---

## Spec — architecture/decisions/

### Update

- [ ] D002 [US1] Update `ADR-0007-system-testing-quality-gate.md` — accept the decision
  - File: `architecture/decisions/ADR-0007-system-testing-quality-gate.md`
  - Change `Status` field from `Draft` to `Accepted`
  - Add a `## Follow-Up Actions` section (or update the existing Follow-Up section) listing:
    - Backend tasks are in `specs/0008-system-testing-quality-gate/tasks/backend.md`
    - Frontend tasks are in `specs/0008-system-testing-quality-gate/tasks/frontend.md`
    - Quality-gate report contract is at `specs/0008-system-testing-quality-gate/contracts/quality-gate-report.md`
  - Do not change any other ADR section content (Context, Decision, Consequences, Alternatives Considered) unless the plan has changed since the ADR was drafted
  - Acceptance: `Status` reads `Accepted`; Follow-Up section references the task and contract file paths

---

## Verification-Only Scope (no edits)

All eight backend services and three frontend surfaces are **reused supporting services** for this feature. Their service docs and shared catalog entries must not be edited as part of this feature.

| Service / Surface | Action |
| ----------------- | ------ |
| `services/api-gateway/README.md` | Read for context only; no edit |
| `services/identity-service/README.md` | Read for context only; no edit |
| `services/user-service/README.md` | Read for context only; no edit |
| `services/catalog-service/README.md` | Read for context only; no edit |
| `services/media-service/README.md` | Read for context only; no edit |
| `services/order-service/README.md` | Read for context only; no edit |
| `services/payment-service/README.md` | Read for context only; no edit |
| `services/notification-service/README.md` | Read for context only; no edit |
| `shared/api-catalog.md` | Read for context only; no edit |
| `shared/event-catalog.md` | Read for context only; no edit |

If implementation discovers an actual public contract change (new endpoint, changed event, changed saga), that is out of scope and must be treated as scope drift requiring a separate spec.
