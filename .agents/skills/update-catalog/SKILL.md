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
  - Preserve existing contract names, owners, consumers, and descriptions.
- Do not rename or remove existing events.

Step 3 - Update API catalog
- Read `shared/api-catalog.md`.
- Extract all new public HTTP or realtime endpoints from the plan and tasks.
- For each endpoint:
  - Check whether the method and path already exist.
  - If missing, add it to the correct service section using the existing table shape.
  - Preserve existing paths, auth policies, and purposes.
- Do not remove existing endpoints.

Step 4 - Report
- Summarize every catalog entry added.
- If nothing changed, say `Catalogs are already up to date.`
