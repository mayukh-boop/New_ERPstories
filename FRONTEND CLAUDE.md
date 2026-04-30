# CLAUDE.md — c-edge-ui

## Project Overview

A React 19 SPA for the C-Edge ERP platform. Covers authentication flows (login, registration, forgot password, tenant selection), a collapsible dashboard layout with API-driven sidebar navigation, and role-based access control.

Backend proxies to `http://localhost:8080/cedge` during development.

---

## Tech Stack

| Layer | Library | Version |
|---|---|---|
| Framework | React | 19.2.4 |
| Language | TypeScript | 6.0.2 (strict) |
| Build tool | Vite | 8.0.7 |
| Routing | React Router DOM | 7.14.0 |
| State | Zustand (with `persist`) | 5.0.12 |
| HTTP | Axios | 1.14.0 |
| Server state | TanStack React Query | 5.96.2 |
| Forms | React Hook Form | 7.72.1 |
| Validation | Zod | 4.3.6 |
| Form resolvers | @hookform/resolvers | 5.2.2 |
| Styling | Tailwind CSS | 4.2.2 |
| Class merging | clsx + tailwind-merge | 2.1.1 / 3.5.0 |
| Icons | Lucide React | 1.7.0 |
| Material icons | Google Material Symbols Outlined | (CDN — `index.html`) |
| Geo dropdowns | react-select + country-state-city | 5.10.2 / 3.2.1 |

---

## Commands

```bash
npm run dev       # Dev server → http://localhost:5173
npm run build     # Production build → dist/
npm run preview   # Preview production build
npm run lint      # ESLint check
```

---

## Project Structure

```
src/
├── api/
│   ├── axios.ts              # Axios instance + interceptors (auth attach, 401 → cookie refresh)
│   ├── auth/
│   │   └── index.ts          # All auth API functions
│   └── menu/
│       └── index.ts          # GET /menu → MenuModuleDTO[]
├── components/
│   ├── layout/
│   │   ├── DashboardLayout.tsx  # Shell: sidebar + header + <Outlet />, sidebar toggle button
│   │   ├── Sidebar.tsx          # Collapsible sidebar with API-driven nav + logout
│   │   └── Header.tsx           # Top bar: breadcrumb, user avatar (first letter of username)
│   └── ui/
│       ├── Button.tsx
│       └── Input.tsx
├── hooks/
│   └── useMenu.ts            # React Query hook → GET /api/v1/menu (10-min staleTime)
├── pages/
│   ├── auth/
│   │   ├── LoginPage.tsx         # 2-step login (email/pass → auto/manual tenant select)
│   │   ├── RegisterPage.tsx      # 3-step registration (OTP → account → company)
│   │   ├── ForgotPasswordPage.tsx # 3-step forgot password (OTP → verify → reset)
│   │   └── TenantSelectionPage.tsx # Manual tenant picker for multi-tenant users
│   └── dashboard/
│       └── DashboardPage.tsx     # "Coming Soon" placeholder
├── store/
│   └── authStore.ts          # Zustand store — accessToken in memory only; user info persisted
├── types/
│   ├── auth.ts               # All auth DTOs and API response types
│   └── menu.ts               # MenuModuleDTO, MenuLinkItem
├── lib/
│   └── utils.ts              # cn() helper (clsx + tailwind-merge)
└── App.tsx                   # Router setup + ProtectedRoute + session recovery on startup
```

---

## Path Alias

Use `@/` for all imports from `src/`:

```ts
import { useAuthStore } from '@/store/authStore'
import type { UserDTO } from '@/types/auth'
```

---

## Auth Flow

### Login (2 steps)
1. `POST /auth/login` → `preAuthToken` + list of `tenants`
2. `POST /auth/select-tenant` → `accessToken` in body; `refreshToken` set as HttpOnly cookie by backend
3. Single tenant: auto-selects immediately. Multi-tenant: navigates to `/select-tenant`

### Registration (3 steps)
1. `POST /auth/send-otp` — send OTP to email
2. `POST /auth/verify-otp` → `registrationToken`
3. `POST /auth/register` with company details — no tokens returned; user logs in separately

### Forgot Password (3 steps)
1. `POST /auth/forgot-password` — send reset OTP
2. `POST /auth/verify-reset-otp` → `passwordResetToken`
3. `POST /auth/reset-password` — all sessions revoked; user must log in again

### Token Strategy
- **Access token**: stored in Zustand memory only (never localStorage). Lost on page refresh — recovered via cookie.
- **Refresh token**: stored as HttpOnly cookie set by the backend. Never visible to JavaScript.
- **Session recovery**: `App.tsx` calls `POST /auth/refresh` on mount. If the cookie is valid, a new `accessToken` is returned and stored in memory. Shows a spinner until recovery completes.

### Token Refresh (401 interceptor)
- Axios response interceptor catches 401s on protected endpoints
- Calls `POST /auth/refresh` with **no body** — browser sends the `refresh_token` cookie automatically (`withCredentials: true`)
- On success: stores new `accessToken` in memory, retries the original request
- On failure: clears auth state, redirects to `/login`

### Protected Routes
`ProtectedRoute` in `App.tsx` checks `accessToken` in Zustand store. No token → redirect to `/login`. The `isRestoring` flag in `App.tsx` prevents premature redirect while session recovery is in progress.

---

## State Management (Zustand)

Store: `src/store/authStore.ts`

**Persisted fields** (localStorage key `cedge-auth` — no secrets):
- `userId`, `username`, `email`, `tenantSchema`, `role`

**Memory-only fields** (cleared on page reload, restored via `/auth/refresh`):
- `accessToken`

**Transient fields** (never persisted):
- `preAuthToken`, `tenants`, `firstLogin`

**Actions:**
| Action | Signature | Purpose |
|---|---|---|
| `setPreAuth` | `(preAuthToken, tenants)` | After step-1 login |
| `setTokens` | `(accessToken)` | After silent token refresh |
| `setSession` | `(accessToken, userId, username, email, tenantSchema, role, firstLogin)` | After select-tenant / force-login |
| `clearAuth` | `()` | Logout / auth failure |

---

## API Layer

`src/api/axios.ts` — base URL `/cedge/api/v1`, `withCredentials: true`.

All auth endpoints in `src/api/auth/index.ts`:

| Function | Method | Notes |
|---|---|---|
| `login(email, password)` | POST /auth/login | Returns preAuthToken + tenants |
| `selectTenant(preAuthToken, schema, deviceInfo?)` | POST /auth/select-tenant | Returns AuthResponse; sets refresh cookie |
| `forceLogin(forceLoginToken, deviceInfo?)` | POST /auth/force-login | Returns AuthResponse; sets refresh cookie |
| `register(payload)` | POST /auth/register | Returns void |
| `refresh()` | POST /auth/refresh | No body — cookie sent automatically |
| `logout()` | POST /auth/logout | No body — cookie sent automatically; backend clears cookie |
| `sendEmailOtp(email)` | POST /auth/send-otp | |
| `verifyEmailOtp(email, otp)` | POST /auth/verify-otp | Returns registrationToken |
| `forgotPassword(email)` | POST /auth/forgot-password | |
| `verifyResetOtp(email, otp)` | POST /auth/verify-reset-otp | Returns passwordResetToken |
| `resetPassword(token, newPassword)` | POST /auth/reset-password | |

API responses are typed as `ApiResponse<T>` with `status: 'SUCCESS' | 'ERROR' | 'SESSION_LIMIT_REACHED'`.

---

## Dashboard Layout

`DashboardLayout.tsx` wraps all `/dashboard/*` routes:
- Fixed sidebar (collapsible to 64 px icon-only via toggle button)
- Fixed header (`Header.tsx`) — user avatar shows first letter of username
- `<Outlet />` renders the active page in a scrollable main area
- Toggle button floats on the right edge of the sidebar, aligned to the header bottom border

### Sidebar (`Sidebar.tsx`)
- **Dashboard** link hardcoded at the top (icon: `dashboard_customize`)
- **Module links** loaded from `GET /api/v1/menu` via `useMenu` hook (10-min cache)
- **Bottom links**: User Profile, Settings, Logout
- Logout calls `authApi.logout()` (no args — cookie sent automatically), then `clearAuth()` + navigate to `/login`
- Collapsed state: icons visible, text hidden, pointer-events disabled
- Smooth transitions: `opacity + max-width` for text, `max-height + opacity` for sub-menus

### Navigation Icons
Uses **Google Material Symbols Outlined** loaded via CDN in `index.html`. Module icons come from `moduleIcon` field in the menu API response (Google Fonts icon names). Never use Lucide icons for sidebar nav items.

---

## CSS Variables & Theming

All nav/sidebar colours are CSS custom properties in `src/index.css` so users can retheme without touching component code:

```css
:root {
  --sidebar-bg: #ffffff;
  --sidebar-width: 240px;
  --nav-text-primary: #111827;
  --nav-text-secondary: #6b7280;
  --brand-accent: #2563eb;
  --nav-active-text: var(--brand-accent);
  --nav-active-bg: #eff6ff;
  --nav-hover-text: var(--brand-accent);
  --nav-hover-bg: #eff6ff;
  --app-bg: #f0f2f7;
  --card-bg: #ffffff;
}
```

Hover states use `.nav-item:hover` and `.nav-sub-link:hover` CSS classes (not Tailwind `hover:`) so they pick up CSS variable colours.

---

## Styling Conventions

- Tailwind CSS only — no component library (no Shadcn, no MUI)
- Brand colors: primary `#2563eb` (blue), secondary `#FF9900` (orange)
- Backgrounds: `--app-bg: #f0f2f7`
- Form fields use bottom-border style (not box borders)
- Use `cn()` from `@/lib/utils` to merge Tailwind classes conditionally

---

## Forms

- Use **React Hook Form** + **Zod** resolver for all forms
- Multi-step pages use separate `useForm` instances (`form1`, `form2`, etc.)
- Password requirements: 8+ chars, uppercase, lowercase, digit, special character

---

## TypeScript

- Strict mode is on — no `any` without suppression comment
- Auth shapes in `src/types/auth.ts`; nav shapes in `src/types/menu.ts`
- Key auth types: `UserDTO`, `TenantDTO`, `AuthResponse`, `PreAuthResponse`, `RegisterRequest`, `ApiResponse<T>`
- `AuthResponse` does NOT contain `refreshToken` — it is set as an HttpOnly cookie by the backend
- Roles: `'SUPER_ADMIN' | 'ADMIN' | 'USER'`

---

## User Management (Login Management Module)

Three pages under `/login-management/users`:

| Route | Component | File |
|-------|-----------|------|
| `/login-management/users` | `UserManagementPage` | `src/pages/admin/users/UserManagementPage.tsx` |
| `/login-management/users/new` | `CreateUserPage` | `src/pages/admin/users/CreateUserPage.tsx` |
| `/login-management/users/:id/edit` | `EditUserPage` | `src/pages/admin/users/EditUserPage.tsx` |

### Authorization guard (all 3 pages)
```tsx
const role = useAuthStore((s) => s.role)
if (role !== 'SUPER_ADMIN' && role !== 'ADMIN') return <NotAuthorizedUI />
```

### SUPER_ADMIN vs ADMIN behaviour
- **SUPER_ADMIN**: sees all users (bypasses tenant filter), can assign any company, SUPER_ADMIN role option visible in role dropdown
- **ADMIN**: sees own-tenant users only, company field locked to own company, SUPER_ADMIN role hidden from dropdown

### Supporting files
| File | Purpose |
|------|---------|
| `src/types/user.ts` | `PageResponse<T>`, `UserListDTO`, `UserTenantMembershipDTO`, `CompanyListItem`, `RoleOption`, `TenantMembershipRequest`, `CreateUserRequest`, `UpdateUserRequest` |
| `src/api/user/index.ts` | `userApi` — `getUsers`, `getUserById`, `createUser`, `updateUser`, `disableUser`, `getCompanyList`, `getRolesByCompany` |
| `src/hooks/useUsers.ts` | React Query hooks — `useUserList`, `useUserDetail`, `useCompanyList`, `useRolesByCompany`, `useCreateUser`, `useUpdateUser`, `useDisableUser` |
| `src/lib/selectStyles.ts` | Shared react-select bottom-border styles (`selectStyles`, `selectStylesDisabled`) — do NOT modify `RegisterPage.tsx` |
| `src/lib/userFormSchemas.ts` | Zod schemas — `createUserSchema`, `editUserSchema`, `tenantMembershipSchema` |

### Key API endpoints used
- `GET /api/v1/users?page=&size=&search=` — paginated user list
- `GET /api/v1/users/:id` — single user with tenantMemberships
- `POST /api/v1/users` — create user (requires ADMIN/SUPER_ADMIN)
- `PUT /api/v1/users/:id` — update user (requires ADMIN/SUPER_ADMIN)
- `PATCH /api/v1/users/:id/disable` — disable user
- `GET /api/v1/companies/list` — flat company list for dropdowns
- `GET /api/v1/roles?companyId=&size=100` — roles scoped to a company

---

## Image Upload — Cloudflare CDN Pattern

All image uploads in Master pages go through the backend; images are **served directly from Cloudflare CDN**.

### API Layer

| File | Endpoint | Purpose |
|---|---|---|
| `src/api/upload/index.ts` | `POST /upload/image` | Upload `File` as `multipart/form-data`; backend stores in Cloudflare R2/Images and returns `{ url, assetId }` |
| `src/api/upload/index.ts` | `DELETE /upload/image/:assetId` | Backend deletes the Cloudflare asset |

### Two-phase save flow (in every Master page with an image field)

```
Phase 1 — uploading:  uploadApi.uploadImage(file)  → { url, assetId }
Phase 2 — saving:     brandApi.create({ ..., logoUrl: url, logoAssetId: assetId })
```

### Rules

- **Never** upload directly to Cloudflare from the frontend — always use `POST /upload/image`
- **Always** use the Cloudflare `url` as `<img src>` — never the backend URL
- **Always** store both `logoUrl` (for display) and `logoAssetId` (for deletion) in the entity
- Use `savePhase: 'idle' | 'uploading' | 'saving'` to show distinct progress states to the user
- The `LogoUpload` widget in BrandMasterPage is the canonical reference for this pattern

### Types

```ts
// src/types/masters.ts
interface BrandDTO {
  logoUrl:     string | null  // Cloudflare CDN URL — use as <img src>
  logoAssetId: string | null  // Cloudflare asset ID — needed to delete old logo
}
```

Full rules: [`src/components/ui/MASTER_COMPONENTS.md`](src/components/ui/MASTER_COMPONENTS.md#image-upload--cloudflare-storage-flow)

---

## Master Form Components

All **Master** screens (Country, Currency, Product, etc.) must use the shared components in `src/components/ui/`. Full rules: [`src/components/ui/MASTER_COMPONENTS.md`](src/components/ui/MASTER_COMPONENTS.md).

### Available components

| Component | File | Use for |
|---|---|---|
| `MasterFormField` | `ui/MasterFormField.tsx` | **Primary entry point** — every field in a Master form |
| `Dropdown` | `ui/Dropdown.tsx` | Single-select (used internally by `MasterFormField`) |
| `MultiSelect` | `ui/MultiSelect.tsx` | Multi-select with tag chips (used internally) |
| `ImageUpload` | `ui/ImageUpload.tsx` | Image file upload (used internally) |

### Field types supported by `MasterFormField`

| `fieldType` | Behaviour |
|---|---|
| `'text'` (default) | Free-text input |
| `'numeric'` | Integer only — blocks non-digit keys; `inputMode="numeric"` |
| `'decimal'` | Digits + one `.`; max **2 decimal places** enforced; `inputMode="decimal"` |
| `'dropdown'` | Single-select; requires `options: { value, label }[]` |
| `'multiselect'` | Multi-select; `value: string[]`; tag chips for selected items |
| `'image'` | Drag-and-drop / click upload; JPG/PNG/WebP/SVG; default max **2 MB** |

### Rules to follow in every Master page

1. Use `MasterFormField` — never raw `<input>` / `<select>` / `<textarea>`
2. Set `required={true}` on mandatory fields (renders red `*`)
3. Always back mandatory fields with a Zod schema rule
4. Use `register()` for `text` / `numeric` / `decimal`; use `Controller` for `dropdown` / `multiselect` / `image`
5. Pass the Zod error message to the `error` prop

---

## What Is Not Yet Built

- Full dashboard widgets / analytics
- Environment variable setup (`.env` file — not required; proxy is hardcoded)

---

## Vite Proxy (Dev Only)

Requests to `/cedge/*` are proxied to `http://localhost:8080`. No `VITE_API_BASE_URL` env var — base path is hardcoded in `src/api/axios.ts`.

---

## CORS Note

The backend CORS config allows only `http://localhost:5173` (required because `withCredentials: true` forbids wildcard origins). If the dev port changes, update `SecurityConfig.java` → `allowedOrigins`.

---

## ESLint Notes

- Unused variables prefixed with `_` are allowed
- `@typescript-eslint/no-explicit-any` is a warning, not an error
- React Hooks rules are enforced
