# Research: System Testing Quality Gate

## Decision: Use repo-local test suites coordinated by a quality-gate contract

**Rationale**: HiveSpace implementation lives in separate backend and frontend repositories. Keeping tests close to the source code lets each repo use its native build graph and instructions, while a shared quality-gate report contract gives maintainers and release owners one decision model.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Centralize all tests in `hivespace.spec` | The spec repo is planning-only and must not contain runnable product code. |
| Add only manual QA checklists | Does not satisfy repeatable pass/fail quality-gate requirements. |
| Require full critical-path checks for every change | Too slow for typical merge readiness and conflicts with clarified impact-based gating. |

## Decision: Extend the existing backend xUnit pattern

**Rationale**: `HiveSpace.PaymentService.Tests` already uses xUnit, FluentAssertions, coverlet, and centrally versioned package references. Reusing that pattern avoids introducing a parallel backend test stack and respects NuGet version governance.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Introduce a different backend test framework | Adds package and style fragmentation without a clear benefit. |
| Use only `dotnet build` | Build checks do not cover the required critical journeys or failure modes. |
| Place all backend tests in one global project | Makes service ownership and impact-based gating harder to maintain. |

## Decision: Add frontend test tooling through pnpm/Turbo workspace scripts

**Rationale**: The frontend repo is a pnpm/Turbo monorepo with three apps and a shared package. Test scripts should be workspace-aware so affected apps and shared package checks can run independently or as a full gate.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Add app-local test commands only | Does not provide a root quality-gate entrypoint or shared package coverage. |
| Use only lint/type-check/build | Existing checks miss behavioral store/service/component expectations required by the spec. |
| Put frontend tests in one app | Violates app boundaries and misses shared package ownership. |

## Decision: Use controlled substitutes for external providers

**Rationale**: The spec explicitly forbids real customer data, real payment credentials, and live customer-impacting notification delivery. Payment, email, blob storage, and realtime behavior need deterministic substitutes, fakes, local emulators, or provider stubs.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Use live sandbox providers everywhere | Increases flakiness and may still require sensitive credentials. |
| Skip external-provider paths | Leaves required critical payment, media, email, and notification behavior uncovered. |
| Mock everything at the UI only | Does not cover service-level idempotency, event, or workflow expectations. |

## Decision: Treat repeated inconsistency as a blocking unstable-check result

**Rationale**: The clarified spec allows one rerun but requires repeated inconsistency to block unless accepted risk is recorded. This prevents unstable checks from silently passing while still allowing transient environment issues to self-correct once.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Block on the first inconsistent result | Too strict for local dependency and infrastructure blips. |
| Allow repeated reruns | Can hide real instability and delay merge/release decisions. |
| Count pass-once as success | Contradicts the quality-gate reliability requirement. |

## Decision: Record accepted risk by scope and approving role

**Rationale**: Maintainers may accept merge risk, while release owners must accept release risk. The quality-gate report must preserve that distinction so bypass decisions remain auditable.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Let any contributor accept risk | Weakens the gate and makes bypass too easy. |
| Require release owner for every merge risk | Slows day-to-day development unnecessarily. |
| Disallow accepted risk entirely | Removes a necessary release-management path for known gaps. |

## Decision: Organize test projects by service with layer-based internal folders

**Rationale**: One test project per service keeps impact-based gating clean — running one project covers exactly one service's checks. Internal folders mirror Clean Architecture layers (`Domain/`, `Application/`, `Consumers/`) so the folder tells you both where the tested code lives and what kind of test it is. No separate `Unit/` or `Integration/` folders are needed because the layer is the test type: `Domain/` is always unit (pure entity logic, no I/O), `Application/` always executes the real handler/query/service unit using the lightest valid pattern (`NSubstitute` for orchestration-only verification, `IClassFixture` with in-memory EF when persistence/query behavior must be observed), and `Consumers/` is always integration with the MassTransit harness. All domain microservices include a `Domain/` folder. The ApiGateway has no domain model or EF Core, so it uses `Routing/` and `Auth/` instead.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| One test project per layer per service | Too many projects; cross-layer fixture sharing becomes awkward; impact-based gating would need to select multiple projects per service. |
| Flat files with no subfolders | Unnavigable at scale; loses the connection between test and the layer it covers. |
| `Unit/` and `Integration/` type-based folders | Redundant with layer knowledge; makes TDD navigation harder (you'd need to know both the type and the layer to find a test). |

## Decision: Exclude ApiGateway from dedicated test coverage

**Rationale**: ApiGateway owns only YARP routing and reverse proxy configuration — no domain model, no EF Core, no aggregates, and no business logic. Dedicated gateway tests would cover only config-level forwarding rules, which are brittle (break on any config rename) and carry no business-behavior value. Public route ownership is verified by `shared/api-catalog.md` (D001). Authorization enforcement is covered per-service: each service's handler-level integration tests assert that unauthorized requests are rejected at the service layer, which is where the enforcement logic actually lives.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Include `HiveSpace.ApiGateway.Tests` with `Routing/` and `Auth/` folders | Tests would assert YARP config mappings, not product behavior. Fragile config assertions provide low signal — a route rename breaks the test without any regression in user journeys. |
| Test auth enforcement at the gateway only | Auth enforcement belongs in service handlers. Testing it at the gateway would miss the actual enforcement layer and require maintaining a fragile `WebApplicationFactory` wrapping YARP. |

## Decision: Use shared fakes as lightweight service contracts (not Pact)

**Rationale**: `HiveSpace.Testing.Shared` fakes define expected provider behavior without a contract testing framework. When a real provider's interface changes, the fake is updated and all dependent service tests that use it fail immediately — the same signal that Pact would give. This gives the core contract-safety benefit with zero additional tooling for an internal monorepo where all services are co-owned and deployed together.

**Alternatives considered**:

| Alternative | Why rejected |
| ----------- | ------------ |
| Pact consumer-driven contract tests | Overhead (broker, publish/verify pipeline) without proportional benefit for services owned by one team and deployed from the same org; adds a third system to maintain. |
| Mock frameworks (Moq, NSubstitute) per test | Stateless mocks require re-specifying behavior in every test; fakes are stateful, shareable, and more realistic — closer to the real provider's observable behavior. |
| No contract mechanism | Provider interface changes would break callers silently; catching regressions would depend entirely on integration or E2E tests. |
