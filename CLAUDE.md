# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Purpose of This Repository

This is the **requirements and design center** for the C-Edge ERP project. It does NOT contain application source code. It contains:

- User story documents (feature specifications)
- HTML/CSS UI reference prototypes (screen mockups)
- A shared CSS design system (`design-system.css`)
- Architecture reference guides for both application layers

Anything produced here — user stories, Jira tickets, HTML mockups, API contracts, DB schema proposals, UI flows — must be grounded in the actual tech stack and patterns used in the real project.

---

## MANDATORY: Read Before Creating Anything

**Before writing any user story, Jira ticket, HTML prototype, API design, DB schema, or UI flow — always read the relevant architecture guide first.**

| What you are working on | Read this first |
|------------------------|----------------|
| API endpoints, DB tables, auth flows, backend logic, entity design, migrations | [BACKEND CLAUDE.md](BACKEND%20CLAUDE.md) |
| UI screens, component design, form behavior, routing, state management, API calls from frontend | [FRONTEND CLAUDE.md](FRONTEND%20CLAUDE.md) |
| Both (full-feature stories, Jira epics, end-to-end flows) | Read **both** |

Do not invent patterns, naming conventions, or structures that contradict these files. Stories and designs must align with what is already built and how the team builds.

---

## What the Architecture Guides Cover

### [BACKEND CLAUDE.md](BACKEND%20CLAUDE.md) — Spring Boot 3 + Java 21
- Tech stack: Spring Boot 3.2.4, Java 21, PostgreSQL, Flyway, MapStruct, JPA/Hibernate, JJWT, SpringDoc
- Layered architecture: `Controller → Service → Repository → Database`
- Multi-tenancy model: shared DB, isolated schemas per tenant; `TenantContext` routing via JWT claims
- Auth flows: two-step login, 3-step OTP registration, forgot password, token refresh, session limits
- JWT structure: claims (`sub`, `tenantSchema`, `tenantId`, `role`, `roleId`), access token (30 min), refresh token (24 hr HttpOnly cookie, single-use rotation)
- RBAC: `SUPER_ADMIN`, `ADMIN`, `USER` roles; `hasAuthority()` checks; module/menu permission matrix
- Database schemas: public schema (users, tenants, roles, RBAC tables) vs. tenant schema (all business data — no `tenant_id` columns needed)
- File uploads: S3 key format, CloudFront URL pattern, `FileUrlUtil`
- Coding standards: DTO pattern (never expose entities), MapStruct, pagination on all list endpoints, MDC logging, error handling via `@ControllerAdvice`
- Mandatory after every change: test cases, Swagger annotations, Postman collection update

### [FRONTEND CLAUDE.md](FRONTEND%20CLAUDE.md) — React 19 + TypeScript
- Tech stack: React 19, TypeScript (strict), Vite, Zustand, Axios, TanStack React Query, React Hook Form, Zod, Tailwind CSS 4
- Auth flow: 2-step login → access token in memory (Zustand), refresh token as HttpOnly cookie; session recovery on app mount via `App.tsx`
- API layer: Axios base URL `/cedge/api/v1`, `withCredentials: true`, 401 interceptor auto-refreshes token
- State: Zustand `authStore` — `accessToken` memory-only; `userId`, `username`, `email`, `tenantSchema`, `role` persisted to localStorage key `cedge-auth`
- Layout: fixed sidebar (220px, collapsible) + fixed header (60px) + scrollable main; sidebar nav driven by `GET /api/v1/menu`
- Icons: **Google Material Symbols Outlined** (CDN) for sidebar nav; Lucide React for general UI — never mix them
- Styling: Tailwind CSS only, no component library (no Shadcn, no MUI); brand colors `#2563eb` (blue) / `#FF9900` (orange); CSS custom properties for theming
- Forms: React Hook Form + Zod resolver on all forms; `MasterFormField` component for all master-data screens
- Image uploads: always via backend `POST /upload/image` → Cloudflare R2; never upload directly from frontend
- User management: role-guarded pages under `/login-management/users`; `SUPER_ADMIN` vs `ADMIN` behavior differs

---

## HTML Prototype Convention

Prototypes in this repo are reference designs — not production code. When creating a new prototype:

1. Always link `design-system.css` as the sole stylesheet
2. Use the fixed sidebar + header shell from `design-system.css` layout classes
3. Use **Google Material Symbols Outlined** (CDN) for all icons — same as the real frontend
4. Use `.card`, `.btn`, `.badge`, `.table-container`, `.status-*` classes from the design system
5. Do not hardcode colors or sizes — use CSS custom properties (`--bg-primary`, `--accent-blue`, etc.)
6. Field layouts must match the frontend conventions: bottom-border style inputs, `MasterFormField`-equivalent behavior

---

## User Story / Jira Ticket Convention

When writing user stories or Jira tickets for C-Edge ERP features:

- **Reference actual DB table names** as they exist or will exist in the tenant schema (check [BACKEND CLAUDE.md](BACKEND%20CLAUDE.md) for established tables)
- **Reference actual API endpoint paths** using the `/cedge/api/v1/` prefix and REST conventions from the backend guide
- **Specify role-based behavior** using the exact role names: `SUPER_ADMIN`, `ADMIN`, `USER`
- **Include acceptance criteria** for both the UI behavior (aligned with frontend conventions) and the backend behavior (aligned with backend conventions)
- **For multi-tenant features**: clarify whether data lives in the public schema or the tenant schema
- **For file/image features**: follow the two-phase upload pattern (upload → get URL → save entity)

---

## Features Currently in Development

Documented in [user-stories-grn-qc-approval.md](user-stories-grn-qc-approval.md) and prototyped in HTML:

| Ticket | Feature | Prototype | Key Entities |
|--------|---------|-----------|--------------|
| NE-13 | GRN Excess Quantity Approval Workstation | `grn-approval.html` | `grn_header` (status: `EXCESS_PENDING`), `ApprovalTrail`, tolerance % |
| NE-14A | Inline QC at GRN Receipt | `grn-qc.html` | `GrnLine` (acceptedQty, rejectedQty), inline QC status flags |
| NE-14B | QC Inspection Page + Debit Notes | `grn-qc.html` | `QcInspection`, `DebitNote`, `VendorReturn`, `InventoryMovement` |
| NE-PM-01 to NE-PM-08 | Product Management Module | `product-list.html`, `product-create.html`, `product-edit.html`, `product-view.html` | `products`, `product_sizes`, `product_bom`, `product_customer_names`, `product_descriptions`, `product_attachments`, `size_master` |
| NE-CST-01 to NE-CST-04 | Garment Costing Sheet | `costing-list.html`, `costing-create.html`, `costing-edit.html`, `costing-view.html` | `costing_header`, `costing_fabric_lines`, `costing_cm_lines`, `costing_packing_lines`, `costing_overhead_lines` |

---

## Repository File Index

```
/
├── BACKEND CLAUDE.md                   # Backend architecture — read before any backend story/design
├── FRONTEND CLAUDE.md                  # Frontend architecture — read before any UI story/design
├── design-system.css                   # Shared CSS design system for all HTML prototypes
├── user-stories-grn-qc-approval.md     # Feature specs: GRN approval, QC inspection, debit notes
├── purchase-order.html                 # PO master entry screen
├── po-receipts.html                    # GRN receipt entry screen
├── grn-approval.html                   # GRN excess quantity approval workstation
├── grn-qc.html                         # QC inspection workstation
├── purchase-order-list.html            # PO list view
├── masters-*.html (9 files)            # Master data screens: color, UOM, vendor, fabric, process, etc.
├── user-stories-product-management.md  # Feature specs: Product Management module (NE-PM-01 to NE-PM-08)
├── product-list.html                   # Product list page prototype
├── product-create.html                 # Product create page prototype
├── product-edit.html                   # Product edit page prototype
├── product-view.html                   # Product view (read-only) page prototype
├── costing-list.html                   # Garment costing list — all styles, PDF download, stats
├── costing-create.html                 # Create costing — product/sample selector, BOM auto-load, 5 sections, live calc
├── costing-edit.html                   # Edit costing — pre-filled from CVS-2026-F04 PDF data
└── costing-view.html                   # View costing (read-only) — PDF/print output, summary cards
```


