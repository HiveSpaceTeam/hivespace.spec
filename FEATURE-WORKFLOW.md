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

## 2. Create The Spec

Run:

```text
/speckit-specify [feature description]
```

This creates an active feature under:

```text
specs/NNN-feature-name/spec.md
```

Review the spec for user stories, acceptance criteria, and business rules.

## 3. Clarify Requirements

Run:

```text
/speckit-clarify
```

Use this step to settle:

- Service ownership
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

```text
/speckit-checklist
```

Fix any missing or unclear requirements before generating the technical plan.

## 5. Create The Technical Plan

Run:

```text
/speckit-plan
```

The plan should cover:

- Affected services and ownership boundaries
- Domain model changes
- Application commands and queries
- Infrastructure changes and migrations
- Public endpoints
- Integration events and consumers
- Saga design when needed
- Frontend apps and shared package impact
- Verification strategy

If the feature introduces or changes a multi-service async workflow,
compensation path, timeout, retry-driven workflow, or MassTransit state machine,
create `specs/NNN-feature-name/saga-design.md` from the saga template.

If the feature changes service boundaries, data ownership, messaging patterns,
or makes a non-obvious architecture choice, create an ADR under
`architecture/decisions/`.

## 6. Generate Tasks

Run:

```text
/speckit-tasks
```

Review `tasks.md`. It is the implementation blueprint and should be ordered by
dependency, with parallelizable tasks marked clearly.

Recommended quality gate:

```text
/speckit-analyze
```

Use the analysis output to fix inconsistencies across `spec.md`, `plan.md`, and
`tasks.md` before handing work to implementation.

## 7. Update Catalogs

Run:

```text
/update-catalog
```

Every new public endpoint must be recorded in `shared/api-catalog.md`.
Every new integration event, command, saga message, timeout event, failure event,
or projection event must be recorded in `shared/event-catalog.md`.

Do not create duplicate endpoints or duplicate events with different names.

## 8. Generate Handoff Prompts

Run:

```text
/handoff
```

This produces backend and frontend implementation prompts scoped from the current
feature's `spec.md`, `plan.md`, and `tasks.md`.

Before switching repos, make sure the handoff includes:

- Feature spec, plan, and tasks references
- Affected service docs
- API and event catalog references
- Reminder to follow the target repo's `AGENTS.md` and `CLAUDE.md`

## 9. Implement Backend

Switch to the backend repo:

```bash
cd ../hivespace.microservice
```

Read that repo's `AGENTS.md` and `CLAUDE.md`, then paste the backend handoff.

Typical backend story commands:

```text
/start-story
/done-story
```

Backend implementation should follow the repo rules: .NET 8, Clean Architecture,
CQRS, Minimal APIs for new feature work, SQL Server, EF Core, MassTransit, and
transactional outbox for outgoing integration messages.

## 10. Implement Frontend

Switch to the frontend repo:

```bash
cd ../hivespace.web
```

Read that repo's `AGENTS.md` and `CLAUDE.md`, then paste the frontend handoff.

Typical frontend story commands:

```text
/start-story
/done-story
```

Frontend implementation should follow this order:

1. Types
2. Services
3. Stores
4. Components
5. Pages
6. Routes
7. i18n

Always inspect `packages/shared/src` before creating local frontend code.

## 11. Wrap Up

After the feature ships, return to this repo:

```bash
cd ../hivespace.spec
```

Run:

```text
/wrap-up NNN-feature-name
```

Wrap-up should:

- Update affected `services/<service-name>/README.md` docs.
- Verify `shared/api-catalog.md` and `shared/event-catalog.md`.
- Mark the feature status as done.
- Update `docs-backlog.md` when relevant.
- Summarize changed files.
