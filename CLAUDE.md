# hivespace-spec - Agent Context

This file provides guidance to Claude Code when working in this repository.

## Documentation Sync

`AGENTS.md` and `CLAUDE.md` must stay in sync. Any change made to one file must be applied to the other file in the same update so Codex and Claude Code receive the same project rules.

Keep agent-specific differences limited to command syntax and file locations:

| Concern | Codex | Claude Code |
| --- | --- | --- |
| Primary instruction file | `AGENTS.md` | `CLAUDE.md` |
| Spec Kit skills | `.agents/skills/` | `.claude/skills/` |
| Built-in Spec Kit command style | `$speckit-clarify` | `/speckit-clarify` |
| Git extension command style | `$speckit-git-feature` | `/speckit-git-feature` |
| Custom commands | `.agents/skills/<command>/SKILL.md` | `.claude/commands/<command>.md` |

When adding or changing a shared Spec Kit skill, update both `.agents/skills/` and `.claude/skills/`. When changing repo workflow expectations, update both `AGENTS.md` and `CLAUDE.md`.

## Repository Purpose

This is the planning and documentation repo for HiveSpace. No runnable product code lives here.

Implementation lives in sibling repositories:

| Repository | Path | Purpose |
| --- | --- | --- |
| Backend | `../hivespace.microservice` | .NET 8 backend services |
| Frontend | `../hivespace.web` | Vue 3 frontend monorepo |
| Config | `../hivespace.config` | Local/cloud infrastructure configuration |

## On Every Session Start

Do this silently before responding:

1. Read `.specify/memory/constitution.md`
2. Read `shared/event-catalog.md`
3. Read `shared/api-catalog.md`
4. Read `architecture/overview.md`

## Key Locations

| Content | Path |
| --- | --- |
| Project principles | `.specify/memory/constitution.md` |
| Per-feature specs | `specs/NNNN-feature-name/spec.md` |
| Per-feature technical plans | `specs/NNNN-feature-name/plan.md` |
| Per-feature task index | `specs/NNNN-feature-name/tasks.md` |
| Per-feature detailed task lists | `specs/NNNN-feature-name/tasks/` |
| Current feature pointer | `.specify/feature.json` |
| Architecture decisions | `architecture/decisions/` |
| Service documentation | `services/<name>/README.md` |
| Event catalog | `shared/event-catalog.md` |
| API catalog | `shared/api-catalog.md` |
| Domain glossary | `shared/glossary.md` |

## Spec Kit Commands

Use the command name that matches the active agent:

| Purpose | Codex | Claude Code |
| --- | --- | --- |
| Create or update spec | `$speckit-specify` | `/speckit-specify` |
| Clarify an existing spec | `$speckit-clarify` | `/speckit-clarify` |
| Generate checklist | `$speckit-checklist` | `/speckit-checklist` |
| Create implementation plan | `$speckit-plan` | `/speckit-plan` |
| Generate tasks | `$speckit-tasks` | `/speckit-tasks` |
| Analyze artifacts | `$speckit-analyze` | `/speckit-analyze` |
| Execute tasks | `$speckit-implement` | `/speckit-implement` |

Git extension skills are available to both agents. Use `$speckit-git-feature`, `$speckit-git-commit`, `$speckit-git-initialize`, `$speckit-git-remote`, and `$speckit-git-validate` in Codex. Use `/speckit-git-feature`, `/speckit-git-commit`, `/speckit-git-initialize`, `/speckit-git-remote`, and `/speckit-git-validate` in Claude Code.

## Custom Commands

| Purpose | Codex | Claude Code |
| --- | --- | --- |
| Add new events and endpoints to catalogs | `$update-catalog` | `/update-catalog` |
| Update service docs and mark a shipped feature done | `$wrap-up [NNNN-name]` | `/wrap-up [NNNN-name]` |

Codex custom commands live as skills under `.agents/skills/<command>/SKILL.md`. Claude Code custom commands live under `.claude/commands/<command>.md`. Keep paired command content semantically equivalent.

## Planning Rules

Always follow this order:

1. Ask any obvious preliminary business questions before creating a spec.
2. Run specify before clarify because clarify requires an existing active spec.
3. Run clarify before planning.
4. Run checklist before planning.
5. Update catalogs after task generation.
6. Wrap up after the full feature ships.

When referring to a command, use the active agent's syntax from the table above.

## Service Documentation Scope

Classify every service mentioned by a feature before planning, task generation, catalog update, or wrap-up:

| Classification | Meaning | Documentation/catalog rule |
| --- | --- | --- |
| Owning service | Domain data, business rules, workflow, API, or events change for the feature | Update its service docs and any new/changed catalog entries |
| Changed supporting service | A reused service has an actual API, event, validation, workflow, ownership, or behavior change | Update that service's docs and any changed catalog entries |
| Reused supporting service | The feature only consumes an existing common capability | Read for context and verification only; do not update its docs or rewrite common catalog rows |

Example: if UserService stores a buyer avatar reference and consumes existing MediaService upload/event contracts unchanged, update UserService docs only. Do not update MediaService docs or shared media catalog entries.

## When The User Describes A Feature Idea

1. Read the relevant `services/<name>/README.md` files for owning services and supporting services.
2. Ask any blocking preliminary questions that are clear before writing files.
3. Run the specify workflow to create an active feature spec under `specs/`.
4. Run the clarify workflow on that active spec.
5. After the user answers, propose an approach covering services, events, saga needs, ADR needs, and trade-offs.
6. Wait for user approval before planning technical artifacts.
7. Then run checklist, plan, tasks, and catalog update in that order.

## Repo Rules

- This repo is planning-only; do not add runnable product code here.
- Do not create new branches in this repo unless the user explicitly asks. Default to pulling the latest code, making requested updates on the current branch or `master` as directed, then pushing to `master`/remote.
- Every new public endpoint, or actual change to an existing public endpoint, must be added to `shared/api-catalog.md`.
- Every new cross-service event, command, saga message, failure event, timeout event, projection event, or actual change to an existing message contract/consumer set must be added to `shared/event-catalog.md`.
- Do not rewrite common service docs or shared catalog descriptions just because a feature uses an existing common capability.
- Write ADRs under `architecture/decisions/` for service-boundary changes, non-obvious technical decisions, data-ownership decisions, messaging decisions, or new architectural patterns.
- Create feature saga design docs from `.specify/templates/saga-design-template.md` only when a feature introduces or changes a MassTransit saga state machine. Do not create them for ordinary cross-service events, direct-upload flows, simple async consumers, or REST workflows that do not add/change saga state.
- Before implementation work in a source repo, read that repo's `AGENTS.md` and `CLAUDE.md` if both exist.

## Source Repo Preparation Rules

Before switching to `../hivespace.microservice` or `../hivespace.web`:

1. Use the current feature's `spec.md`, `plan.md`, `tasks.md`, and detailed `tasks/` files as the implementation scope.
2. Include owning service docs, changed supporting service docs, and relevant catalog references in the implementation context.
3. Keep backend and frontend work scoped to one coherent story or task group.
4. Follow the target repo's own agent instruction files.

## Config Repo Scope

`../hivespace.config` remains the local/cloud infrastructure source, but feature specs, plans, tasks, and implementation work must not require updates to that repo. Put source-repo runtime settings, appsettings, gateway route config, and frontend environment typing under backend or frontend planning/tasks instead. References to `../hivespace.config` are allowed only for infrastructure context such as starting Docker Compose.

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read `specs/0006-aspire-setup/plan.md`.
<!-- SPECKIT END -->
