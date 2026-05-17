---
name: "wrap-up"
description: "Update documentation after a HiveSpace feature has shipped."
compatibility: "HiveSpace spec repo custom command"
---

Update documentation after a feature has shipped.
Usage: `/wrap-up [NNN-feature-name]`

Step 1 - Resolve the feature
- If a feature name is supplied, use `specs/[feature-name]`.
- Otherwise read `.specify/feature.json`; if absent, use the most recently modified folder under `specs/`.
- Read `spec.md`, `plan.md`, and `tasks.md`.

Step 2 - Update service docs
For each affected service:
- Read `services/<name>/README.md` and related `api.md`, `data-and-events.md`, or `workflows.md` files when present.
- Add shipped endpoints.
- Add published and consumed events.
- Add workflow or saga changes.
- Add ADR links for decisions that shaped the implementation.
- Remove known limitations that the shipped feature resolved.

Step 3 - Verify catalogs
- Read `shared/event-catalog.md` and verify all shipped events/messages are present with correct ownership and consumers.
- Read `shared/api-catalog.md` and verify all shipped endpoints are present with correct auth and service ownership.
- Fix missing or stale entries.

Step 4 - Mark spec done
- In the feature `spec.md`, set status to `Implemented`.
- Add the implementation date.

Step 5 - Update backlog
- If `docs-backlog.md` exists, tick off items resolved by this feature.

Step 6 - Summary
- Show a table of every file changed with a one-line description of the change.
