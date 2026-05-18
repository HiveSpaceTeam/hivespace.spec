# Feature Development Workflow

Use this workflow for every new HiveSpace feature, from idea to shipped and
documented.

## 1. Describe The Feature

Start in this repo:

```bash
cd hivespace.spec
```

Describe the feature in plain language:

```text
I want to add [feature]. It should let [user] do [outcome].
```

Before creating files, resolve any obvious business questions.

Command examples use hyphenated names. Codex commands start with `$`; Claude
Code commands start with `/`.

## 2. Create The Spec

Run:

| Agent | Command |
| --- | --- |
| Codex | `$speckit-specify [feature description]` |
| Claude Code | `/speckit-specify [feature description]` |

This creates an active feature under:

```text
specs/NNNN-feature-name/spec.md
```

Review the spec for user stories, acceptance criteria, and business rules.

## 3. Clarify Requirements

Run:

| Agent | Command |
| --- | --- |
| Codex | `$speckit-clarify` |
| Claude Code | `/speckit-clarify` |

Use this step to settle:

- Service ownership
- Whether each service is an owning service, changed supporting service, or reused supporting service
- User roles and permissions
- Edge cases and failure behavior
- Whether a new API is needed
- Whether a new cross-service event is needed
- Whether a saga or compensation path is needed
- Whether an ADR is needed
- Which frontend surfaces are affected

After clarification, approve the business direction before planning.

## 4. Validate The Spec

Run:

| Agent | Command |
| --- | --- |
| Codex | `$speckit-checklist` |
| Claude Code | `/speckit-checklist` |

Fix any missing or unclear requirements before generating the technical plan.

## 5. Create The Technical Plan

Run:

| Agent | Command |
| --- | --- |
| Codex | `$speckit-plan` |
| Claude Code | `/speckit-plan` |

The plan should cover:

- Affected services and ownership boundaries, classified as owning service, changed supporting service, or reused supporting service
- Domain model changes
- Application commands and queries
- Infrastructure changes and migrations
- Public endpoints
- Integration events and consumers
- Saga design when needed
- Frontend apps and shared package impact
- Verification strategy

If the feature introduces or changes a MassTransit saga state machine, create
`specs/NNNN-feature-name/saga-design.md` from the saga template.

Do not create `saga-design.md` for ordinary cross-service events, direct-upload
flows, simple async consumers, or REST workflows that do not add/change saga
state.

If the feature changes service boundaries, data ownership, messaging patterns,
or makes a non-obvious architecture choice, create an ADR under
`architecture/decisions/`.

## 6. Generate Tasks

Run:

| Agent | Command |
| --- | --- |
| Codex | `$speckit-tasks` |
| Claude Code | `/speckit-tasks` |

Review `tasks.md`. It is the implementation blueprint and should be ordered by
dependency, with parallelizable tasks marked clearly.

Recommended quality gate:

| Agent | Command |
| --- | --- |
| Codex | `$speckit-analyze` |
| Claude Code | `/speckit-analyze` |

Use the analysis output to fix inconsistencies across `spec.md`, `plan.md`, and
`tasks.md` before handing work to implementation.

## 7. Update Catalogs

Run:

| Agent | Command |
| --- | --- |
| Codex | `$update-catalog` |
| Claude Code | `/update-catalog` |

Every new public endpoint must be recorded in `shared/api-catalog.md`.
Every new integration event, command, saga message, timeout event, failure event,
or projection event must be recorded in `shared/event-catalog.md`.

Only update existing catalog rows when the actual contract, owner, auth policy,
message semantics, or consumer set changed. Do not rewrite common endpoint/event
descriptions just because a feature reuses them.

Do not create duplicate endpoints or duplicate events with different names.

## 8. Implement Backend

Switch to the backend repo:

```bash
cd ../hivespace.microservice
```

Read that repo's `AGENTS.md` and `CLAUDE.md`, then use the current feature's
`spec.md`, `plan.md`, `tasks.md`, owning/changed supporting service docs, and
catalog references as the implementation scope.

Typical backend story commands, when provided by that repo:

| Agent | Commands |
| --- | --- |
| Codex | `$start-story`, `$done-story` |
| Claude Code | `/start-story`, `/done-story` |

Backend implementation should follow the repo rules: .NET 8, Clean Architecture,
CQRS, Minimal APIs for new feature work, SQL Server, EF Core, MassTransit, and
transactional outbox for outgoing integration messages.

## 9. Implement Frontend

Switch to the frontend repo:

```bash
cd ../hivespace.web
```

Read that repo's `AGENTS.md` and `CLAUDE.md`, then use the current feature's
`spec.md`, `plan.md`, `tasks.md`, affected frontend surface notes, and API
catalog references as the implementation scope.

Typical frontend story commands, when provided by that repo:

| Agent | Commands |
| --- | --- |
| Codex | `$start-story`, `$done-story` |
| Claude Code | `/start-story`, `/done-story` |

Frontend implementation should follow this order:

1. Types
2. Services
3. Stores
4. Components
5. Pages
6. Routes
7. i18n

Always inspect `packages/shared/src` before creating local frontend code.

## 10. Wrap Up

After the feature ships, return to this repo:

```bash
cd ../hivespace.spec
```

Run:

| Agent | Command |
| --- | --- |
| Codex | `$wrap-up NNNN-feature-name` |
| Claude Code | `/wrap-up NNNN-feature-name` |

Wrap-up should:

- Update affected `services/<service-name>/README.md` docs.
- Update only owning service docs or supporting service docs whose contracts or behavior changed.
- Verify `shared/api-catalog.md` and `shared/event-catalog.md`, but edit only new/changed contracts.
- Mark the feature status as done.
- Summarize changed files.
