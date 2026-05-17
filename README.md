# HiveSpace Spec Repository

This repository is the planning and documentation source of truth for HiveSpace.
It does not contain runnable product code. Backend and frontend implementation
live in sibling repositories.

## Repository Map

| Repository | Path | Purpose |
| --- | --- | --- |
| Spec | `hivespace.spec` | Feature specs, architecture docs, service docs, API and event catalogs |
| Backend | `../hivespace.microservice` | .NET 8 backend microservices |
| Frontend | `../hivespace.web` | Vue 3 frontend monorepo |
| Config | `../hivespace.config` | Local and cloud infrastructure configuration |

## What This Repo Is For

Use this repo to:

- Turn feature ideas into clear product specs.
- Clarify business requirements before implementation starts.
- Plan service ownership, API contracts, events, sagas, and ADRs.
- Generate implementation tasks and handoff prompts.
- Keep service docs, API catalog, and event catalog current after shipping.

Do not add runnable backend or frontend product code here.

## Key Files And Folders

| Path | Purpose |
| --- | --- |
| `.specify/memory/constitution.md` | Project rules and non-negotiable architecture principles |
| `architecture/overview.md` | System architecture overview |
| `architecture/decisions/` | Architecture Decision Records |
| `services/_inventory.md` | Service inventory |
| `services/<service-name>/README.md` | Service-specific ownership and behavior docs |
| `shared/api-catalog.md` | Public HTTP and realtime API catalog |
| `shared/event-catalog.md` | Cross-service event, command, saga, and projection catalog |
| `shared/coding-conventions.md` | Backend and frontend coding conventions |
| `shared/glossary.md` | Domain terminology |
| `specs/NNN-feature-name/spec.md` | Feature requirements |
| `specs/NNN-feature-name/plan.md` | Technical implementation plan |
| `specs/NNN-feature-name/tasks.md` | Ordered implementation task list |
| `.specify/feature.json` | Current active feature pointer |
| `AGENTS.md` | Codex instructions |
| `CLAUDE.md` | Claude Code instructions |

`AGENTS.md` and `CLAUDE.md` must stay in sync when workflow or agent rules change.

## New Member Onboarding

Start by reading these files in order:

1. `.specify/memory/constitution.md`
2. `architecture/overview.md`
3. `services/_inventory.md`
4. `shared/api-catalog.md`
5. `shared/event-catalog.md`
6. `shared/coding-conventions.md`
7. `shared/glossary.md`
8. The affected `services/<service-name>/README.md` files for the feature you will touch

Then inspect the implementation source in `../hivespace.microservice` or
`../hivespace.web` before making code changes in those repos.

## Feature Workflow

Follow [FEATURE-WORKFLOW.md](FEATURE-WORKFLOW.md) for the full feature process,
from idea capture through spec, clarify, plan, tasks, implementation handoff,
catalog updates, and wrap-up.

## Command Cheat Sheet

| Command | Purpose |
| --- | --- |
| `/speckit-specify [idea]` | Create or update a feature spec |
| `/speckit-clarify` | Clarify an existing active spec |
| `/speckit-checklist` | Validate requirement completeness |
| `/speckit-plan` | Create technical planning artifacts |
| `/speckit-tasks` | Create ordered implementation tasks |
| `/speckit-analyze` | Check consistency across spec, plan, and tasks |
| `/update-catalog` | Update API and event catalogs |
| `/handoff` | Generate implementation prompts |
| `/wrap-up [NNN-name]` | Update docs after shipping |

## Architecture Rules To Remember

- Each service owns its own data.
- No service may read another service database directly.
- Browser-facing APIs go through the YARP API Gateway under `/api/v1`.
- Cross-service data flows through APIs or integration events, not shared tables.
- Outgoing integration messages use the transactional outbox.
- Consumers must be idempotent.
- Use a MassTransit saga for multi-service workflows with waiting, timeouts, or compensation.
- Write ADRs for service-boundary changes, data-ownership decisions, messaging decisions, or new architecture patterns.
- All user-facing frontend text uses i18n keys with English and Vietnamese resources updated together.

## Existing Workflow Notes

Use this README, [FEATURE-WORKFLOW.md](FEATURE-WORKFLOW.md), `AGENTS.md`, and
`CLAUDE.md` for current command names, repository paths, and required planning
order.
