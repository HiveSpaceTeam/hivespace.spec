# Quickstart: System Testing Quality Gate

This quickstart describes how an implementer should verify the planned feature once tasks exist. Commands are source-repo commands; no implementation code belongs in `hivespace.spec`.

## 1. Backend

From `../hivespace.microservice`:

```powershell
dotnet restore
dotnet build
dotnet test
```

Expected backend implementation outcome:

- All new test projects are included in `hivespace.microservice.sln`.
- Test package versions remain centralized in `Directory.Packages.props`.
- Service-specific tests can run independently for impact-based checks.
- Full `dotnet test` covers release-readiness backend checks.

## 2. Frontend

From `../hivespace.web` after test scripts are implemented:

```powershell
pnpm install
pnpm test
pnpm type-check
pnpm lint
```

Expected frontend implementation outcome:

- Root workspace and affected app/shared packages expose test scripts through pnpm/Turbo.
- Buyer, seller, admin, and shared package checks can run independently.
- Known baseline lint/type-check failures remain documented and are not expanded by this feature.

## 3. Quality Gate

Run the implemented quality-gate entrypoint in source repos with one of these scopes:

```text
docs-only
backend:<service>
frontend:<app>
shared
release
```

Expected quality-gate output:

- Matches [contracts/quality-gate-report.md](./contracts/quality-gate-report.md).
- Shows pass/fail/not-applicable/accepted-risk/environment/missing-data/unstable-check results.
- Runs shared baseline checks for runtime changes.
- Allows one rerun for inconsistent checks, then reports blocking unstable-check unless risk is accepted.

## 4. Safety Expectations

- Do not use real customer data.
- Do not use real payment credentials.
- Do not send live customer-impacting notifications.
- Do not require changes to `../hivespace.config`.
