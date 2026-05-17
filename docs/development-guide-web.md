# Development Guide: hivespace.web

## Prerequisites

| Tool | Required version |
|------|-----------------|
| Node.js | >= 20.0.0 |
| pnpm | 9.15.4 (declared in `packageManager`) |
| Turbo | Installed via devDependencies (no global install needed) |

Install the correct pnpm version:
```bash
corepack enable
corepack prepare pnpm@9.15.4 --activate
```

---

## Installation

From the repo root:

```bash
pnpm install
```

This installs all workspace packages and creates symlinks between them. `@hivespace/shared` is available to apps as a workspace dependency resolved via `pnpm-workspace.yaml`.

---

## Running Apps

### From the repo root (via Turbo):

```bash
pnpm dev:admin    # runs @hivespace/admin on port 5173
pnpm dev:seller   # runs @hivespace/seller on port 5174
pnpm dev:buyer    # runs @hivespace/buyer on port 5175
pnpm dev          # runs all three apps in parallel
```

### From within an app directory:

```bash
cd apps/admin && pnpm dev
cd apps/seller && pnpm dev
cd apps/buyer && pnpm dev
```

The port can be overridden via `VITE_DEV_PORT` or `PORT` environment variables.

**Important**: In development, `@hivespace/shared` is resolved directly from `packages/shared/src/index.ts` via Vite alias — you do **not** need to build the shared package first for `pnpm dev` to work. However, `pnpm build` and `pnpm type-check` do require a prior `@hivespace/shared` build because they depend on `dist/`.

---

## Building

### Build everything (respects Turbo dependency order):
```bash
pnpm build
```
Turbo builds `@hivespace/shared` first (because apps `dependsOn: ["^build"]`), then builds all three apps in parallel.

### Build the shared package only:
```bash
pnpm build:shared
```
Required before running `pnpm type-check` or `pnpm build` in a fresh checkout (or after changing `packages/shared/src/`).

### Build a single app:
```bash
pnpm build --filter=@hivespace/admin
pnpm build --filter=@hivespace/seller
pnpm build --filter=@hivespace/buyer
```

### Watch mode for shared package:
```bash
cd packages/shared && pnpm dev
# runs: vite build --watch
```
Use this alongside a dev app server when actively modifying `@hivespace/shared`.

---

## Type Checking

```bash
pnpm type-check           # type-check all workspaces
```

Or per-app from its directory:
```bash
cd apps/admin && pnpm type-check     # vue-tsc --build
cd apps/seller && pnpm type-check
cd apps/buyer && pnpm type-check
```

Per-workspace from `packages/shared`:
```bash
cd packages/shared && pnpm type-check   # vue-tsc --noEmit
```

**Known baseline failures** (do not treat as regressions unless you changed affected files):
- `apps/admin` and `apps/seller`: pre-existing demo component type errors in dev-only files.
- `@hivespace/shared`: some strict-null and unused-param warnings in the package itself.
- `apps/buyer`: minor type-check issues in demo integration code.

---

## Linting

```bash
pnpm lint          # ESLint --fix all workspaces
```

Or per-app:
```bash
cd apps/admin && pnpm lint
cd apps/seller && pnpm lint
cd apps/buyer && pnpm lint
```

**Known baseline lint errors**: pre-existing demo components in `@hivespace/demo` have unused import and style errors. These are not regressions.

---

## Formatting

```bash
pnpm format       # Prettier --write src/ for all workspaces
```

---

## Environment Variables

Each app requires a `.env` file. A `.env.development` is already committed in each app as a reference. Create `.env` in each app directory for local overrides.

### Variable Resolution Order

The `src/config/index.ts` in each app resolves the API gateway URL in this order:
1. `VITE_GATEWAY_BASE_URL`
2. `VITE_API_BASE_URL`
3. `VITE_API_URL`
4. Fallback: `https://localhost:7001/api`

### Required Variables (all apps)

| Variable | Description | Example |
|----------|-------------|---------|
| `VITE_APP_CLIENT_ID` | OIDC client ID registered in Duende IdentityServer | `adminportal` / `sellercenter` / `storefront` |
| `VITE_APP_REDIRECT_URI` | OIDC login callback URL | `http://localhost:{port}/callback/login` |
| `VITE_APP_POST_LOGOUT_REDIRECT_URI` | OIDC logout callback URL | `http://localhost:{port}/callback/logout` |
| `VITE_APP_RESPONSE_TYPE` | OIDC response type | `code` |
| `VITE_APP_SCOPE` | OIDC scopes requested | see per-app below |
| `VITE_APP_RESPONSE_MODE` | OIDC response mode | `query` |
| `VITE_AUTH_AUTHORITY_URL` | Identity Server base URL | `http://localhost:5001` |
| `VITE_API_URL` or `VITE_GATEWAY_BASE_URL` | API Gateway base URL | `http://localhost:5000` |

### Optional Variables (all apps)

| Variable | Default | Description |
|----------|---------|-------------|
| `VITE_APP_NAME` | `HiveSpace [App Name]` | Application display name |
| `VITE_APP_VERSION` | `1.0.0` | App version string |
| `VITE_APP_ENVIRONMENT` | `development` | `development \| staging \| production` |
| `VITE_API_TIMEOUT` | `30000` | Axios request timeout (ms) |
| `VITE_ENABLE_LOGGING` | `false` | Enable verbose logging |
| `VITE_ENABLE_DEBUG` | `false` | Enable debug API logging |
| `VITE_STORAGE_BASE_URL` | `https://storage.hivespace.com` | Azure Blob/storage base URL |
| `VITE_CDN_BASE_URL` | `https://cdn.hivespace.com` | CDN base URL |

### Per-App Scope Defaults

| App | OIDC Scopes |
|-----|-------------|
| admin | `openid profile user.fullaccess` |
| seller | `openid profile offline_access user.fullaccess catalog.fullaccess order.fullaccess notification.fullaccess media.fullaccess` |
| buyer | `openid profile offline_access user.fullaccess catalog.fullaccess order.fullaccess notification.fullaccess` |

Note: `offline_access` is required for refresh token support (seller and buyer). Admin does not use token refresh.

---

## OIDC Configuration for Local Development

1. Register three OIDC clients in the local Duende IdentityServer (`hivespace.microservice/src/HiveSpace.UserService`):
   - `adminportal` — redirect URIs: `http://localhost:5173/callback/login`, post-logout: `http://localhost:5173/callback/logout`
   - `sellercenter` — redirect URIs: `http://localhost:5174/callback/login`, post-logout: `http://localhost:5174/callback/logout`
   - `storefront` — redirect URIs: `http://localhost:5175/callback/login`, post-logout: `http://localhost:5175/callback/logout`

2. Start the UserService at `http://localhost:5001`.

3. Start the API Gateway at `http://localhost:5000`.

4. Set `VITE_AUTH_AUTHORITY_URL=http://localhost:5001` and `VITE_API_URL=http://localhost:5000` in each app's `.env`.

5. The OIDC flow uses `response_type=code` + PKCE. No iframe silent renewal — token refresh uses the `refresh_token` grant directly.

---

## Shared Package Development Workflow

When making changes to `packages/shared/src/`:

**During development** (hot-reload OK):
- Apps in `pnpm dev` mode use the Vite alias `@hivespace/shared → packages/shared/src/index.ts` directly, so source changes are immediately reflected.

**For build and type-check**:
```bash
pnpm build:shared    # rebuilds dist/ from packages/shared/src/
pnpm type-check      # now uses the new dist/
```

**CSS changes in shared**:
- Edit `packages/shared/src/styles/*.css`.
- Changes are live in dev (Vite processes CSS directly from source via `@hivespace/shared/style.css` import).
- Run `pnpm build:shared` to update the `dist/` for production builds.

**Adding a new export to shared**:
1. Create the component/composable/type in the appropriate `src/` subdirectory.
2. Add the export to the nearest `index.ts` barrel file.
3. Ensure it reaches `src/internal.ts` (which exports everything from `src/index.ts`).
4. Run `pnpm build:shared` and verify the export appears in `dist/index.d.ts`.

---

## Turbo Pipeline

```json
{
  "build":      { "dependsOn": ["^build"], "outputs": ["dist/**"] },
  "dev":        { "cache": false, "persistent": true },
  "type-check": { "dependsOn": ["^build"], "outputs": [] },
  "lint":       { "outputs": [] },
  "format":     { "outputs": [] }
}
```

Key behaviors:
- `^build` means "build all dependencies first". `@hivespace/shared` must build before any app.
- `dev` has `cache: false` and `persistent: true` — long-running, never cached.
- `build` and `type-check` are cached using hashes of `src/**`, `tsconfig*.json`, etc.
- Cache lives in `.turbo/` (local) and can be remote with Turbo Remote Cache.

---

## Vite Configuration

All three apps share the same `vite.config.ts` pattern:

```ts
defineConfig({
  plugins: [vue(), vueJsx(), vueDevTools(), tailwindcss()],
  resolve: {
    alias: [
      { find: '@', replacement: fileURLToPath(new URL('./src', import.meta.url)) },
      ...workspaceAliases(), // adds @hivespace/shared and @hivespace/demo aliases
    ],
    dedupe: ['vue', 'vue-router', 'pinia'],
  },
  server: { port: 5173 | 5174 | 5175 },
})
```

`workspaceAliases()` from `@hivespace/vite-config` adds:
- `@hivespace/shared` → `packages/shared/src/index.ts` (exact regex match, so subpath imports like `@hivespace/shared/style.css` still use the package's `exports` field)
- `@hivespace/demo` → `packages/demo/src/index.ts`

`dedupe` prevents multiple copies of vue, vue-router, and pinia in the module graph.

---

## TypeScript Configuration

Each app extends `@hivespace/tsconfig/vue-app.json`:

```json
{
  "target": "ES2020",
  "module": "ESNext",
  "moduleResolution": "bundler",
  "strict": true,
  "noUnusedLocals": true,
  "noUnusedParameters": true
}
```

The `@` alias is declared in each app's `tsconfig.app.json`:
```json
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }
  }
}
```

---

## Feature Development Order

When adding a new feature to any app:

1. **Define types** in `src/types/` (new file or add to existing). Export from `src/types/index.ts`.
2. **Create service** in `src/services/` extending `BaseService`. Add to `src/services/index.ts`.
3. **Create or update Pinia store** in `src/stores/` using `defineStore` with setup syntax. Add to `src/stores/index.ts`.
4. **Build components** in `src/components/{feature}/` if feature-specific UI is needed.
5. **Create page view** in `src/pages/{Feature}/`.
6. **Add route** in `src/router/index.ts`.
7. **Add i18n keys** in `src/i18n/locales/{en,vi}/{feature}.json` and include them in `src/i18n/index.ts`.

For shared components/composables, follow the same pattern but in `packages/shared/src/` and export from `src/internal.ts`.

---

## Code Conventions

- **Components**: `<script setup lang="ts">` always. No Options API.
- **Styling**: Tailwind utility classes only. No scoped CSS unless absolutely necessary.
- **Functions**: Arrow functions for non-top-level declarations.
- **i18n**: All user-facing strings via `$t('module.key')` or `t('module.key')`. No hardcoded strings.
- **Imports**: Use `@/...` for app-local, `@hivespace/shared` for shared exports. Never cross-import between apps.
- **Types**: `import type { ... }` for type-only imports. App-local types in `src/types/`. Component-specific types inline in the `.vue` file.
- **Pinia stores**: `defineStore(id, setup)` form only. State is `ref()`, actions are plain functions. No `$reset()` — use `$patch()` or manual reset functions.
- **Error handling**: Never swallow errors silently in stores — use `useAppStore().notifyError()` for user-visible errors.
- **Comments**: Sparingly. Only comment complex logic.

---

## Verification After Changes

### After changing a single app:
```bash
cd apps/{admin|seller|buyer}
pnpm lint
pnpm type-check
```

### After changing `@hivespace/shared`:
```bash
cd packages/shared
pnpm type-check

# Then verify consumer apps still build:
pnpm build:shared
cd apps/{admin|seller|buyer}
pnpm type-check
```

### Full repo verification:
```bash
# From repo root:
pnpm build:shared && pnpm type-check && pnpm lint
```

---

## Known Baseline Issues (Do Not Fix Unless Working on Affected Files)

| App / Package | Issue | Affected file(s) |
|---------------|-------|-----------------|
| `@hivespace/admin` | `type-check` failures in demo integration | `src/pages/DemoWrapper.vue`, demo-related pages |
| `@hivespace/seller` | Pre-existing lint errors in demo components | `@hivespace/demo` routes |
| `@hivespace/buyer` | Minor type-check issues in demo | Demo route integration |
| `@hivespace/shared` | `vue-i18n` version conflict between shared (11.x) and apps (9.x) — managed via root `pnpm.overrides` locking apps to `^9.3.0` | All apps' `vue-i18n` usage |
| `@hivespace/seller` | `debugger` statement left in router guard | `apps/seller/src/router/index.ts` line 224 |

---

## CI/CD

No GitHub Actions workflows were found in the root `.github/` directory of `hivespace.web`. Individual app directories (`apps/admin/.github/`, `apps/seller/.github/`) may contain app-local CI configurations, but they were not scanned as part of this documentation.

The standard CI verification commands to run are:
```bash
pnpm build:shared
pnpm lint
pnpm type-check
pnpm build
```
