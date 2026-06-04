---
name: "speckit-tasks"
description: "Generate an actionable, dependency-ordered task index and detailed task files for the feature based on available design artifacts."
argument-hint: "Optional task generation constraints"
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "github-spec-kit"
  source: "templates/commands/tasks.md"
user-invocable: true
disable-model-invocation: false
---


## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before tasks generation)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_tasks` key
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
- When constructing slash commands from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` -> `/speckit-git-commit`.
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}
    
    Wait for the result of the hook command before proceeding to the Outline.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

## Outline

1. **Setup**: Run `.specify/scripts/powershell/setup-tasks.ps1 -Json` from repo root and parse FEATURE_DIR, TASKS_TEMPLATE, and AVAILABLE_DOCS list. `FEATURE_DIR` and `TASKS_TEMPLATE` must be absolute paths when provided. `AVAILABLE_DOCS` is a list of document names/relative paths available under `FEATURE_DIR` (for example `research.md` or `contracts/`). For single quotes in args like "I'm Groot", use escape syntax: e.g 'I'\''m Groot' (or double-quote if possible: "I'm Groot").

2. **Load design documents**: Read from FEATURE_DIR:
   - **Required**: plan.md (tech stack, libraries, structure), spec.md (user stories with priorities)
   - **Optional**: data-model.md (entities), contracts/ (interface contracts), research.md (decisions), quickstart.md (test scenarios), saga-design.md (saga states/messages/compensation), linked ADRs under architecture/decisions/ (architecture decisions)
   - Note: Not all projects have all documents. Generate tasks based on what's available.

3. **Execute task generation workflow**:
   - Load plan.md and extract tech stack, libraries, project structure, source repositories, services, apps, packages, and verification commands
   - Load spec.md and extract user stories with their priorities (P1, P2, P3, etc.) for traceability labels
   - If data-model.md exists: Extract entities and map them to owning services/libs and user story labels
   - If contracts/ exists: Map interface contracts to owning services/apps and user story labels
   - If research.md exists: Extract decisions that affect setup, implementation constraints, or verification
   - If saga-design.md exists: Generate detailed backend, docs/catalog, and verification tasks for saga state, messages, consumers, compensation, timeout/idempotency handling, observability, and event catalog updates
   - If plan.md links ADRs: Generate tasks that honor the accepted/draft architectural decisions and include required follow-up docs/catalog updates
   - Generate docs/catalog update tasks only for owning services, changed supporting services, new contracts, or changed contracts
   - Generate verification-only tasks for reused supporting services and unchanged common API/event contracts
   - Generate detailed tasks under `FEATURE_DIR/tasks/`, grouped by implementation ownership, then service/app/package/lib, then action (see Task Generation Rules below)
   - Use user-story labels only for traceability, not as the primary grouping
   - Generate dependency order showing implementation group sequencing and story traceability
   - Validate task completeness: each task has exact file path/file set, intended change detail, forbidden behavior, dependencies/callers where relevant, and acceptance

4. **Generate task artifacts**: Read the tasks template from TASKS_TEMPLATE and use it as the structure for `tasks.md`. If TASKS_TEMPLATE is empty, fall back to `.specify/templates/tasks-template.md`. Fill with:
   - Correct feature name from plan.md
   - `tasks.md` as the compatibility entrypoint, summary, task-file index, dependency order, story traceability table, and completion checklist
   - Detailed task files under `FEATURE_DIR/tasks/` as needed: `backend.md`, `frontend.md`, `docs-catalog.md`, and `verification.md`
   - Detailed tasks grouped by service/app/package/lib and then action headings: Create, Update, Remove, Move/Rename, Verify
   - Detailed tasks with stable prefixes: B### backend, F### frontend, D### docs/catalog, V### verification
   - Tests only if requested, placed in the relevant detailed file or `verification.md` depending on whether they create test code or run checks
   - Documentation/catalog scope that says which service docs and catalog entries are editable, and which supporting services are verification-only

5. **Report**: Output path to generated `tasks.md` and generated detailed task files, plus summary:
   - Total task count across all detailed files
   - Task count by detailed file and by user story label
   - Implementation groups and dependency order
   - Independent test criteria for each story
   - Suggested MVP scope (typically the minimal backend/frontend/docs/verification groups needed for User Story 1)
   - Format validation: Confirm ALL detailed tasks follow the detailed checkbox format with IDs, story labels where applicable, file paths/file sets, exact change details, and acceptance checks

6. **Check for extension hooks**: After tasks are generated, check if `.specify/extensions.yml` exists in the project root.
   - If it exists, read it and look for entries under the `hooks.after_tasks` key
   - If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally
   - Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
   - For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
     - If the hook has no `condition` field, or it is null/empty, treat the hook as executable
     - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation
   - When constructing slash commands from hook command names, replace dots (`.`) with hyphens (`-`). For example, `speckit.git.commit` -> `/speckit-git-commit`.
   - For each executable hook, output the following based on its `optional` flag:
     - **Optional hook** (`optional: true`):
       ```
       ## Extension Hooks

       **Optional Hook**: {extension}
       Command: `/{command}`
       Description: {description}

       Prompt: {prompt}
       To execute: `/{command}`
       ```
     - **Mandatory hook** (`optional: false`):
       ```
       ## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
       ```
   - If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently

Context for task generation: $ARGUMENTS

The generated task set should be immediately executable. `tasks.md` is the entrypoint and detailed tasks under `tasks/` must be specific enough that an LLM can complete each task without additional context.

## Task Generation Rules

**CRITICAL**: Detailed tasks MUST be organized by implementation ownership, not primarily by user story. Preserve user-story labels (`[US1]`, `[US2]`, etc.) for traceability and independent acceptance.

**Tests are OPTIONAL**: Only generate test tasks if explicitly requested in the feature specification or if user requests TDD approach.

### Task Files (REQUIRED)

Keep `tasks.md` as the compatibility entrypoint and high-level tracker. Generate detailed files under `FEATURE_DIR/tasks/` only when that area has work:

- `backend.md` for backend services, APIs, domain/application/infrastructure code, shared backend libs, migrations
- `frontend.md` for frontend apps, shared package, services, stores, components, pages, routes, i18n
- `docs-catalog.md` for service docs, API catalog, event catalog, ADRs, architecture docs
- `verification.md` for builds, tests, lint/type-check, quickstart/manual validation, final searches

Do not generate `tasks/config.md`, `C###` task IDs, or feature tasks that edit `../hivespace.config`. Source-repo runtime settings, appsettings, gateway route config, and frontend environment typing belong in `backend.md` or `frontend.md` based on the owning source repo. The config repo may remain referenced only as local infrastructure context, such as starting Docker Compose.

### Detailed Checklist Format (REQUIRED)

Every detailed task MUST strictly follow this format:

```text
- [ ] [TaskID] [Story?] Action `PrimaryTarget`
  - File: `exact/path/or/file-set`
  - [Detail bullets describing exact implementation]
  - Acceptance: [verifiable completion check]
```

**Format Components**:

1. **Checkbox**: ALWAYS start with `- [ ]` (markdown checkbox)
2. **Task ID**: Unique ID across all detailed files using area prefixes: B###, F###, D###, V###
3. **[Story] label**: REQUIRED for feature work that maps to a user story
   - Format: [US1], [US2], [US3], etc. (maps to user stories from spec.md)
   - Cross-cutting setup, docs, and verification tasks may omit story labels only when they do not map to a single story
4. **Action and primary target**: Start with Create, Update, Remove, Move/Rename, or Verify
5. **Detail bullets**: Include target file path or explicit file set, exact fields/types/methods/routes/events/config keys/docs sections to create/update/remove/move, forbidden behavior, affected callers/dependencies when relevant, and acceptance

**Examples**:

- CORRECT: `- [ ] B001 [US1] Create ApplicationUser.cs` followed by File, Include fields, Add/update methods, Do not include, and Acceptance bullets
- CORRECT: `- [ ] F003 [US1] Update buyer OIDC authority config` followed by File, Change, Keep, Verify affected callers, and Acceptance bullets
- WRONG: `- [ ] Create User model` (missing ID, story label, file path, and implementation detail)
- WRONG: `- [ ] B001 [US1] Create ApplicationUser` with no detail bullets
- WRONG: `- [ ] T001 [US1] Create model` (wrong ID prefix and missing file path/detail)

### Task Organization

1. **From Implementation Ownership** - PRIMARY ORGANIZATION:
   - Group by detailed file: backend, frontend, docs/catalog, verification
   - Inside each file, group by service/app/package/lib/docs area
   - Inside each area, group by action: Create, Update, Remove, Move/Rename, Verify
   - Use user-story labels for traceability only

2. **From Contracts**:
   - Map each interface contract to the owning implementation area and the user story it serves
   - If tests requested: create test-code tasks in the relevant backend/frontend group before implementation tasks, and run-test tasks in `verification.md`

3. **From Data Model**:
   - Map each entity to the owning service/lib and user story labels
   - If an entity serves multiple stories, place it in the owning service group and list all relevant story labels or note shared dependency in the detail bullets
   - Relationships become exact field/method/mapping details in the relevant task bullets

4. **From Setup/Infrastructure**:
   - Backend setup goes in `backend.md`
   - Frontend setup goes in `frontend.md`
   - Source-repo appsettings, gateway route config, and frontend environment typing go in `backend.md` or `frontend.md`
   - Do not create feature tasks for Docker, local/cloud infrastructure config, environment sync, or `../hivespace.config`
   - Cross-cutting verification goes in `verification.md`

5. **From Documentation/Catalog Scope**:
   - Owning service docs: doc update tasks when the shipped feature changes that service
   - Changed supporting service docs: doc update tasks only when that service's API, event, validation, workflow, ownership, or behavior changed
   - Reused supporting services: verification-only tasks; do not generate doc edit tasks
   - Existing shared API/event catalog rows: verification-only tasks unless the contract itself changed

### Dependency Structure

- `tasks.md` must include the overall dependency order across detailed files.
- Detailed tasks should avoid vague "implement feature" wording. Split tasks when a file needs unrelated actions or when acceptance would be hard to verify.
- Tasks that affect the same file must make sequencing clear in the detail bullets or dependency order.
- Keep each user story independently verifiable through acceptance criteria and `verification.md`, even though tasks are grouped by implementation ownership.
