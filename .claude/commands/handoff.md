Generate implementation handoff prompts for the planned feature.
Output only the two code blocks below, with no surrounding explanation.

Step 1 - Read context
- Find the active feature from `.specify/feature.json`; if absent, use the most recently modified folder under `specs/`.
- Read `spec.md`, `plan.md`, and `tasks.md`.
- Identify affected backend services, affected frontend surfaces, and the first user story's initial implementation tasks.

Step 2 - Output BACKEND PROMPT

```text
Read these files before we start:
- ../hivespace.spec/.specify/memory/constitution.md
- ../hivespace.spec/shared/event-catalog.md
- ../hivespace.spec/shared/api-catalog.md
- ../hivespace.spec/services/[primary-affected-service]/README.md
- ../hivespace.spec/specs/[NNN-feature-name]/spec.md
- ../hivespace.spec/specs/[NNN-feature-name]/plan.md
- ../hivespace.spec/specs/[NNN-feature-name]/tasks.md
- AGENTS.md
- CLAUDE.md

We are implementing Story [X.X]: [story title] - backend.

Scope for this session:
[extract the first coherent backend task group from tasks.md]

Do not implement unrelated stories. Follow the repo architecture and show any scope ambiguity before editing.
```

Step 3 - Output FRONTEND PROMPT

```text
Read these files before we start:
- ../hivespace.spec/.specify/memory/constitution.md
- ../hivespace.spec/shared/api-catalog.md
- ../hivespace.spec/specs/[NNN-feature-name]/spec.md
- ../hivespace.spec/specs/[NNN-feature-name]/plan.md
- ../hivespace.spec/specs/[NNN-feature-name]/tasks.md
- AGENTS.md
- CLAUDE.md

We are implementing Story [X.X]: [story title] - frontend.

Surface: [Admin / Seller / Buyer]
Mandatory build order: types -> service -> store -> components/pages -> route + i18n

API endpoints this story uses:
[extract relevant endpoints from plan.md]

Start with the types file.
```
