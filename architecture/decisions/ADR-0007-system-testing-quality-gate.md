# ADR-0007: System Testing Quality Gate

- **Status**: Draft
- **Date**: 2026-06-12
- **Feature**: `0008-system-testing-quality-gate`
- **Deciders**: Project maintainers

## Context

HiveSpace spans a planning repo, a backend microservice repo, and a frontend monorepo. The testing feature must protect critical buyer, seller, admin, and service workflows without moving runnable product code into `hivespace.spec`.

The backend already has one xUnit-based `HiveSpace.PaymentService.Tests` project and centralized test package versions. The frontend has pnpm/Turbo lint, type-check, and build scripts, but no test runner scripts. The quality gate must support impact-based checks for normal merge readiness and full critical-path coverage for release readiness.

## Decision

Implement the quality gate as repo-local test suites coordinated by a shared quality-gate report contract:

- Backend tests live in `../hivespace.microservice` and extend the existing xUnit/FluentAssertions/coverlet pattern.
- Frontend tests live in `../hivespace.web` and are wired through pnpm/Turbo workspace scripts.
- `hivespace.spec` owns the planning artifacts and quality-gate result contract only.
- The quality-gate runner/report distinguishes product failures, environment failures, missing data, unstable checks, not-applicable checks, and accepted risk.
- Impact-based merge checks run affected journeys plus shared baseline checks; release readiness runs full critical-path coverage unless release-owner accepted risk is recorded.

## Consequences

### Positive

- Tests stay close to the source code and follow each repo's tooling and guardrails.
- The spec repo remains planning-only.
- Maintainers get one consistent result model across backend and frontend checks.
- Impact-based checks keep merge readiness practical while preserving full release coverage.

### Negative / Trade-offs

- Coordination is split across two source repos, so task generation must assign backend and frontend work carefully.
- A shared report contract must be maintained even though it is not a public product API.
- Frontend test tooling introduces new workspace dependencies and scripts.

### Risks

- Quality-gate scripts could become slow if impact mapping is too broad. Mitigation: keep service/app-specific commands and full release commands separate.
- Unstable checks could erode trust. Mitigation: allow one rerun only, then block unless accepted risk is recorded.
- Accepted risk could become a bypass. Mitigation: require maintainer approval for merge risk and release-owner approval for release risk.

## Alternatives Considered

| Option | Why rejected |
| ------ | ------------ |
| Put all tests in `hivespace.spec` | Violates the planning-only repo rule and separates tests from implementation source. |
| Require full critical-path checks for every runtime change | Too slow for merge readiness and conflicts with the clarified impact-based scope. |
| Use only existing build/lint/type-check commands | Does not validate required critical user journeys, service workflows, or failure categories. |
| Defer frontend test tooling | Leaves buyer, seller, admin, and shared package critical paths below the requested full-stack scope. |
| Dedicated `HiveSpace.ApiGateway.Tests` project | ApiGateway owns only YARP routing and reverse proxy configuration — no domain model, no EF Core, no aggregates, no business logic. Tests would cover only config-level forwarding rules, which are brittle and have no business-behavior value. Route ownership and auth enforcement are validated indirectly through integration tests on the downstream services. |

## Follow-Up

- Generate backend and frontend tasks from `specs/0008-system-testing-quality-gate/plan.md`.
- Keep `shared/api-catalog.md` and `shared/event-catalog.md` unchanged unless implementation reveals an actual product contract change.
