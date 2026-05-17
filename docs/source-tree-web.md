# Source Tree: hivespace.web

Annotated directory tree for `apps/` and `packages/`. Entry points and key files are marked.

---

## Root

```
hivespace.web/
├── apps/                           # Application workspaces
├── packages/                       # Shared library workspaces
├── package.json                    # Root: turbo scripts, devDependencies (turbo, typescript, prettier, eslint)
├── pnpm-workspace.yaml             # Workspace definitions: 'apps/*', 'packages/*'
├── turbo.json                      # Turbo task pipeline (build, dev, type-check, lint, format)
├── eslint.config.ts                # Root ESLint config (extends @hivespace/eslint-config)
├── .npmrc                          # pnpm settings (e.g. node-linker)
└── README.md                       # Monorepo setup and commands
```

---

## apps/admin — @hivespace/admin (port 5173)

Platform admin dashboard for admins and system admins.

```
apps/admin/
├── src/
│   ├── App.vue                     # [ENTRY] Root component: SidebarProvider, RouterView, ModalManager, ToastContainer
│   ├── main.ts                     # [ENTRY] App bootstrap: Pinia, vfm, i18n, VueApexCharts, initializeAuth, router
│   ├── index.d.ts                  # Ambient type declarations
│   ├── vue.shims.d.ts              # Vue SFC type shim
│   │
│   ├── components/
│   │   └── accounts/
│   │       ├── AccountsInviteModal.vue   # Invite new admin modal
│   │       └── AdminDetailModal.vue      # Admin detail view/edit modal
│   │
│   ├── composables/                # (empty — uses @hivespace/shared composables only)
│   │
│   ├── config/
│   │   └── index.ts                # [KEY] AppConfig, createConfig(), buildApiUrl(), env-var validation
│   │
│   ├── i18n/
│   │   ├── index.ts                # createI18n() with lazy-loaded locale messages
│   │   └── locales/
│   │       ├── en/
│   │       │   ├── accounts.json
│   │       │   ├── admins.json
│   │       │   ├── audit-log.json
│   │       │   ├── backend-errors.json
│   │       │   ├── buyers.json
│   │       │   ├── common.json
│   │       │   ├── configuration.json
│   │       │   ├── dashboard.json
│   │       │   ├── icons.json
│   │       │   ├── layout.json
│   │       │   ├── merchants.json
│   │       │   ├── notification.json
│   │       │   └── users.json
│   │       └── vi/                 # Vietnamese mirrors of en/
│   │
│   ├── pages/
│   │   ├── DashboardPage.vue       # /dashboard — main admin overview
│   │   ├── DemoWrapper.vue         # /demo — dev-only demo route wrapper
│   │   ├── IconsPage.vue           # /demo/icons — icon showcase
│   │   ├── Accounts/
│   │   │   └── AccountsPage.vue    # /accounts — admin user management
│   │   ├── AuditLog/
│   │   │   └── AuditLogPage.vue    # /audit-log
│   │   ├── Buyers/
│   │   │   └── BuyersPage.vue      # /buyers — buyer user management
│   │   ├── Callback/
│   │   │   ├── LoginCallbackPage.vue   # /callback/login — OIDC signin callback
│   │   │   └── LogoutCallbackPage.vue  # /callback/logout — OIDC signout callback
│   │   ├── Configuration/
│   │   │   └── ConfigurationPage.vue   # /configuration
│   │   ├── Disputes/
│   │   │   └── DisputesPage.vue    # /disputes
│   │   ├── KycQueue/
│   │   │   └── KycQueuePage.vue    # /kyc-queue
│   │   ├── Merchants/
│   │   │   └── MerchantsPage.vue   # /merchants
│   │   └── ScheduledJobs/
│   │       └── ScheduledJobsPage.vue   # /scheduled-jobs
│   │
│   ├── router/
│   │   └── index.ts                # [KEY] createRouter + beforeEach admin auth guard
│   │
│   ├── services/
│   │   ├── api.ts                  # [KEY] ApiService singleton (no token refresh)
│   │   ├── admin.service.ts        # AdminService: GET/POST /admins, PUT/DELETE user status
│   │   ├── base.service.ts         # BaseService: delegates to apiService singleton
│   │   ├── index.ts                # Barrel re-export of all services
│   │   ├── notification.service.ts # createNotificationService factory wrapper
│   │   ├── profile.service.ts      # createUserProfileService factory wrapper
│   │   ├── user-settings.service.ts # createUserSettingsService factory wrapper
│   │   └── user.service.ts         # UserService: GET /admins/users, PUT status, DELETE
│   │
│   ├── stores/
│   │   ├── admin.store.ts          # [KEY] useAdminStore: admins[], pagination, CRUD
│   │   ├── index.ts                # Barrel re-export of all stores
│   │   ├── notification.store.ts   # useNotificationStore via createNotificationStore
│   │   ├── profile.store.ts        # useProfileStore via createUserProfileStore
│   │   ├── user-settings.store.ts  # useUserSettingsStore via createUserSettingsStore
│   │   └── user.store.ts           # useUserStore: users[], pagination, toggle status
│   │
│   └── types/
│       ├── admin.types.ts          # Admin, CreateAdminRequest/Response, GetAdminListQuery/Response
│       ├── index.ts                # Barrel re-export
│       ├── merchant.types.ts       # Merchant types
│       └── user.types.ts           # User, GetUserListQuery/Response
│
├── public/                         # Static assets (favicon, etc.)
├── .env.development                # [KEY] Dev defaults (OIDC client: adminportal, port 5173)
├── vite.config.ts                  # Vite config: vue, vueJsx, devtools, tailwindcss, workspaceAliases
├── tsconfig.json                   # References tsconfig.app.json + tsconfig.node.json
├── tsconfig.app.json               # Extends @hivespace/tsconfig/vue-app.json, paths: @/*
├── index.html                      # HTML shell: <div id="app">
├── package.json                    # @hivespace/admin dependencies
└── README.md                       # App-level README
```

---

## apps/seller — @hivespace/seller (port 5174)

Seller dashboard for product, order, coupon, and notification management.

```
apps/seller/
├── src/
│   ├── App.vue                     # [ENTRY] SidebarProvider, RouterView, ModalManager,
│   │                               #         ToastContainer, NotificationPreviewToast,
│   │                               #         useNotificationRealtime (SignalR active)
│   ├── main.ts                     # [ENTRY] Pinia, vfm, i18n, VueApexCharts, initializeAuth, router
│   ├── index.d.ts
│   ├── vue.shims.d.ts
│   │
│   ├── components/
│   │   ├── marketing/
│   │   │   └── SelectProductsModal.vue   # Product picker for coupon assignment
│   │   ├── notifications/
│   │   │   └── NotificationRow.vue       # Single notification list row
│   │   └── orders/
│   │       └── ProductCell.vue           # Product cell within order row
│   │
│   ├── config/
│   │   └── index.ts                # AppConfig, buildApiUrl(), env-var resolution
│   │
│   ├── i18n/
│   │   ├── index.ts
│   │   └── locales/
│   │       ├── en/
│   │       │   ├── backend-errors.json
│   │       │   ├── common.json
│   │       │   ├── coupon.json
│   │       │   ├── notification.json
│   │       │   ├── order.json
│   │       │   ├── pages.json
│   │       │   ├── product.json
│   │       │   ├── register-store.json
│   │       │   ├── verifyEmail.json
│   │       │   └── verifyEmailCallback.json
│   │       └── vi/                 # Vietnamese mirrors of en/
│   │
│   ├── pages/
│   │   ├── DemoWrapperPage.vue     # Dev-only demo wrapper
│   │   ├── IconsPage.vue           # Icon showcase
│   │   ├── RegisterStorePage.vue   # /register-seller — onboarding for new sellers
│   │   ├── VerifyEmailPage.vue     # /verify-email — email verification prompt
│   │   ├── Callback/
│   │   │   ├── LoginCallbackPage.vue
│   │   │   ├── LogoutCallbackPage.vue
│   │   │   └── VerifyEmailCallbackPage.vue   # /verify-email-callback
│   │   ├── Marketing/
│   │   │   ├── CouponListPage.vue    # /marketing/coupons
│   │   │   └── CouponDetailPage.vue  # /marketing/coupons/create, /marketing/coupons/detail/:id
│   │   ├── Notifications/
│   │   │   └── NotificationsPage.vue # /notifications
│   │   ├── Orders/
│   │   │   └── OrderManagementPage.vue  # /orders/all — tabbed order list + confirm/reject
│   │   └── Products/
│   │       ├── ProductListPage.vue     # /product/list
│   │       └── UpsertProductPage.vue   # /product/new, /product/:id
│   │
│   ├── router/
│   │   └── index.ts                # [KEY] createRouter + seller role/registration guard
│   │
│   ├── services/
│   │   ├── api.ts                  # [KEY] ApiService singleton with ensureFreshUser callback
│   │   ├── account.service.ts      # AccountService: email verification endpoints
│   │   ├── base.service.ts
│   │   ├── category.service.ts     # CategoryService: /categories, /categories/:id/attributes
│   │   ├── coupon.service.ts       # CouponService: full CRUD + end coupon
│   │   ├── index.ts
│   │   ├── media.service.ts        # createMediaUploadService factory
│   │   ├── notification.service.ts # createNotificationService factory
│   │   ├── order.service.ts        # OrderService: /orders/seller, confirm, reject
│   │   ├── product.service.ts      # ProductService: full CRUD /products
│   │   ├── profile.service.ts      # createUserProfileService factory
│   │   ├── refresh.service.ts      # Token refresh: delegates to createRefreshService
│   │   ├── store.service.ts        # StoreService: POST /stores
│   │   └── user-settings.service.ts
│   │
│   ├── stores/
│   │   ├── account.store.ts        # useAccountStore: email verification state
│   │   ├── coupon.store.ts         # [KEY] useCouponStore: list, CRUD, pagination
│   │   ├── index.ts
│   │   ├── media.store.ts          # useMediaStore via createMediaUploadStore
│   │   ├── notification.store.ts   # useNotificationStore (seller events)
│   │   ├── order.store.ts          # [KEY] useOrderStore: tabs, fetchOrders, confirm/cancel
│   │   ├── product.store.ts        # [KEY] useProductStore: CRUD, categories, attributes
│   │   ├── profile.store.ts        # useProfileStore via createUserProfileStore
│   │   ├── store.store.ts          # useStoreStore: registerStore
│   │   └── user-settings.store.ts
│   │
│   └── types/
│       ├── account.types.ts
│       ├── coupon.types.ts         # CouponDto, CreateCouponRequest, UpdateCouponRequest, etc.
│       ├── index.ts
│       ├── order.types.ts          # Order, OrderProcessStatus enum, GetOrderListQuery/Response
│       ├── product.types.ts        # [KEY] Product, ProductSku, ProductVariant, Category, CategoryAttribute
│       └── store.types.ts
│
├── public/
├── .env.development                # [KEY] Dev defaults (client: sellercenter, port 5174)
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── index.html
├── package.json
└── README.md
```

---

## apps/buyer — @hivespace/buyer (port 5175)

Customer-facing storefront for browsing, cart, checkout, and orders.

```
apps/buyer/
├── src/
│   ├── App.vue                     # [ENTRY] Conditional StorefrontLayout, ModalManager,
│   │                               #         ToastContainer, NotificationPreviewToast,
│   │                               #         useNotificationRealtime
│   ├── main.ts                     # [ENTRY] Pinia, vfm, i18n, initializeAuth, router
│   ├── index.d.ts
│   ├── vue.shims.d.ts
│   ├── style.css                   # App-level global styles
│   │
│   ├── components/
│   │   ├── checkout/
│   │   │   ├── ChangeAddressModal.vue        # Delivery address picker at checkout
│   │   │   └── AvailableCouponPopover.vue    # Store coupon picker popover
│   │   ├── common/                           # (no local files — uses @hivespace/shared)
│   │   ├── home/
│   │   │   ├── CategoryBar.vue              # Homepage category icon strip
│   │   │   ├── FlashSale.vue                # Flash sale countdown + products
│   │   │   ├── HeroBanner.vue               # Swiper hero carousel
│   │   │   ├── ProductCard.vue              # Product thumbnail card
│   │   │   └── ProductGrid.vue              # Grid wrapper for ProductCard list
│   │   ├── layout/
│   │   │   ├── AppHeader.vue                # Auth-section header
│   │   │   ├── CartHeader.vue               # Minimal header for /cart
│   │   │   ├── CheckoutHeader.vue           # Minimal header for /checkout
│   │   │   ├── StorefrontFooter.vue         # Full site footer
│   │   │   ├── StorefrontHeader.vue         # Main storefront nav header
│   │   │   └── StorefrontLayout.vue         # Page shell: header + slot + footer
│   │   ├── notifications/
│   │   │   └── NotificationRow.vue          # Notification list row
│   │   └── profile/
│   │       ├── AddressFormModal.vue         # Create/edit address modal
│   │       └── ProfileSidebar.vue           # Profile section left nav
│   │
│   ├── composables/
│   │   └── useAsyncAction.ts                # Local async action wrapper (also in shared)
│   │
│   ├── config/
│   │   └── index.ts                # AppConfig, buildApiUrl()
│   │
│   ├── i18n/
│   │   ├── index.ts
│   │   └── locales/
│   │       ├── en/                 # English locale files
│   │       ├── vi/                 # Vietnamese locale files
│   │       ├── address.json
│   │       ├── cart.json
│   │       ├── categories.json
│   │       ├── checkout.json
│   │       ├── common.json
│   │       ├── footer.json
│   │       ├── notification.json
│   │       ├── orders-page.json
│   │       ├── orders.json
│   │       ├── payment.json
│   │       ├── productDetail.json
│   │       └── profile-page.json
│   │
│   ├── pages/
│   │   ├── Callback/
│   │   │   ├── LoginCallbackPage.vue
│   │   │   └── LogoutCallbackPage.vue
│   │   ├── Account/
│   │   │   └── OrderDetailPage.vue     # /account/orders/:id
│   │   ├── Cart/
│   │   │   └── CartPage.vue            # /cart
│   │   ├── Checkout/
│   │   │   └── CheckoutPage.vue        # /checkout
│   │   ├── Home/
│   │   │   └── HomePage.vue            # / (anonymous)
│   │   ├── Notifications/
│   │   │   └── NotificationsPage.vue   # /notifications
│   │   ├── Payment/
│   │   │   └── PaymentResultPage.vue   # /payment/result (anonymous)
│   │   ├── Product/
│   │   │   └── ProductDetailPage.vue   # /product (anonymous)
│   │   └── Profile/
│   │       ├── ProfilePage.vue         # /profile
│   │       ├── AddressPage.vue         # /profile/address
│   │       └── OrdersPage.vue          # /orders
│   │
│   ├── router/
│   │   └── index.ts                # [KEY] createRouter + basic auth guard (login redirect only)
│   │
│   ├── services/
│   │   ├── api.ts                  # [KEY] ApiService singleton with ensureFreshUser
│   │   ├── address.service.ts      # AddressService: /users/address CRUD
│   │   ├── base.service.ts
│   │   ├── cart.service.ts         # [KEY] CartService: summary, items, coupons
│   │   ├── category.service.ts     # CategoryService: /categories/homepage
│   │   ├── checkout.service.ts     # CheckoutService: /orders/checkout/preview, /orders/checkout
│   │   ├── coupon.service.ts       # CouponService: /coupons/available
│   │   ├── notification.service.ts
│   │   ├── order.service.ts        # OrderService: /orders, /orders/:id
│   │   ├── payment.service.ts      # PaymentService: /payments/:id, /payments/by-order/:id
│   │   ├── product.service.ts      # ProductService: /products/summaries, /products/detail/:id
│   │   ├── refresh.service.ts
│   │   ├── user-settings.service.ts
│   │   └── user.service.ts         # UserService: /users/me GET/PUT
│   │
│   ├── stores/
│   │   ├── address.store.ts        # useAddressStore: address list, default
│   │   ├── cart.store.ts           # [KEY] useCartStore: full cart state, coupons, summaries
│   │   ├── category.store.ts       # useCategoryStore: homepage categories
│   │   ├── checkout.store.ts       # [KEY] useCheckoutStore: preview, submit, coupon state
│   │   ├── coupon-equality.ts      # Pure equality helpers for coupon change detection
│   │   ├── coupon.store.ts         # useCouponStore: available coupons
│   │   ├── index.ts
│   │   ├── notification.store.ts   # useNotificationStore (buyer events)
│   │   ├── orders.store.ts         # useOrdersStore: list, detail, tab/search/loadMore
│   │   ├── payment.store.ts        # usePaymentStore: payment result state
│   │   ├── product.store.ts        # useProductStore: list, detail
│   │   ├── profile.store.ts        # useProfileStore: GET/PUT /users/me, form binding
│   │   └── user-settings.store.ts
│   │
│   └── types/
│       ├── address.types.ts        # UserAddress, AddressApiPayload
│       ├── cart.types.ts           # CartGroup, CartItem, CartSummary, coupon applied types
│       ├── category.types.ts       # HomepageCategory
│       ├── checkout.types.ts       # CheckoutPreview, CheckoutRequest, CheckoutResult, DeliveryPackage
│       ├── coupon.types.ts         # GetAvailableCouponsResponse, CouponSummary
│       ├── index.ts
│       ├── order.types.ts          # Order, OrderDetail, CustomerOrderProcessStatus
│       ├── payment.types.ts        # PaymentDto
│       ├── product.types.ts        # ProductSummary, GetProductDetailResponse, GetProductListQuery/Response
│       └── profile.types.ts        # UserProfileResponse, UpdateProfileRequest, ProfileFormData
│
├── public/
├── .env.development                # [KEY] Dev defaults (client: storefront, port 5175)
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── index.html
└── package.json
```

---

## packages/shared — @hivespace/shared (v1.1.20)

Shared component library, composables, feature modules, and utilities.

```
packages/shared/
├── src/
│   ├── index.ts                    # [ENTRY] Public re-export → internal.ts
│   ├── internal.ts                 # [ENTRY] Barrel: re-exports components, composables, types,
│   │                               #         utils, i18n, pages, icons, stores, features
│   ├── env.d.ts                    # Vite env type declarations
│   │
│   ├── components/
│   │   ├── index.ts
│   │   ├── charts/
│   │   │   ├── BarChart/           # ApexCharts bar chart wrapper
│   │   │   ├── LineChart/          # ApexCharts line chart wrapper
│   │   │   └── index.ts
│   │   ├── common/                 # [KEY] All reusable UI primitives
│   │   │   ├── Alert.vue, Avatar.vue, Badge.vue, Button.vue, Checkbox.vue
│   │   │   ├── ConfirmModal.vue, CountDown.vue, DatePicker.vue, DateTimePicker.vue
│   │   │   ├── DropdownMenu.vue, Dropzone.vue, FileInput.vue, FilterChips.vue
│   │   │   ├── FullscreenLoader.vue, Input.vue, Link.vue, Modal.vue
│   │   │   ├── MultipleSelect.vue, NotificationPreviewToast.vue, OrderTimeline.vue
│   │   │   ├── PageBreadcrumb.vue, Pagination.vue, QuantityControl.vue
│   │   │   ├── Radio.vue, RadioGroup.vue, ResponsiveImage.vue, Select.vue
│   │   │   ├── Spinner.vue, Tabs.vue, TextArea.vue, ThemeToggler.vue
│   │   │   ├── TimePicker.vue, Toast.vue, ToastContainer.vue, ToggleSwitch.vue
│   │   │   ├── TwoColumnImageGrid.vue, ThreeColumnImageGrid.vue
│   │   │   ├── YouTubeEmbed.vue, CommonGridShape.vue, v-click-outside.vue
│   │   │   ├── ComponentCard.vue
│   │   │   └── index.ts
│   │   ├── layout/                 # [KEY] App shell layout components
│   │   │   ├── AppShell.vue        # Main layout shell
│   │   │   ├── AppSidebar.vue      # Collapsible sidebar
│   │   │   ├── AppHeader.vue       # Top header bar
│   │   │   ├── SidebarProvider.vue # Sidebar context provider
│   │   │   ├── Backdrop.vue        # Mobile sidebar overlay
│   │   │   ├── FullScreenLayout.vue
│   │   │   ├── SidebarWidget.vue
│   │   │   ├── header/
│   │   │   │   ├── HeaderLogo.vue
│   │   │   │   ├── LanguageSwitcher.vue
│   │   │   │   ├── NotificationMenu.vue
│   │   │   │   ├── UserMenu.vue
│   │   │   │   └── index.ts
│   │   │   └── index.ts
│   │   ├── modal/
│   │   │   ├── ModalManager.vue    # [KEY] Global modal outlet (placed in App.vue)
│   │   │   ├── ModalWrapper.vue    # Modal chrome wrapper
│   │   │   └── index.ts
│   │   └── tables/
│   │       ├── basic-tables/       # BasicTable component
│   │       └── index.ts
│   │
│   ├── composables/                # [KEY] Shared composable hooks
│   │   ├── index.ts
│   │   ├── useAppShell.ts          # provideAppShell / useAppShell (DI for shell context)
│   │   ├── useAsyncAction.ts       # isLoading + run(fn) wrapper
│   │   ├── useAuth.ts              # [KEY] OIDC auth: login, logout, getCurrentUser, etc.
│   │   ├── useConfirmModal.ts      # Convenience wrapper around useModal for confirm dialogs
│   │   ├── useCooldown.ts          # Timer-based cooldown (e.g., resend code buttons)
│   │   ├── useDebounce.ts          # Generic debounce composable
│   │   ├── useFieldValidation.ts   # Field-level validation state
│   │   ├── useFormatDate.ts        # Date formatting helpers
│   │   ├── useModal.ts             # [KEY] Global modal singleton: openModal/closeModal
│   │   ├── useNotificationHub.ts   # [KEY] SignalR connection management
│   │   ├── useNumberInputFormatter.ts  # Numeric input formatting
│   │   ├── useSidebar.ts           # [KEY] Sidebar context: useSidebarProvider / useSidebar
│   │   └── useValidationRules.ts   # Reusable validation rule builders
│   │
│   ├── features/                   # [KEY] Shared feature modules (factory pattern)
│   │   ├── index.ts
│   │   ├── service.types.ts        # ApiRequestService, BuildApiUrl interfaces
│   │   ├── auth/
│   │   │   ├── refresh.service.ts  # [KEY] createRefreshService — token refresh via /connect/token
│   │   │   └── index.ts
│   │   ├── media-upload/
│   │   │   ├── createMediaUploadStore.ts  # Factory: Pinia upload store
│   │   │   ├── media-upload.service.ts    # createMediaUploadService: presign + confirm + blob PUT
│   │   │   ├── media-upload.types.ts      # PresignUrlRequest/Response, IMediaUploadService
│   │   │   └── index.ts
│   │   ├── notifications/
│   │   │   ├── createNotificationStore.ts # [KEY] Factory: notification Pinia store
│   │   │   ├── notifications.service.ts   # createNotificationService: GET/PUT /notifications
│   │   │   ├── notifications.types.ts     # NotificationDto, InAppNotification, NotificationHubEvent
│   │   │   ├── useNotificationRealtime.ts # [KEY] SignalR lifecycle composable
│   │   │   └── index.ts
│   │   ├── user-profile/
│   │   │   ├── createUserProfileStore.ts  # Factory: profile Pinia store
│   │   │   ├── user-profile.service.ts    # createUserProfileService: GET /users/me
│   │   │   ├── user-profile.types.ts      # GetMyProfileResponse, IUserProfileService
│   │   │   └── index.ts
│   │   └── user-settings/
│   │       ├── createUserSettingsStore.ts # Factory: settings Pinia store
│   │       ├── user-settings.service.ts   # createUserSettingsService: GET/PUT /users/settings
│   │       ├── user-settings.types.ts     # UserSettings, CultureText, ThemeText, CULTURE_TEXT, THEME_TEXT
│   │       └── index.ts
│   │
│   ├── i18n/
│   │   └── index.ts                # Shared i18n messages (component-level keys, pagination, etc.)
│   │
│   ├── icons/                      # [KEY] 70+ SVG icon components
│   │   ├── index.ts                # Exports all icons by name
│   │   ├── GridIcon.vue, UserCircleIcon.vue, BoxIcon.vue, ListIcon.vue, ...
│   │   └── (70+ .vue files)
│   │
│   ├── pages/                      # Shared route pages used by all apps
│   │   ├── index.ts
│   │   ├── Default.vue             # / route: auth redirect handler
│   │   ├── Maintenance.vue         # /maintenance — 503 page
│   │   ├── NotFound.vue            # /:pathMatch(.*)* — 404 page
│   │   └── ServerError.vue         # /server-error — 500 page
│   │
│   ├── stores/
│   │   ├── index.ts
│   │   └── app.store.ts            # [KEY] useAppStore: isLoading, theme, toast notifications
│   │
│   ├── styles/
│   │   ├── main.css                # Full stylesheet entry (imported as @hivespace/shared/style.css)
│   │   ├── theme.css               # Tailwind @theme CSS variables
│   │   ├── base.css                # Base/reset styles
│   │   └── utilities.css           # Tailwind utility overrides
│   │
│   ├── types/
│   │   ├── index.ts                # Barrel: all types
│   │   ├── api.types.ts            # ApiConfig
│   │   ├── app-user.ts             # [KEY] AppUser interface + toAppUser() converter
│   │   ├── common.types.ts         # [KEY] ApiResponse, PaginationMetadata, Status, UserType, enums
│   │   ├── component.common.ts     # MenuItem, SelectOption, ModalProps, etc.
│   │   ├── coupon.types.ts         # Shared coupon type fragments
│   │   ├── sidebar.types.ts        # SidebarMenuGroup, SidebarMenuItem, SidebarSubMenuItem
│   │   └── util.type.ts            # Nullable, Optional, LoadingState, Environment, etc.
│   │
│   └── utils/
│       ├── index.ts
│       ├── api.ts                  # [KEY] ApiService class (axios with interceptors, retry, auth)
│       ├── base.service.ts         # BaseService abstract class
│       ├── config.ts               # validateUrl, validateEnvironment, parseBoolean, parseNumber, joinUrl
│       ├── cookie.ts               # getCookie(name)
│       └── theme.ts                # Theme utilities
│
├── package.json                    # @hivespace/shared: exports, peerDeps, devDeps
└── vite.config.ts                  # Library build config (vite-plugin-dts, @tailwindcss/vite)
```

---

## packages/demo — @hivespace/demo

Dev-only demo routes and pages. Imported only when `import.meta.env.DEV === true`. Not documented further as it is not production code.

---

## packages/eslint-config — @hivespace/eslint-config

```
packages/eslint-config/
├── index.js    # Shared ESLint flat config (extends eslint-plugin-vue + typescript-eslint)
└── package.json
```

---

## packages/tsconfig — @hivespace/tsconfig

```
packages/tsconfig/
├── base.json      # Shared compilerOptions: ES2020, strict, bundler moduleResolution
├── vue-app.json   # Extends base.json; adds Vue-specific options
├── node.json      # Node.js-targeted tsconfig
└── package.json
```

---

## packages/vite-config — @hivespace/vite-config

```
packages/vite-config/
├── index.ts    # [KEY] workspaceAliases() — returns Vite alias array for @hivespace/shared and @hivespace/demo
└── package.json
```

---

## Key File Index

| File | Purpose |
|------|---------|
| `apps/admin/src/main.ts` | Admin app bootstrap |
| `apps/seller/src/main.ts` | Seller app bootstrap |
| `apps/buyer/src/main.ts` | Buyer app bootstrap |
| `apps/admin/src/App.vue` | Admin root component (no SignalR) |
| `apps/seller/src/App.vue` | Seller root component (SignalR active) |
| `apps/buyer/src/App.vue` | Buyer root component (SignalR active, layout toggle) |
| `apps/*/src/router/index.ts` | Per-app route definitions + auth guard |
| `apps/*/src/services/api.ts` | Per-app ApiService singleton |
| `apps/*/src/config/index.ts` | Per-app AppConfig + buildApiUrl |
| `packages/shared/src/index.ts` | Library public entry point |
| `packages/shared/src/internal.ts` | Internal barrel (avoids circular deps) |
| `packages/shared/src/utils/api.ts` | ApiService class with interceptors |
| `packages/shared/src/composables/useAuth.ts` | OIDC auth composable |
| `packages/shared/src/composables/useModal.ts` | Global modal singleton |
| `packages/shared/src/composables/useNotificationHub.ts` | SignalR hub connection |
| `packages/shared/src/stores/app.store.ts` | useAppStore (loading, theme, toasts) |
| `packages/shared/src/features/auth/refresh.service.ts` | Token refresh logic |
| `packages/shared/src/features/notifications/createNotificationStore.ts` | Notification store factory |
| `packages/shared/src/features/notifications/useNotificationRealtime.ts` | SignalR lifecycle composable |
| `packages/shared/src/types/app-user.ts` | AppUser + role helpers |
| `packages/shared/src/types/common.types.ts` | Shared API/pagination types, enums |
| `packages/vite-config/index.ts` | workspaceAliases() for all apps |
| `turbo.json` | Turbo task pipeline |
| `pnpm-workspace.yaml` | Workspace definition |
