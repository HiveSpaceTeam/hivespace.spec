---
name: "update-catalog"
description: "Update HiveSpace event and API catalogs after planning is complete."
compatibility: "HiveSpace spec repo custom command"
---

Update the shared event and API catalogs after planning is complete.

Step 1 - Find the active feature
- Read `.specify/feature.json` if it exists and use its `feature_directory`.
- If it does not exist, inspect `specs/` and select the most recently modified feature folder.
- Read the feature's `spec.md`, `plan.md`, and `tasks.md` when present.

Step 2 - Update event catalog
- Read `shared/event-catalog.md`.
- Extract all new integration events, commands, saga messages, failure events, timeout events, and projection events from the plan and tasks.
- For each message:
  - Check whether a contract with the same name or equivalent meaning already exists.
  - If missing, add it to the correct section using the existing table shape for that section.
  - If an existing contract changed in name, owner, payload semantics, or consumer set, update only the stale fields.
  - Preserve existing contract names, owners, consumers, and descriptions when the feature only reuses that contract.
- Do not rename, remove, or rewrite existing events just to mention one feature-specific use of a common message.

Step 3 - Update API catalog
- Read `shared/api-catalog.md`.
- Extract all new public HTTP or realtime endpoints from the plan and tasks.
- For each endpoint:
  - Check whether the method and path already exist.
  - If missing, add it to the correct service section using the existing table shape.
  - If an existing endpoint changed in path, method, auth, owner, request/response contract, or purpose, update only the stale fields.
  - Preserve existing paths, auth policies, and purposes when the feature only reuses that endpoint.
- Do not remove or rewrite existing endpoints just to mention one feature-specific use of a common API.

Step 4 - Report
- Summarize every catalog entry added or updated.
- If nothing changed, say `Catalogs are already up to date.`
