# Architecture: HiveSpace Web Frontend (hivespace.web)

## Executive Summary

`hivespace.web` is a **pnpm monorepo** (pnpm 9.15.4, Turbo 2.3.0) containing three production Vue 3 applications and four support packages. All apps share a common technology foundation, auth strategy, and design language via the `@hivespace/shared` package. There is no cross-app importing; all shared code flows exclusively through `@hivespace/shared`.

| App | Package name | Purpose | Dev port |
|-----|-------------|---------|----------|
| `apps/admin` | `@hivespace/admin` v2.0.1 | Platform admin dashboard | 5173 |
| `apps/seller` | `@hivespace/seller` v2.0.1 | Seller dashboard | 5174 |
| `apps/buyer` | `@hivespace/buyer` v1.0.0 | Customer storefront | 5175 |

| Package | Purpose |
|---------|---------|
| `packages/shared` (`@hivespace/shared` v1.1.20) | Shared UI components, composables, services, stores |
| `packages/demo` (`@hivespace/demo`) | Dev-only demo/storybook-style pages |
| `packages/eslint-config` | Shared ESLint rule set |
| `packages/tsconfig` | Shared TypeScript configs (`base`, `vue-app`, `node`) |
| `packages/vite-config` | `workspaceAliases()` helper for Vite |

---

## Technology Stack

| Category | Library / Version |
|----------|-------------------|
| UI Framework | Vue 3 (`^3.5.13`; buyer `^3.5.25`) |
| Build tool | Vite `^6.0.11` |
| TypeScript | `~5.7.3` |
| Router | vue-router `^4.5.0` |
| State management | Pinia `^3.0.3` |
| Styling | Tailwind CSS v4 (`^4.0–4.2`) — CSS-first, `@tailwindcss/vite` plugin, no `tailwind.config.js` |
| HTTP client | Axios `^1.11.0` (shared `^1.13.4`) |
| Auth | oidc-client-ts `^3.3.0` |
| Real-time | `@microsoft/signalr` `^8.0.7` |
| i18n | vue-i18n `^9.14.5` (shared `^11.2.2`) |
| Modals | vue-final-modal `^4.5.5` |
| Rich text | `@vueup/vue-quill` `^1.2.0` |
| Charts | vue3-apexcharts `^1.8.0` / apexcharts `^5.10.0` |
| Calendar | `@fullcalendar/vue3` `^6.1.15` + plugins |
| Date picker | vue-flatpickr-component `^11.0.5` / flatpickr `^4.6.13` |
| File upload | dropzone `^6.0.0-beta.2` |
| Icons (buyer only) | lucide-vue-next `^0.474.0` |
| Vector maps | jsvectormap `^1.6.0` / vuevectormap `^2.0.1` |
| Drag-and-drop | vuedraggable `^4.1.0` |
| Carousel | swiper `^11.2.x` |
| Type checking | vue-tsc `^2.2.0` |
| Monorepo orchestrator | Turbo 2.3.0 |
| Package manager | pnpm 9.15.4 (Node >= 20.0.0) |

---

## Monorepo Structure

```
hivespace.web/
├── apps/
│   ├── admin/          # @hivespace/admin
│   ├── seller/         # @hivespace/seller
│   └── buyer/          # @hivespace/buyer
├── packages/
│   ├── shared/         # @hivespace/shared — runtime shared library
│   ├── demo/           # @hivespace/demo — dev-only
│   ├── eslint-config/  # shared lint rules
│   ├── tsconfig/       # base / vue-app / node tsconfigs
│   └── vite-config/    # workspaceAliases() helper
├── turbo.json
└── pnpm-workspace.yaml
```

### Turbo Task Graph

| Task | Depends on | Effect |
|------|-----------|--------|
| `build` | `^build` | Builds upstream packages first |
| `type-check` | `^build` | Requires shared to be built |
| `dev` | (none) | All workspaces run in parallel |
| `lint` | (none) | Independent per workspace |
| `format` | (none) | Independent per workspace |

`pnpm build` from root always builds `packages/shared` before any app. `pnpm dev` runs all workspaces concurrently — shared source resolution in dev is handled by the Vite alias (see [Vite Configuration](#vite-configuration)).

---

## Architecture Patterns

### Composition API Only

Every component uses `<script setup lang="ts">`. The Options API is prohibited across all apps.

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
// ...
</script>
```

### Pinia Setup Store Pattern

All stores use the Composition API (setup) form, not the Options API form:

```ts
export const useMyStore = defineStore('my', () => {
  const items = ref<Item[]>([])
  const total = computed(() => items.value.length)

  async function fetchItems() { /* ... */ }

  return { items, total, fetchItems }
})
```

Use `storeToRefs` when destructuring reactive state from a store to preserve reactivity.

### Shared Factory Pattern

The shared package exposes factory functions for stores and services that need app-specific configuration at creation time. Each app calls the factory once during setup and re-exports the resulting composable/store.

```ts
// In apps/{app}/src/stores/notification.ts
import { createNotificationStore } from '@hivespace/shared'
export const useNotificationStore = createNotificationStore({ /* app-specific config */ })
```

Factory exports from shared:
- `createNotificationStore(config)`
- `createUserProfileStore(config)`
- `createUserSettingsStore(config)`
- `createMediaUploadStore(config)`
- `createNotificationService(config)`
- `createUserProfileService(config)`
- `createUserSettingsService(config)`
- `createMediaUploadService(config)`

### Service Layer (`ApiService` / `BaseService`)

All API calls route through the shared `ApiService` class, which wraps Axios with:
- Automatic `Authorization: Bearer {token}` injection
- Retry logic on transient failures
- Structured error handling (maps backend error codes to translated messages)
- Correlation ID tracing (`X-Correlation-ID` header)

Domain services extend `BaseService` and call `ApiService` methods. All services live in `src/services/` and are named `*.service.ts`.

### Styling

Tailwind CSS v4 is used exclusively via utility classes. There is no `tailwind.config.js` — configuration is done in CSS via `@tailwindcss/vite` plugin directives. Custom CSS is written only for styles that cannot be expressed as utility classes.

### Functions

Arrow functions are used exclusively, except for top-level declarations that require hoisting.

---

## Per-App Deep Dive

### Admin App (`apps/admin`, port 5173)

**Purpose**: Platform administration — managing admin accounts, buyer accounts, merchants (seller KYC), audit logs, disputes, scheduled jobs, and system configuration.

#### Routes

| Path | Auth required | Notes |
|------|--------------|-------|
| `/dashboard` | RequireAdmin/SystemAdmin | — |
| `/merchants` | RequireAdmin/SystemAdmin | Seller KYC queue |
| `/configuration` | RequireAdmin/SystemAdmin | — |
| `/accounts` | RequireAdmin/SystemAdmin | Admin user management |
| `/buyers` | RequireAdmin/SystemAdmin | Buyer account management |
| `/audit-log` | RequireAdmin/SystemAdmin | — |
| `/kyc-queue` | RequireAdmin/SystemAdmin | — |
| `/disputes` | RequireAdmin/SystemAdmin | — |
| `/scheduled-jobs` | RequireAdmin/SystemAdmin | — |
| `/callback/login` | Anonymous | OIDC redirect handler |
| `/callback/logout` | Anonymous | Post-logout handler |
| `/server-error` | Anonymous | Error page |
| `/maintenance` | Anonymous | Maintenance page |
| `/` | Anonymous | Landing / redirect |

#### Auth Guard Logic

```
if user is NOT isAdmin() AND NOT isSystemAdmin():
    logout()
```

Simple role check — any non-admin user (including authenticated buyers or sellers) is forced to log out.

#### Pinia Stores

| Store | Key responsibility |
|-------|--------------------|
| `useAdminStore` | Admin accounts CRUD (create, list, update status, delete) |
| `useUserStore` | Buyer account list + status management |
| `useNotificationStore` | Shared factory — in-app notifications |
| `useProfileStore` | Shared factory — authenticated user profile |
| `useUserSettingsStore` | Shared factory — locale/theme preferences |

#### Key Features

- Invite/create admin accounts (`AccountsInviteModal`, `AdminDetailModal`)
- View and manage buyer accounts
- Merchant KYC review queue
- Audit log viewer
- Dispute management panel

---

### Seller App (`apps/seller`, port 5174)

**Purpose**: Seller-facing dashboard for product management, order fulfillment, marketing (coupons), and store registration/onboarding.

#### Routes

| Path | Auth required | Notes |
|------|--------------|-------|
| `/product/list` | Auth required | Product listing |
| `/product/new` | Auth required | Create product |
| `/product/:id` | Auth required | Edit product (`UpsertProductPage`) |
| `/orders/all` | Auth required | Order management |
| `/marketing/coupons` | Auth required | Coupon list |
| `/marketing/coupons/create` | Auth required | Create coupon |
| `/marketing/coupons/detail/:id` | Auth required | Coupon detail/edit |
| `/notifications` | Auth required | — |
| `/register-seller` | Auth required | Seller onboarding |
| `/verify-email` | Auth required | Email verification prompt |
| `/callback/login` | Anonymous | OIDC redirect handler |
| `/callback/logout` | Anonymous | — |
| `/verify-email-callback` | Anonymous | Email verification link callback |
| `/server-error` | Anonymous | — |
| `/maintenance` | Anonymous | — |
| `/` | Anonymous | — |

#### Auth Guard Logic (multi-step)

```
1. If user IS admin → logout()                              (admins cannot use seller app)
2. If email NOT verified → redirect /verify-email
3. If email verified BUT no store registered → redirect /register-seller
4. After store registration → force token refresh           (to obtain StoreOwner role claim)
```

> **Known dev artifact**: A `debugger` statement exists in the seller router guard (approximately line 223). This must be removed before any production deployment.

#### Pinia Stores

| Store | Key responsibility |
|-------|--------------------|
| `useProductStore` | Product CRUD, fetch categories and attributes |
| `useOrderStore` | Order list with filter/pagination, confirm/reject order |
| `useCouponStore` | Coupon CRUD + end-coupon action |
| `useStoreStore` | Store registration + forced token refresh post-registration |
| `useMediaStore` | Shared factory — presigned URL media uploads |
| `useNotificationStore` | Shared factory |
| `useProfileStore` | Shared factory |
| `useUserSettingsStore` | Shared factory |

#### Key Features

- Full product lifecycle management with category/attribute selection (`SelectProductsModal`)
- Order confirmation and rejection workflow
- Coupon creation and management
- Two-step seller onboarding: email verification → store registration → automatic role upgrade via token refresh
- Media upload via presigned S3 URLs

---

### Buyer App (`apps/buyer`, port 5175)

**Purpose**: Customer-facing storefront — product discovery, cart, checkout, order tracking, and account management.

#### Routes

| Path | Auth required | Notes |
|------|--------------|-------|
| `/` | Anonymous | Home page |
| `/product` | Anonymous | Product listing/search |
| `/payment/result` | Anonymous | Payment result page |
| `/callback/login` | Anonymous | OIDC redirect |
| `/callback/logout` | Anonymous | — |
| `/server-error` | Anonymous | — |
| `/maintenance` | Anonymous | — |
| `/cart` | Auth required | Shopping cart |
| `/checkout` | Auth required | Checkout flow |
| `/profile` | Auth required | User profile |
| `/profile/address` | Auth required | Address management |
| `/orders` | Auth required | Order history |
| `/account/orders/:id` | Auth required | Order detail |
| `/notifications` | Auth required | — |

**Layouts**: Most pages use `StorefrontLayout`. Cart, checkout, and payment result pages use no layout wrapper (full-page).

#### Auth Guard Logic

Any authenticated user is permitted — no role check beyond being logged in.

#### Pinia Stores

| Store | Key responsibility |
|-------|--------------------|
| `useProductStore` | `homeProducts`, `recommendedProducts`, `similarProducts`, `productDetail` |
| `useCartStore` | `cartGroups`, `selectedCount`, `summary`, `platformCoupons`, `invalidatedCoupons`; serializes mutations via Promise chain queue; in-place reactivity patching |
| `useCheckoutStore` | Checkout preview state; also uses `mutationQueue`; computed: `packages`, `totalItems`, `subtotal`, `originalSubtotal`, `totalShippingFee`, `grandTotal`, `shippingDiscount`, `totalSaved` |
| `usePaymentStore` | `payment`, `isLoading`; `fetchPaymentByOrder` |
| `useAddressStore` | `addresses` (sorted default-first), `defaultAddress`; `saveAddress(data, editId?)` |
| `useOrdersStore` | `orders`, `activeTab`, `searchQuery`, `hasNextPage`; debounced search 400 ms |
| `useCouponStore` | `availableCouponStoresByKey` keyed by `"storeId:productIds"`; prevents duplicate fetches |
| `useProfileStore` | `profile`, `form` (`ProfileFormData` with `birth day/month/year`, `gender` as int mapping) |
| `useCategoryStore` | Homepage category tree |

#### Key Design: Cart Mutation Queue

The `useCartStore` and `useCheckoutStore` use a **Promise chain mutation queue** to serialize all write operations. This prevents race conditions when the user rapidly adds/removes items or applies coupons, while maintaining instant UI feedback via in-place reactivity patching before the network call completes.

#### Key Features

- Home page: hero banner, flash sale, category bar, product grid, recommendations
- Cart with serialized mutation queue
- Checkout: preview + submit; computes full price breakdown including shipping discounts
- Coupon system: platform-wide and per-store coupons (`AvailableCouponPopover`)
- Order tracking with tabbed status filtering and debounced search
- Real-time notifications via SignalR (`useNotificationHub` / `useNotificationRealtime`)
- Address management with default-address promotion

---

## Shared Package (`packages/shared`)

`@hivespace/shared` is the runtime shared library. During development, the Vite alias bypasses `dist/` and points directly to `packages/shared/src/index.ts` for live-reload.

### What It Provides

| Category | Exports |
|----------|---------|
| Auth | `initializeAuth(config)`, `useAuth()`, `createRefreshService()` |
| HTTP | `ApiService` (Axios wrapper), `BaseService` (abstract base) |
| App store | `useAppStore` |
| Factory stores | `createNotificationStore`, `createUserProfileStore`, `createUserSettingsStore`, `createMediaUploadStore` |
| Factory services | `createNotificationService`, `createUserProfileService`, `createUserSettingsService`, `createMediaUploadService` |
| Composables | See table below |
| UI Components | Common, Layout, Modal, Pages (see `component-inventory-web.md`) |
| SVG Icons | Large set of icon components exported via index |

### Composable Catalog

| Composable | Purpose |
|------------|---------|
| `useAppShell` | App shell state (sidebar open/close, etc.) |
| `useAsyncAction` | Wraps async calls with `isLoading` / `error` state |
| `useConfirmModal` | Programmatic confirmation dialog |
| `useCooldown` | Button cooldown timer |
| `useDebounce` | Debounced ref or function |
| `useFieldValidation` | Per-field validation state management |
| `useFormatDate` | Locale-aware date formatting |
| `useModal` | Programmatic modal open/close control |
| `useNotificationHub` | SignalR connection lifecycle management |
| `useNotificationRealtime` | Reactive notification feed from hub |
| `useNumberInputFormatter` | Numeric input formatting and masking |
| `useSidebar` | Sidebar open/close state |
| `useValidationRules` | Reusable validation rule objects |

---

## Auth Architecture

### OIDC Authorization Code Flow + PKCE

All three apps authenticate via `oidc-client-ts` using Authorization Code Flow with PKCE. The authority is a Duende IdentityServer instance referenced by `VITE_AUTH_AUTHORITY_URL`.

```
User → App → IdentityServer (/connect/authorize)
           ← authorization code
App → IdentityServer (/connect/token, code + PKCE verifier)
    ← { access_token, refresh_token, id_token }
```

### Token Storage

| App | Storage |
|-----|---------|
| admin | `sessionStorage` |
| seller | `sessionStorage` |
| buyer | `localStorage` |

### Token Refresh Strategy

No silent iframe renewal. Explicit refresh token grant is used. The `createRefreshService()` from shared manages proactive refresh when the token has less than **60 seconds** remaining.

```
POST {VITE_AUTH_AUTHORITY_URL}/connect/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
client_id={VITE_APP_CLIENT_ID}
refresh_token={stored_refresh_token}
```

Axios interceptors in `ApiService` attach the current access token as `Authorization: Bearer {token}` on every request.

### AppUser Role Helpers

| Method | Role claim |
|--------|-----------|
| `isSystemAdmin()` | `SystemAdmin` |
| `isAdmin()` | `Admin` |
| `isSeller()` | `StoreOwner` |

### Per-App Guard Summary

| App | Guard logic |
|-----|------------|
| admin | `isAdmin() \|\| isSystemAdmin()` — else `logout()` |
| seller | Not admin; email verified; store registered; StoreOwner role |
| buyer | Authenticated only — no role requirement |

---

## i18n Architecture

### Configuration

```ts
createI18n({
  legacy: false,        // Composition API mode ($t available via useI18n())
  locale: 'vi',         // Default: Vietnamese
  fallbackLocale: 'en',
  messages: { vi, en }
})
```

All user-facing text must use translation keys via `$t('module.key')` or `const { t } = useI18n()`. Hardcoded strings in templates are not permitted.

### Culture Persistence

The selected locale is persisted in a `culture` cookie. For authenticated users, the backend setting (from `GET /users/settings`) overrides the cookie on login.

### Locale File Structure

**Admin** (`apps/admin/src/i18n/locales/{en,vi}/`):

`admins`, `users`, `accounts`, `buyers`, `audit-log`, `dashboard`, `merchants`, `configuration`, `notification`, `layout`, `common`, `backend-errors`, `icons`

**Seller** (`apps/seller/src/i18n/locales/{en,vi}/`):

`product`, `coupon`, `order`, `notification`, `register-store`, `verifyEmail`, `verifyEmailCallback`, `pages`, `common`, `backend-errors`

**Buyer** (`apps/buyer/src/i18n/locales/{en,vi}/`):

All keys namespaced under `storefront.`:
`common`, `address`, `profile-page`, `orders-page`, `cart`, `checkout`, `footer`, `categories`, `productDetail`, `payment`, `notification`, `orders`

### Backend Error Mapping

Each app has a `backend-errors.json` that maps backend error code strings to translated user messages:

```json
{
  "PRODUCT_NOT_FOUND": "Product not found.",
  "INSUFFICIENT_STOCK": "Not enough stock available."
}
```

`ApiService` extracts the error code from error responses and looks up the translation before surfacing the message to the UI.

---

## Vite Configuration

### Workspace Aliases

All apps use the `workspaceAliases()` helper from `packages/vite-config`:

```ts
// apps/{app}/vite.config.ts
import { workspaceAliases } from '@hivespace/vite-config'

export default defineConfig({
  plugins: [...],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, 'src'),
      ...workspaceAliases()
    }
  }
})
```

`workspaceAliases()` adds: `@hivespace/shared` → `packages/shared/src/index.ts`

### Alias Summary

| Alias | Resolves to | Purpose |
|-------|------------|---------|
| `@` | `apps/{app}/src/` | App-local absolute imports |
| `@hivespace/shared` | `packages/shared/src/index.ts` (dev) | Live-reload bypass of dist/ |

### Dev vs Prod Resolution

- **Dev** (`pnpm dev`): Vite alias intercepts `@hivespace/shared` → live TypeScript source in `packages/shared/src/`. Changes to shared components hot-reload in all apps without rebuilding.
- **Prod** (`pnpm build`): Turbo's `^build` dependency ensures `packages/shared` is built first. Apps then resolve `@hivespace/shared` from the compiled `dist/` output.

### File Naming Conventions

| Artifact | Convention |
|----------|-----------|
| `.vue` component files | `PascalCase.vue` |
| `.ts` service / util files | `kebab-case.ts` |
| Variables and functions | `camelCase` |
| Pinia store composables | `useXxxStore` |
