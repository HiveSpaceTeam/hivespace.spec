# hivespace-spec - Agent Context

This file provides guidance to Claude Code when working in this repository.

## Documentation Sync

`AGENTS.md` and `CLAUDE.md` must stay in sync. Any change made to one file must be applied to the other file in the same update so Codex and Claude Code receive the same project rules.

Keep agent-specific differences limited to command syntax and file locations:

| Concern | Codex | Claude Code |
| --- | --- | --- |
| Primary instruction file | `AGENTS.md` | `CLAUDE.md` |
| Spec Kit skills | `.agents/skills/` | `.claude/skills/` |
| Built-in Spec Kit command style | `/speckit-clarify` | `/speckit-clarify` |
| Git extension command style | `/speckit-git-feature` | `/speckit-git-feature` |
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
| Per-feature specs | `specs/NNN-feature-name/spec.md` |
| Per-feature technical plans | `specs/NNN-feature-name/plan.md` |
| Per-feature task lists | `specs/NNN-feature-name/tasks.md` |
| Current feature pointer | `.specify/feature.json` |
| Architecture decisions | `architecture/decisions/` |
| Service documentation | `services/<name>/README.md` |
| Event catalog | `shared/event-catalog.md` |
| API catalog | `shared/api-catalog.md` |
| Domain glossary | `shared/glossary.md` |
| Documentation backlog | `docs-backlog.md` |

## Spec Kit Commands

Use the command name that matches the active agent:

| Purpose | Codex | Claude Code |
| --- | --- | --- |
| Create or update spec | `/speckit-specify` | `/speckit-specify` |
| Clarify an existing spec | `/speckit-clarify` | `/speckit-clarify` |
| Generate checklist | `/speckit-checklist` | `/speckit-checklist` |
| Create implementation plan | `/speckit-plan` | `/speckit-plan` |
| Generate tasks | `/speckit-tasks` | `/speckit-tasks` |
| Analyze artifacts | `/speckit-analyze` | `/speckit-analyze` |
| Execute tasks | `/speckit-implement` | `/speckit-implement` |

Git extension skills are available to both agents as `/speckit-git-feature`, `/speckit-git-commit`, `/speckit-git-initialize`, `/speckit-git-remote`, and `/speckit-git-validate`.

## Custom Commands

| Command | Purpose |
| --- | --- |
| `/update-catalog` | Add new events and endpoints to catalogs |
| `/handoff` | Generate backend and frontend implementation prompts |
| `/wrap-up [NNN-name]` | Update service docs and mark a shipped feature done |

Codex custom commands live as skills under `.agents/skills/<command>/SKILL.md`. Claude Code custom commands live under `.claude/commands/<command>.md`. Keep paired command content semantically equivalent.

## Planning Rules

Always follow this order:

1. Ask any obvious preliminary business questions before creating a spec.
2. Run specify before clarify because `/speckit-clarify` requires an existing active spec.
3. Run clarify before planning.
4. Run checklist before planning.
5. Update catalogs after task generation.
6. Generate a handoff before switching to an implementation repo.
7. Wrap up after the full feature ships.

When referring to a command, use the active agent's syntax from the table above.

## When The User Describes A Feature Idea

1. Read the relevant `services/<name>/README.md` files for affected services.
2. Ask any blocking preliminary questions that are clear before writing files.
3. Run the specify workflow to create an active feature spec under `specs/`.
4. Run the clarify workflow on that active spec.
5. After the user answers, propose an approach covering services, events, saga needs, ADR needs, and trade-offs.
6. Wait for user approval before planning technical artifacts.
7. Then run checklist, plan, tasks, catalog update, and handoff in that order.

## Repo Rules

- This repo is planning-only; do not add runnable product code here.
- Every new public endpoint must be added to `shared/api-catalog.md`.
- Every new cross-service event, command, saga message, failure event, timeout event, or projection event must be added to `shared/event-catalog.md`.
- Write ADRs under `architecture/decisions/` for service-boundary changes, non-obvious technical decisions, data-ownership decisions, messaging decisions, or new architectural patterns.
- Create feature saga design docs from `.specify/templates/saga-design-template.md` when a feature introduces or changes a multi-service async workflow, compensation path, timeout, retry-driven workflow, or MassTransit state machine.
- Before implementation work in a source repo, read that repo's `AGENTS.md` and `CLAUDE.md` if both exist.

## Source Repo Handoff Rules

Before switching to `../hivespace.microservice` or `../hivespace.web`:

1. Generate a handoff prompt with the current feature's `spec.md`, `plan.md`, and `tasks.md`.
2. Include the affected service docs and catalog references.
3. Keep backend and frontend prompts scoped to one coherent story or task group.
4. Remind the implementation agent to follow that repo's own agent instruction files.

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
<!-- SPECKIT END -->
