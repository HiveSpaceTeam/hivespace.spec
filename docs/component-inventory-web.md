# Component Inventory: HiveSpace Web Frontend (hivespace.web)

All Vue components across the monorepo. Every component uses `<script setup lang="ts">` (Composition API only).

---

## Shared Package (`packages/shared/src/components/`)

The `@hivespace/shared` package provides the foundational component library consumed by all three apps.

### Common Components

| Component | Description |
|-----------|-------------|
| `Alert` | Dismissible alert banner (info, warning, error, success variants) |
| `Avatar` | User avatar with image or initials fallback |
| `Badge` | Status/count badge chip |
| `Button` | Primary action button with loading and disabled states |
| `Checkbox` | Accessible checkbox with label |
| `CommonGridShape` | Background decorative grid shape |
| `ComponentCard` | Card container for UI components |
| `ConfirmModal` | Programmatic confirmation dialog (used via `useConfirmModal`) |
| `CountDown` | Countdown timer display |
| `DatePicker` | Date selection input (wraps flatpickr) |
| `DateTimePicker` | Date + time selection input |
| `DropdownMenu` | Accessible dropdown menu with slot-based items |
| `Dropzone` | File drag-and-drop upload zone (wraps dropzone) |
| `FileInput` | Styled file input element |
| `FilterChips` | Horizontal chip list for active filter display |
| `FullscreenLoader` | Overlay loading spinner |
| `Input` | Text input with label, error message, and prefix/suffix slots |
| `Link` | Styled anchor / router-link |
| `Modal` | Base modal wrapper (uses vue-final-modal) |
| `MultipleSelect` | Multi-value select dropdown |
| `NotificationPreviewToast` | Toast preview for incoming notifications |
| `OrderTimeline` | Vertical timeline for order status history |
| `PageBreadcrumb` | Breadcrumb navigation bar |
| `Pagination` | Page navigation controls |
| `QuantityControl` | Increment/decrement quantity input |
| `Radio` | Single radio button |
| `RadioGroup` | Grouped radio buttons |
| `ResponsiveImage` | Image with srcset and lazy loading |
| `Select` | Styled single-value select dropdown |
| `Spinner` | Inline loading spinner |
| `Tabs` | Tab navigation with slot-based panels |
| `TextArea` | Multi-line text input with label and error |
| `ThemeToggler` | Light/dark mode toggle |
| `ThreeColumnImageGrid` | 3-column responsive image grid |
| `TimePicker` | Time-only selection input |
| `Toast` | Individual toast notification |
| `ToastContainer` | Mount point for all toast notifications |
| `ToggleSwitch` | Boolean toggle switch |
| `TwoColumnImageGrid` | 2-column responsive image grid |
| `YouTubeEmbed` | Responsive YouTube video embed |
| `v-click-outside` | Custom directive: fires event on click outside element |

### Layout Components

| Component | Description |
|-----------|-------------|
| `AppHeader` | Top navigation bar (used in admin/seller apps) |
| `AppShell` | Root layout shell (sidebar + header + main slot) |
| `AppSidebar` | Left navigation sidebar |
| `Backdrop` | Semi-transparent overlay (e.g., behind open sidebar on mobile) |
| `FullScreenLayout` | Layout with no sidebar/header (for error/maintenance pages) |
| `SidebarProvider` | Context provider for sidebar open/close state |
| `SidebarWidget` | Widget card rendered inside sidebar |
| `header/HeaderLogo` | Logo slot within `AppHeader` |
| `header/LanguageSwitcher` | Locale selector in header |
| `header/NotificationMenu` | Notification bell + dropdown in header |
| `header/UserMenu` | User avatar + dropdown menu in header |

### Modal Components

| Component | Description |
|-----------|-------------|
| `ModalManager` | Global modal registry and renderer |
| `ModalWrapper` | Standard modal chrome (title bar, close button, content slot) |

### Page Components

| Component | Description |
|-----------|-------------|
| `Default` | Default page wrapper with breadcrumb slot |
| `NotFound` | 404 not found page |
| `ServerError` | 500 server error page |
| `Maintenance` | Maintenance mode page |

---

## Admin App Components (`apps/admin/src/components/`)

| Component | Path | Description |
|-----------|------|-------------|
| `AccountsInviteModal` | `accounts/AccountsInviteModal.vue` | Modal form to invite/create a new admin account |
| `AdminDetailModal` | `accounts/AdminDetailModal.vue` | Modal to view and manage an existing admin's details |

---

## Seller App Components (`apps/seller/src/components/`)

| Component | Path | Description |
|-----------|------|-------------|
| `SelectProductsModal` | `marketing/SelectProductsModal.vue` | Modal for selecting products when creating a coupon |
| `NotificationRow` | `notifications/NotificationRow.vue` | Single notification list item |
| `ProductCell` | `orders/ProductCell.vue` | Product thumbnail + name cell in the orders table |

---

## Buyer App Components (`apps/buyer/src/components/`)

### Layout Components

| Component | Description |
|-----------|-------------|
| `AppHeader` | Buyer-specific top bar (overrides shared `AppHeader`) |
| `CartHeader` | Minimal header for the cart page |
| `CheckoutHeader` | Minimal header for the checkout page |
| `StorefrontHeader` | Full storefront navigation header (search, cart icon, account) |
| `StorefrontFooter` | Site footer with links and branding |
| `StorefrontLayout` | Root layout combining `StorefrontHeader`, slot, and `StorefrontFooter` |

### Home Page Components

| Component | Description |
|-----------|-------------|
| `CategoryBar` | Horizontal scrollable category navigation |
| `FlashSale` | Flash sale product carousel with countdown |
| `HeroBanner` | Full-width promotional hero banner (Swiper carousel) |
| `ProductCard` | Product summary card (image, name, price, rating) |
| `ProductGrid` | Responsive grid of `ProductCard` components |

### Checkout Components

| Component | Description |
|-----------|-------------|
| `ChangeAddressModal` | Modal to select or create a delivery address during checkout |

### Profile Components

| Component | Description |
|-----------|-------------|
| `AddressFormModal` | Modal form to create or edit a delivery address |
| `ProfileSidebar` | Left navigation sidebar for the profile/account section |

### Notification Components

| Component | Description |
|-----------|-------------|
| `NotificationRow` | Single notification list item (buyer variant) |

### Common Components

| Component | Description |
|-----------|-------------|
| `AvailableCouponPopover` | Popover listing available coupons for a store/product set |

---

## Composables Catalog (from `@hivespace/shared`)

All composables are exported from `packages/shared/src/index.ts` and available in any app via `import { useXxx } from '@hivespace/shared'`.

| Composable | Returns / Purpose |
|------------|-------------------|
| `useAppShell` | Sidebar open state, toggle methods |
| `useAsyncAction` | `{ execute, isLoading, error }` — wraps any async function |
| `useConfirmModal` | `{ confirm(options) }` — programmatic confirm dialog returning a Promise<boolean> |
| `useCooldown` | `{ start(ms), isActive }` — prevents rapid repeated actions |
| `useDebounce` | `debounce(fn, delay)` or `debouncedRef(ref, delay)` |
| `useFieldValidation` | Per-field dirty/touched/error tracking |
| `useFormatDate` | `formatDate(date, options?)`, locale-aware |
| `useModal` | `{ open(), close(), isOpen }` — controls a named modal |
| `useNotificationHub` | SignalR hub connection lifecycle (connect, disconnect) |
| `useNotificationRealtime` | `{ notifications }` — reactive array of `NotificationHubEvent` |
| `useNumberInputFormatter` | Formats numeric input as localized number string |
| `useSidebar` | Alias for sidebar open/close state (thin wrapper over `useAppShell`) |
| `useValidationRules` | Collection of reusable field validation rule functions |

---

## SVG Icon Components (`packages/shared/src/icons/`)

A large set of SVG icon components is exported from `@hivespace/shared` via its icons index. All icon components:
- Accept `size` (number, default 24) and `class` props
- Render as inline SVG
- Follow `PascalCase` naming convention

Exact icon inventory should be referenced from `packages/shared/src/icons/index.ts` as the set evolves. Common icon categories include: navigation, actions (edit, delete, add), status (check, warning, error), e-commerce (cart, bag, tag), and media (image, video).

---

## Feature Development: Component Order

When adding a new feature across any app, follow this order:

1. **Types** — define in `src/types/api/` and `src/types/store/`, export from `src/types/index.ts`
2. **Service** — create `src/services/{feature}.service.ts` extending `BaseService`
3. **Store** — create/update `src/stores/{feature}.ts` using Pinia Setup Store pattern
4. **Components** — build reusable UI in `src/components/{feature}/`
5. **View** — create page in `src/views/{Feature}Page.vue`
6. **Route** — add to `src/router/index.ts`
7. **i18n** — add keys to `src/i18n/locales/{en,vi}/{module}.json`
