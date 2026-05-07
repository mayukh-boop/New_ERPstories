# Product Management — User Stories
**Module:** Style Management → Products
**Prepared:** 07 May 2026
**Reference Prototypes:**
- [product-list.html](product-list.html) — List page with configurable column headers
- [product-create.html](product-create.html) — Create product form
- [product-edit.html](product-edit.html) — Edit product form
- [product-view.html](product-view.html) — Read-only view with linked production orders

**Jira Epic:** NE-41
**Tech Stack:** Spring Boot 3.2.4 / Java 21 / PostgreSQL (tenant schema) / React 19 / TypeScript / Tailwind CSS 4

> **Important:** This document is the single source of truth for backend development. All DB tables, API endpoints, DTOs, field validations, and business rules are fully specified here. The HTML prototypes are the UI reference — read them alongside this document.

---

## Story Index

| Story ID | Title | Priority | Points |
|----------|-------|----------|--------|
| NE-PM-01 | Product List Page with Configurable Columns | High | 5 |
| NE-PM-02 | Create Product | High | 13 |
| NE-PM-03 | Edit Product | High | 8 |
| NE-PM-04 | View Product with Linked Production Orders | Medium | 5 |
| NE-PM-05 | Archive & Restore Product | Medium | 3 |
| NE-PM-06 | Bulk Import & Export | Low | 5 |
| NE-PM-07 | Role-Based Access Control | High | 3 |

---

## Database Schema (Tenant Schema)

All tables below live in the **tenant schema** (no `tenant_id` column needed — schema isolation handles multi-tenancy).

### Table: `products`

```sql
CREATE TABLE products (
    id                  BIGSERIAL PRIMARY KEY,
    product_code        VARCHAR(30)     NOT NULL UNIQUE,  -- auto-generated PROD-YYYY-NNNN
    product_name        VARCHAR(100),                     -- nullable; code used as fallback display
    color_id            BIGINT          REFERENCES color_master(id),
    color_name          VARCHAR(100),                     -- denormalised snapshot at save time
    color_hex           VARCHAR(7),                       -- denormalised snapshot at save time
    category_id         BIGINT          REFERENCES product_category(id),
    category_name       VARCHAR(100),                     -- denormalised snapshot
    brand_id            BIGINT          REFERENCES brand_master(id),
    brand_name          VARCHAR(100),                     -- denormalised snapshot
    uom_id              BIGINT          NOT NULL REFERENCES uom_master(id),
    uom_code            VARCHAR(20)     NOT NULL,          -- denormalised snapshot
    hsn_code            VARCHAR(20),
    gst_percent         NUMERIC(5,2)    CHECK (gst_percent >= 0 AND gst_percent <= 100),
    standard_cost       NUMERIC(12,2),
    mrp                 NUMERIC(12,2),
    moq                 INTEGER         CHECK (moq > 0),
    remarks             VARCHAR(500),
    product_image       VARCHAR(500),   -- stored value: {timestamp}!_!{filename}; CloudFront URL built at read time
    status              VARCHAR(20)     NOT NULL DEFAULT 'DRAFT'
                            CHECK (status IN ('DRAFT','ACTIVE','HOLD','ARCHIVED')),
    created_by          BIGINT          NOT NULL REFERENCES public.users(id),
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_by          BIGINT          REFERENCES public.users(id),
    updated_at          TIMESTAMPTZ
);
```

### Table: `product_customer_names`

```sql
CREATE TABLE product_customer_names (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    name        VARCHAR(100) NOT NULL,
    sort_order  INTEGER NOT NULL DEFAULT 0
);
```

### Table: `product_descriptions`

```sql
CREATE TABLE product_descriptions (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    description VARCHAR(200) NOT NULL,
    sort_order  INTEGER NOT NULL DEFAULT 0
);
```

### Table: `product_sizes`

```sql
CREATE TABLE product_sizes (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    size_id     BIGINT  REFERENCES size_master(id),
    size_code   VARCHAR(20) NOT NULL,  -- denormalised snapshot
    quantity    INTEGER,
    sort_order  INTEGER NOT NULL DEFAULT 0
);
```

### Table: `product_bom`

```sql
CREATE TABLE product_bom (
    id              BIGSERIAL PRIMARY KEY,
    product_id      BIGINT  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    material_type   VARCHAR(50) NOT NULL,   -- 'Fabric', 'Thread', 'Button', 'Label', 'Other'
    material_id     BIGINT,                 -- NULL when material_source = 'NEW'
    material_name   VARCHAR(200) NOT NULL,
    material_source VARCHAR(10)  NOT NULL DEFAULT 'EXISTING'
                        CHECK (material_source IN ('EXISTING','NEW')),
    consumption     NUMERIC(10,4) NOT NULL,
    uom_id          BIGINT  REFERENCES uom_master(id),
    uom_code        VARCHAR(20),            -- denormalised snapshot
    sort_order      INTEGER NOT NULL DEFAULT 0
);
```

### Table: `product_attachments`

```sql
CREATE TABLE product_attachments (
    id              BIGSERIAL PRIMARY KEY,
    product_id      BIGINT  NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    file_name       VARCHAR(300) NOT NULL,
    stored_value    VARCHAR(500) NOT NULL,  -- {timestamp}!_!{filename}; CloudFront URL at read time
    file_size_bytes BIGINT,
    uploaded_by     BIGINT  REFERENCES public.users(id),
    uploaded_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### Table: `product_category` (master)

```sql
CREATE TABLE product_category (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL UNIQUE,
    created_by  BIGINT REFERENCES public.users(id),
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### S3 Key Convention

```
Key:  {tenantCode}/STYLE_PRODUCT/{timestamp}!_!{sanitisedFileName}
DB:   stores only {timestamp}!_!{sanitisedFileName}
URL:  constructed at read time via FileUrlUtil using CloudFront base
```

---

## API Endpoints

All endpoints under base path `/cedge/api/v1/products`.

### Endpoint Summary

| Method | Path | Role | Description |
|--------|------|------|-------------|
| GET | `/products` | ADMIN, USER | Paginated list with filters; excludes ARCHIVED |
| GET | `/products/archived` | ADMIN | Archived products list |
| GET | `/products/{id}` | ADMIN, USER | Full product detail |
| POST | `/products` | ADMIN | Create product (with multi-colour expansion) |
| PUT | `/products/{id}` | ADMIN | Update product |
| PUT | `/products/{id}/status` | ADMIN | Status-only change |
| GET | `/products/export` | ADMIN, USER | Export filtered list as XLSX |
| POST | `/products/import` | ADMIN | Bulk import from XLSX/CSV |
| GET | `/products/import/template` | ADMIN | Download blank import template |
| GET | `/products/{id}/production-orders` | ADMIN, USER | List production orders linked to product |

---

### GET `/products` — Paginated List

**Query Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| page | int | 0 | Zero-based page index |
| size | int | 10 | Page size |
| search | String | — | Partial match on product_name OR product_code (case-insensitive) |
| status | String | — | DRAFT / ACTIVE / HOLD (ARCHIVED excluded from this endpoint always) |
| categoryId | Long | — | Filter by category |
| fromDate | LocalDate | — | created_at >= fromDate |
| toDate | LocalDate | — | created_at <= toDate |
| sortBy | String | created_at | Field to sort by |
| sortDir | String | desc | asc / desc |

**Response:** `Page<ProductListItemDto>`

```java
public record ProductListItemDto(
    Long id,
    String productCode,
    String productName,
    String colorName,
    String colorHex,
    String categoryName,
    String brandName,
    String uomCode,
    String mrp,
    String productImageUrl,   // CloudFront URL, null if no image
    List<ColorChipDto> colors,
    String sizes,             // comma-joined size codes
    String status,
    String createdByName,
    LocalDateTime createdAt
) {}

public record ColorChipDto(String name, String hex) {}
```

**Service Rules:**
- Always append `AND status != 'ARCHIVED'` to the query regardless of other filters.
- `createdByName` is resolved via join to `public.users.full_name`.
- `productImageUrl` = `FileUrlUtil.buildUrl(product.getProductImage())` — null if field is null.

---

### GET `/products/{id}` — Full Detail

**Response:** `ProductDetailDto`

```java
public record ProductDetailDto(
    Long id,
    String productCode,
    String productName,
    Long colorId,
    String colorName,
    String colorHex,
    Long categoryId,
    String categoryName,
    Long brandId,
    String brandName,
    Long uomId,
    String uomCode,
    String hsnCode,
    BigDecimal gstPercent,
    BigDecimal standardCost,
    BigDecimal mrp,
    Integer moq,
    String remarks,
    String productImageUrl,
    String status,
    List<CustomerNameDto> customerNames,
    List<DescriptionDto> descriptions,
    List<ProductSizeDto> sizes,
    List<BomLineDto> bomLines,
    List<AttachmentDto> attachments,
    String createdByName,
    LocalDateTime createdAt,
    String updatedByName,
    LocalDateTime updatedAt
) {}

public record CustomerNameDto(Long id, String name, int sortOrder) {}
public record DescriptionDto(Long id, String description, int sortOrder) {}
public record ProductSizeDto(Long id, Long sizeId, String sizeCode, Integer quantity, int sortOrder) {}
public record BomLineDto(
    Long id, String materialType, Long materialId, String materialName,
    String materialSource, BigDecimal consumption, Long uomId, String uomCode, int sortOrder
) {}
public record AttachmentDto(Long id, String fileName, String fileUrl, Long fileSizeBytes, LocalDateTime uploadedAt) {}
```

---

### POST `/products` — Create (with Multi-Colour Expansion)

**Request Body:** `CreateProductRequest`

```java
public record CreateProductRequest(
    String productName,                         // nullable; auto-code generated if blank
    @NotEmpty List<Long> colorIds,             // min 1; one product created per colour
    List<String> customerNames,
    List<String> descriptions,
    Long categoryId,
    Long brandId,
    @NotNull Long uomId,
    String hsnCode,
    @DecimalMin("0") @DecimalMax("100") BigDecimal gstPercent,
    BigDecimal standardCost,
    BigDecimal mrp,
    @Min(1) Integer moq,
    List<ProductSizeRequest> sizes,
    List<BomLineRequest> bomLines,
    String remarks,
    String productImage,                        // stored value from upload endpoint
    List<String> attachmentStoredValues,        // stored values from upload endpoint
    String status                               // 'DRAFT' | 'ACTIVE'
) {}

public record ProductSizeRequest(Long sizeId, String sizeCode, Integer quantity) {}
public record BomLineRequest(
    String materialType, Long materialId, String materialName,
    String materialSource, BigDecimal consumption, Long uomId
) {}
```

**Service Logic:**
1. For each `colorId` in `colorIds`:
   - Resolve `color_name` and `color_hex` from `color_master`.
   - Generate `product_code` using sequence: `PROD-{YEAR}-{LPAD(seq,4,'0')}`. Sequence is tenant-scoped (stored as a counter in a utility table or via DB sequence named `product_code_seq`).
   - Insert one `products` row.
   - Insert all child rows (`product_customer_names`, `product_descriptions`, `product_sizes`, `product_bom`, `product_attachments`) with `product_id` set to the new product's id.
2. Return `List<ProductCreatedDto>` with one entry per created product.

**Response:** `201 Created`

```java
public record ProductCreatedDto(Long id, String productCode, String colorName) {}
```

**Validation errors:** `400 Bad Request` with field-level messages in `Map<String, String>`.

---

### PUT `/products/{id}` — Update

**Request Body:** Same shape as `CreateProductRequest` but:
- `colorIds` is a single `Long colorId` (no multi-colour expansion on edit).
- `attachmentStoredValues` contains stored values for **new** attachments only.
- `keepAttachmentIds` contains `List<Long>` of existing attachment IDs to retain. Any attachment not in this list is deleted (file reference removed from DB; actual file deletion is async/background).

**Service Logic:**
1. Load product; if status = ARCHIVED → return `422 Unprocessable Entity` ("Cannot edit an archived product. Restore it first.").
2. Update `products` row.
3. Delete and re-insert all child rows: `product_customer_names`, `product_descriptions`, `product_sizes`, `product_bom`.
4. Attachments: delete rows where `id NOT IN keepAttachmentIds`; insert new rows for `attachmentStoredValues`.
5. Set `updated_by` = userId from JWT, `updated_at` = NOW().

**Response:** `200 OK` with `ProductDetailDto`.

---

### PUT `/products/{id}/status` — Status Change

**Request Body:**

```java
public record StatusChangeRequest(@NotBlank String status) {}
```

**Valid Transitions (server-enforced):**

| From | To | Allowed |
|------|----|---------|
| DRAFT | ACTIVE | ✅ |
| DRAFT | ARCHIVED | ✅ |
| ACTIVE | HOLD | ✅ |
| ACTIVE | ARCHIVED | ✅ |
| HOLD | ACTIVE | ✅ |
| HOLD | ARCHIVED | ✅ |
| ARCHIVED | DRAFT | ✅ (restore) |
| ARCHIVED | ACTIVE/HOLD | ❌ 422 |

**Response:** `200 OK` with `{ "id": ..., "status": "...", "productCode": "..." }`.

---

### GET `/products/archived` — Archived List

**Query Parameters:** `page`, `size`, `search` (name or code).

**Response:** `Page<ArchivedProductDto>`

```java
public record ArchivedProductDto(
    Long id, String productCode, String productName,
    String categoryName, LocalDateTime archivedAt, String archivedByName
) {}
```

---

### GET `/products/{id}/production-orders` — Linked Production Orders

Returns all production orders where `production_orders.product_id = {id}`.

**Response:** `List<LinkedProductionOrderDto>`

```java
public record LinkedProductionOrderDto(
    Long id,
    String poNumber,
    String styleDescription,
    String buyerName,
    Integer plannedQty,
    Integer producedQty,
    LocalDate plannedStartDate,
    LocalDate plannedEndDate,
    String status
) {}
```

**Note:** This endpoint reads from the `production_orders` table in the tenant schema. It is read-only for the Products module — production orders are managed by the Production module.

---

### POST `/products/import` — Bulk Import

**Request:** `multipart/form-data` with field `file` (`.xlsx` or `.csv`, max 5 MB).

**Expected columns in template (in order):**
`product_name`, `color_name`, `category_name`, `brand_name`, `uom_code`, `hsn_code`, `gst_percent`, `standard_cost`, `mrp`, `moq`, `status`

**Response:**

```java
public record ImportResultDto(
    int totalRows,
    int successCount,
    int errorCount,
    List<RowErrorDto> errors
) {}

public record RowErrorDto(int rowNumber, String field, String message) {}
```

**Row-level validation:**
- `uom_code` must match an existing UOM in `uom_master`; error: "UOM '{code}' not found".
- `gst_percent` must be 0–100; error: "GST must be between 0 and 100".
- `moq` must be > 0 if provided.
- `status` must be DRAFT or ACTIVE; default DRAFT if blank.
- `color_name` resolved against `color_master.name` (case-insensitive); created as new if not found.
- Partial success: valid rows are always inserted even if other rows fail.

---

## Flyway Migration

File: `V{next}__add_product_tables.sql`

```sql
-- product_category master
CREATE TABLE product_category (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    created_by BIGINT REFERENCES public.users(id),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- products main table
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_name VARCHAR(100),
    color_id BIGINT REFERENCES color_master(id),
    color_name VARCHAR(100),
    color_hex VARCHAR(7),
    category_id BIGINT REFERENCES product_category(id),
    category_name VARCHAR(100),
    brand_id BIGINT REFERENCES brand_master(id),
    brand_name VARCHAR(100),
    uom_id BIGINT NOT NULL REFERENCES uom_master(id),
    uom_code VARCHAR(20) NOT NULL,
    hsn_code VARCHAR(20),
    gst_percent NUMERIC(5,2) CHECK (gst_percent >= 0 AND gst_percent <= 100),
    standard_cost NUMERIC(12,2),
    mrp NUMERIC(12,2),
    moq INTEGER CHECK (moq > 0),
    remarks VARCHAR(500),
    product_image VARCHAR(500),
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT'
        CHECK (status IN ('DRAFT','ACTIVE','HOLD','ARCHIVED')),
    created_by BIGINT NOT NULL REFERENCES public.users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by BIGINT REFERENCES public.users(id),
    updated_at TIMESTAMPTZ
);

CREATE TABLE product_customer_names (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE product_descriptions (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    description VARCHAR(200) NOT NULL,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE product_sizes (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    size_id BIGINT REFERENCES size_master(id),
    size_code VARCHAR(20) NOT NULL,
    quantity INTEGER,
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE product_bom (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    material_type VARCHAR(50) NOT NULL,
    material_id BIGINT,
    material_name VARCHAR(200) NOT NULL,
    material_source VARCHAR(10) NOT NULL DEFAULT 'EXISTING'
        CHECK (material_source IN ('EXISTING','NEW')),
    consumption NUMERIC(10,4) NOT NULL,
    uom_id BIGINT REFERENCES uom_master(id),
    uom_code VARCHAR(20),
    sort_order INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE product_attachments (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    file_name VARCHAR(300) NOT NULL,
    stored_value VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT,
    uploaded_by BIGINT REFERENCES public.users(id),
    uploaded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_color ON products(color_id);
CREATE INDEX idx_products_created_at ON products(created_at DESC);
CREATE INDEX idx_product_bom_product ON product_bom(product_id);
CREATE INDEX idx_product_sizes_product ON product_sizes(product_id);
```

---

## RBAC — `public.module_wise_links` Seed

```sql
INSERT INTO public.module_wise_links (module_id, link_code, link_name, link_url, icon, sort_order)
VALUES (
    (SELECT id FROM public.modules WHERE module_code = 'STYLE_MANAGEMENT'),
    'STYLE_PRODUCT',
    'Products',
    '/style/products',
    'inventory_2',
    2
);
```

`TenantDefaultDataService` seeds `role_module_menu`:

| Role | canList | canView | canCreate | canEdit | canDelete |
|------|---------|---------|-----------|---------|-----------|
| ADMIN | ✅ | ✅ | ✅ | ✅ | ✅ |
| USER | ✅ | ✅ | ❌ | ❌ | ❌ |

---

---

## NE-PM-01 — Product List Page with Configurable Columns

### User Story
As a **merchandiser or planner**, I want to see all products in a filterable, sortable list where I can choose which columns are visible and in what order, so I can work efficiently without scrolling past columns I don't need.

### Prototype
[product-list.html](product-list.html) — see Columns button in filter bar; drag-to-reorder panel slides in from right.

### Acceptance Criteria

**AC-01 — Page Load**
- Accessible from sidebar: Style Management → Products.
- Page title "Product List", sub-heading "Manage all products — view, edit and create production orders".
- "New Product" button (top-right) navigates to Create page. Hidden for USER role.
- "Import" and "Export" buttons in page header. Import hidden for USER role.
- "View Archived Products" text link below header, right-aligned.

**AC-02 — Status Summary Tiles**
- Three count tiles above table: **Draft**, **Active**, **Hold**.
- Counts reflect currently filtered result set (not total).
- Updated on every filter change.

**AC-03 — Filter Bar**
- Search: partial match on `product_name` OR `product_code`, case-insensitive.
- Status: All / Active / Draft / Hold.
- Category: populated from distinct categories in tenant product data.
- Days Range: Last 7 / 14 / 30 (default) / 90 / Custom. Custom reveals From/To date pickers.
- **Columns button** (with adjustments icon) opens the column configurator panel.
- Clear button resets all filters to defaults.
- Filters applied immediately on change.

**AC-04 — Column Configurator Panel**
- Slides in from the right side of the screen when "Columns" button is clicked.
- Title: "Configure Columns", sub-heading "Show/hide and drag to reorder".
- All configurable columns listed with a toggle (on/off) and a drag handle.
- Fixed columns (#, Image, Product Name, Action) are labelled "Fixed" and cannot be toggled or dragged.
- Dragging a row in the panel reorders the table columns in real time.
- "Reset to Default" button restores the default column set and order.
- "Done" button closes the panel.
- Column configuration is persisted in `localStorage` under the key `cedge-product-list-cols` so it survives page refreshes.
- Default visible columns: #, Image, Product Name, Code, Category, Colors, Creation Date, Status, Created By, Action.
- Additional optional columns: Brand, UOM, Sizes, MRP.

**AC-05 — Table Behaviour**
- `#` column shows row number within current page.
- Image thumbnail: 48×48px, clicking opens fullscreen overlay; placeholder icon if no image.
- Product Name: clickable, opens quick-view modal.
- Code: monospace font.
- Colors: swatch chips (hex-coloured squares) with colour name on hover.
- Status: pill — Draft (grey) / Active (green) / Hold (yellow).
- Action column: **Edit**, **View**, **Archive** (delete icon) buttons.

**AC-06 — Quick-View Modal**
- Opens on product name click.
- Shows: Product Name, Code, Status pill, Category, UOM, Brand, MRP, MOQ, Colors, Sizes, Created By, Created Date, product image.
- Footer: **Edit** → Edit page, **Full View** → View page.
- Closes on × click or outside click.

**AC-07 — Pagination**
- 10 records per page default.
- Count label: "Showing X–Y of Z products".
- Previous/next + page number buttons.
- Archived products excluded from all counts.

**AC-08 — Image Fullscreen**
- Clicking thumbnail or modal image opens a dark fullscreen overlay with the full image and × close button.

### API Used
- `GET /cedge/api/v1/products` — list with all filter params
- `PUT /cedge/api/v1/products/{id}/status` (status: ARCHIVED) — archive action

### Definition of Done
- [ ] `GET /products` endpoint with all query params, pagination, ARCHIVED exclusion.
- [ ] `PUT /products/{id}/status` for ARCHIVED transition.
- [ ] Table with dynamic columns, status tiles, filter bar, paginator all functional.
- [ ] Column configurator panel: toggle, drag-to-reorder, reset, persisted in localStorage.
- [ ] Quick-view modal, image fullscreen.
- [ ] "View Archived" link opens archive overlay.
- [ ] Unit tests: filter by status, search, category, date range; page boundary; no results.
- [ ] Swagger annotations on list endpoint.
- [ ] Postman request for list (with each filter variant) added.

---

## NE-PM-02 — Create Product

### User Story
As a **merchandiser**, I want to create a product with all its specifications — colour, sizes, BOM, pricing, and attachments — so the product is fully defined and available for production planning immediately.

### Prototype
[product-create.html](product-create.html)

### Acceptance Criteria

**AC-01 — Page Access**
- Accessible from "New Product" button on list page.
- Cancel button returns to list without saving.

**AC-02 — Section 1: Product Information**

| Field | Rules |
|-------|-------|
| Product Name | Varchar(100). Optional — if blank, auto-code used as display name. Shows character counter. |
| Color (mandatory) | Multi-select from `GET /cedge/api/v1/masters/color`. Shows swatch + name per option. Selected items appear as chips. "+ Create New Color" inline panel (Name + Hex picker). Banner: "Selecting multiple colours will create separate products per colour." |
| Customer Product Name | Multi-entry rows. First row auto-fills from Product Name input. Each row: varchar(100), delete button. "+ Add another customer name" adds a row. |
| Product Description | Multi-entry rows. Varchar(200) per row. "+ Add description line". Not mandatory. |
| Product Category | Dropdown from `product_category`. "+" opens inline create panel (name only). |
| Brand | Dropdown from `brand_master`. "+" opens inline create panel. |
| Product Image | Drag-drop / click-to-upload. "From Sample" link opens sample selection modal (one-time copy). Upload immediately via `POST /upload/image` → stored value → preview shown. Removable. PNG/JPG/JPEG only. |

**AC-03 — Section 2: Size & Quantity**

| Field | Rules |
|-------|-------|
| Sizes | Multi-select from `size_master`. "+" for inline create (code + name). Selected sizes appear as a table: Size chip | Qty input | Delete. Removing from dropdown removes table row. |
| MOQ | Single integer input, applies to whole product. Optional. |

**AC-04 — Section 3: Pricing & Compliance**

| Field | Type | Rule |
|-------|------|------|
| UOM | Dropdown from `uom_master` + inline create | Mandatory |
| HSN Code | Varchar(20) | Optional |
| GST (%) | Decimal 0–100 | Optional; client clamps to 100; server rejects > 100 |
| Standard Cost | Decimal, ₹ prefix | Optional |
| MRP | Decimal, ₹ prefix | Optional |

**AC-05 — Section 4: BOM**
- Radio: **Use Existing Materials** (default) | **Add New Materials**.
- Existing mode: Material Type dropdown → Material Name dropdown (filtered by type) → Consumption (decimal) → UOM dropdown.
- New mode: Material Type dropdown → Material Name free text (varchar 200) → Consumption → UOM.
- "+ Import from Sample" opens sample modal; imports BOM rows as editable starting point.
- "+ Add Row" adds blank row. Row delete button on each row.
- BOM not mandatory.

**AC-06 — Sections 5 & 6**
- Remarks: Textarea, varchar(500), optional.
- Attachments: Multi-file drop zone. Types: PNG, JPG, JPEG, PDF, DOC, DOCX, XLS, XLSX, CSV, TXT, ZIP, PPT, PPTX. Each file: icon + name + size + Download + Delete.

**AC-07 — Action Bar**
- **Cancel** — back to list.
- **Save as Draft** — status = DRAFT.
- **Save Product** — status = ACTIVE.

**AC-08 — Multi-Colour Expansion**
- N colours selected → on save, confirmation modal: "You have selected {N} colours ({list}). Saving will create {N} separate products — one per colour."
- On confirm: `POST /products` with all `colorIds`; backend creates N records.
- Success toast: "2 products created successfully — PROD-2026-0001, PROD-2026-0002".

**AC-09 — Validation**
- UOM mandatory — "UOM is required."
- At least one colour — "At least one colour is required."
- GST > 100 clamped client-side; `400` from server with "GST must be between 0 and 100."
- All errors shown inline beside field.

### API Used
- `POST /cedge/api/v1/products`
- `GET /cedge/api/v1/masters/color`, `/masters/uom`, `/masters/size`, `/masters/brand`
- `POST /cedge/api/v1/masters/color` (inline create), `/masters/uom`, `/masters/brand`, `/masters/size`
- `GET /cedge/api/v1/product-categories`, `POST /cedge/api/v1/product-categories`
- `POST /cedge/api/v1/upload/image`

### Definition of Done
- [ ] `POST /products` with multi-colour expansion, auto-code generation, all child table inserts.
- [ ] Image upload wired (POST /upload/image → stored value → saved to products.product_image).
- [ ] Attachment upload wired.
- [ ] Inline create for Color, UOM, Brand, Size, Category immediately reflected in dropdowns.
- [ ] BOM "Import from Sample" populates rows.
- [ ] Multi-colour confirmation modal on N > 1 colours.
- [ ] Validation errors inline; success toast with all created codes.
- [ ] Unit tests: single colour, 3 colours → 3 records, blank name auto-generates code, missing UOM → 400, GST 150 → 400.
- [ ] Integration test: full create → verify all child tables.
- [ ] Swagger + Postman.

---

## NE-PM-03 — Edit Product

### User Story
As a **merchandiser or production planner**, I want to edit any field of an existing product — including status and BOM — so I can keep specifications current and control production availability.

### Prototype
[product-edit.html](product-edit.html)

### Acceptance Criteria

**AC-01 — Page Load**
- Loads pre-populated from `GET /products/{id}`.
- Header: "Edit: {Product Name}", product code as sub-heading, "Edit Mode" badge.
- Meta strip: Created by, Created on, Last modified by, Last modified date.
- Navigation buttons: **View** (→ View page) and **Back to List**.

**AC-02 — Editable Fields**
- All fields from Create page editable: Name, Colour (single-select only — no multi-colour on edit), Customer Names, Descriptions, Category, Brand, Sizes + quantities, UOM, HSN, GST, Standard Cost, MRP, MOQ, BOM rows, Remarks, Image, Attachments.
- Colour is **single-select** on Edit. Changing colour updates `color_id`, `color_name`, `color_hex` on the single product record.
- Existing attachments shown with Download + Delete. New files added via drop zone.
- BOM rows editable inline; add/delete rows.

**AC-03 — Status Dropdown**
- Visible **only** on Edit page, not on Create.
- Options: Draft, Active, Hold, Archive.
- Hidden for USER-role users (enforced server-side + frontend).
- Selecting Archive + saving → redirects to Product List with toast.

**AC-04 — Action Bar**
- **Cancel** — back to list without saving.
- **Update Product** — saves all changes; success toast; redirects to View page.
- **Create Production Order** (blue `#2563eb`) — navigates to production order create with `productId` pre-set. Always visible.

**AC-05 — BOM Edit**
- Full BOM section available (Existing / New radio, all fields, add/delete rows).
- "Import from Sample" also available.

**AC-06 — Image Replacement**
- Current image shown if present; click to fullscreen.
- Drag-drop new image replaces it. "✕ Remove" clears it.

**AC-07 — Validation**
- Same as Create (UOM mandatory, Colour required, GST ≤ 100).
- Archived product accessed via URL → `422` from server; frontend redirects to list with "Product is archived — restore it first."

**AC-08 — Unsaved Changes Warning**
- Browser confirmation on Cancel or back-navigation with unsaved changes.

### API Used
- `GET /cedge/api/v1/products/{id}`
- `PUT /cedge/api/v1/products/{id}`
- `PUT /cedge/api/v1/products/{id}/status`
- `POST /cedge/api/v1/upload/image`

### Definition of Done
- [ ] `GET /products/{id}` returns full detail including all child lists with download URLs.
- [ ] `PUT /products/{id}` with full child-table replace, status transition validation, audit fields.
- [ ] Edit page loads pre-populated; all fields editable; meta strip correct.
- [ ] Status dropdown visible on edit, hidden on create, hidden for USER role.
- [ ] Create Production Order button navigates with `productId`.
- [ ] Unsaved-changes confirmation works.
- [ ] Unit tests: happy path update, ARCHIVED product → 422, invalid status transition → 422, missing UOM → 400.
- [ ] Swagger + Postman.

---

## NE-PM-04 — View Product with Linked Production Orders

### User Story
As a **production planner or quality reviewer**, I want to view the complete product specification on a read-only page — including all linked production orders with their status and progress — so I can review all details and raise a new production order directly.

### Prototype
[product-view.html](product-view.html) — see Section 7 "Linked Production Orders" near the bottom, above the Action Bar.

### Acceptance Criteria

**AC-01 — Page Layout**
- Accessible from Edit button on list, Full View in quick-view modal, View button on Edit page.
- Page header: Product Name, Product Code (sub-heading), Status pill, "View Only" badge.
- Meta strip: Created by, Created on, Last modified by, Last modified.

**AC-02 — Sections 1–6 (Read-Only)**
All fields displayed as labelled read-only values:
1. **Product Information** — Name, Code, Colors (swatch chips), Customer Names, Descriptions, Category, Brand, Image (fullscreen on click).
2. **Size & Quantity** — table with Size | Qty | Std Cost | MRP | MOQ. Total quantity shown.
3. **Pricing & Compliance** — UOM, HSN, GST in 3-col grid; Standard Cost and MRP as price cards.
4. **BOM** — table: Material Type (colour-coded badge) | Material Name | Consumption | UOM.
5. **Remarks** — pre-formatted text.
6. **Attachments** — file icon, name, size, Download button, View button.

**AC-07 — Section 7: Linked Production Orders**
- Displayed as a table below Attachments.
- Section header shows count ("3 orders") and a "New Production Order" button (blue `#2563eb`).
- Table columns: # | PO Number (clickable, links to production order) | Style / Description | Buyer | Planned Qty | Produced (with % and mini progress bar) | Start Date | End Date | Status.
- Status pills: Planning (blue), Active (green), Completed (teal), Hold (yellow), Cancelled (red), Draft (grey).
- If no production orders linked: shows "No production orders linked to this product yet. Create one now →".
- Data comes from `GET /cedge/api/v1/products/{id}/production-orders`.

**AC-08 — Action Bar (Top and Bottom)**
Both page header area and bottom action bar show:
- **Print** → `window.print()` (sidebar, topbar, action bar hidden via `@media print`).
- **Edit** → Edit page.
- **Create Production Order** → production order create with `?productId={id}`.
- **Back to List** → Product List.

### API Used
- `GET /cedge/api/v1/products/{id}` — full product detail
- `GET /cedge/api/v1/products/{id}/production-orders` — linked POs

### Definition of Done
- [ ] View page renders all 7 sections from `GET /products/{id}` + `GET /products/{id}/production-orders`.
- [ ] Linked production orders table: PO number clickable, progress bar, status pills.
- [ ] Empty state shown when no production orders linked.
- [ ] "New Production Order" button navigates to create PO with productId.
- [ ] Pricing section shows Standard Cost and MRP as cards.
- [ ] BOM table with colour-coded material type badges.
- [ ] Attachments with Download + View buttons using CloudFront URLs.
- [ ] Image fullscreen on click.
- [ ] Print CSS: hides sidebar, topbar, action bar.
- [ ] "View Only" badge visible; Edit button hidden for USER role.
- [ ] Tested with: product with all fields; product with no image/BOM/attachments/POs (all empty states shown).
- [ ] Swagger annotation on `/products/{id}/production-orders`.
- [ ] Postman request added.

---

## NE-PM-05 — Archive & Restore Product

### User Story
As an **admin**, I want to archive products that are no longer in active use so they are hidden from main lists without permanent deletion, and restore any archived product back to Draft when needed.

### Prototype
[product-list.html](product-list.html) — "View Archived Products" link and archive modal; delete icon in each row.

### Acceptance Criteria

**AC-01 — Archive from List**
- Delete icon triggers confirmation: "Archive product '{name}'? This will move it to archive and it will no longer appear in active lists."
- Confirming calls `PUT /products/{id}/status` with `{ status: "ARCHIVED" }`.
- Row removed from list immediately. Success toast: "'{name}' archived".

**AC-02 — Archive via Edit Page**
- Status dropdown on Edit → select "Archive" → click Update Product.
- After save, redirected to list with toast.

**AC-03 — Archived Products Hidden**
- `GET /products` always excludes ARCHIVED (server-enforced `AND status != 'ARCHIVED'`).
- Status filter on list does NOT include "Archived".
- Status summary counts (Draft / Active / Hold) exclude ARCHIVED records.

**AC-04 — View Archived Modal**
- "View Archived Products" link opens overlay modal.
- Table: Product Name | Code | Category | Archived On | Restore button.
- Empty state if no archived products.
- Data from `GET /products/archived`.

**AC-05 — Restore**
- **Restore** button (green-tinted) calls `PUT /products/{id}/status` with `{ status: "DRAFT" }`.
- Row removed from modal immediately. Toast: "'{name}' restored to active products".
- Restored product appears in main list on next load.

**AC-06 — Production Order Guard**
- An ARCHIVED product cannot be selected when creating a Production Order.
- Production Order API validates `productId` belongs to a non-ARCHIVED product → `422` if not.

### API Used
- `PUT /cedge/api/v1/products/{id}/status`
- `GET /cedge/api/v1/products/archived`

### Definition of Done
- [ ] `PUT /products/{id}/status` handles ARCHIVED from DRAFT/ACTIVE/HOLD; DRAFT restore from ARCHIVED.
- [ ] `GET /products/archived` with pagination and search.
- [ ] Main list endpoint excludes ARCHIVED in all circumstances.
- [ ] Archive confirmation on delete icon click.
- [ ] Archive modal with table and Restore button.
- [ ] Toast on archive and restore.
- [ ] Production Order API rejects archived productId with 422.
- [ ] Unit tests: archive from each valid status, restore, list excludes archived.

---

## NE-PM-06 — Bulk Import & Export

### User Story
As an **admin or merchandiser**, I want to import multiple product records from a spreadsheet and export the current filtered list, so I can onboard products in bulk and share data with other teams.

### Prototype
[product-list.html](product-list.html) — Import button (opens modal), Export button in page header.

### Acceptance Criteria

**AC-01 — Import Modal**
- Clicking "Import" opens modal: drop zone for `.xlsx`/`.xls`/`.csv` (single file, max 5 MB).
- File selected → file name shown → Import button enabled.
- **Download Template** downloads `GET /products/import/template`.
- **Import** submits file to `POST /products/import`; toast on result.

**AC-02 — Import Processing**
- Server validates per-row: UOM exists, GST 0–100, MOQ > 0 if provided, status valid.
- Valid rows inserted as new products with status DRAFT (or as specified in file).
- Invalid rows skipped; response includes `RowErrorDto` list with row number, field, message.
- Partial success: valid rows always saved.

**AC-03 — Export**
- "Export" triggers `GET /products/export` with current filter params.
- Response is `.xlsx` download.
- Columns exported: Code, Name, Colour, Category, Brand, Sizes, UOM, GST%, Std Cost, MRP, MOQ, Status, Created By, Created Date.

### API Used
- `POST /cedge/api/v1/products/import`
- `GET /cedge/api/v1/products/import/template`
- `GET /cedge/api/v1/products/export`

### Definition of Done
- [ ] `POST /products/import` — parses file, row-level validation, batch insert, returns ImportResultDto.
- [ ] `GET /products/import/template` — returns XLSX with headers + one example row.
- [ ] `GET /products/export` — respects filter params, returns XLSX.
- [ ] Import modal: file upload UI, template download, result toast with error count if any.
- [ ] Export button downloads file.
- [ ] Unit tests: valid rows imported, invalid rows skipped with correct messages, missing UOM → row error.

---

## NE-PM-07 — Role-Based Access Control

### User Story
As an **administrator**, I want separate Create, Edit, and View permissions for products so that read-only users can browse without risk of modification.

### Acceptance Criteria

**AC-01 — Permission Matrix**

| Action | ADMIN | USER |
|--------|-------|------|
| View list | ✅ | ✅ |
| View detail | ✅ | ✅ |
| Export list | ✅ | ✅ |
| Create | ✅ | ❌ |
| Edit fields | ✅ | ❌ |
| Change status | ✅ | ❌ |
| Archive / restore | ✅ | ❌ |
| Import | ✅ | ❌ |

**AC-02 — UI Enforcement**
- "New Product", "Edit", "Delete/Archive", "Import" buttons hidden for USER role.
- Status dropdown on Edit hidden for USER role.
- Direct URL navigation to create/edit by USER → redirect to list with "Access denied" toast.

**AC-03 — API Enforcement**
- `POST /products`, `PUT /products/{id}`, `PUT /products/{id}/status`, `POST /products/import` → HTTP 403 for USER token.
- All GET endpoints accessible to both roles.

**AC-04 — Sidebar & Module Seed**
- Products link visible in sidebar for both roles (canList = true for both).
- `public.module_wise_links` row: link_code = `STYLE_PRODUCT`, link_url = `/style/products`.
- `TenantDefaultDataService`: ADMIN → all flags true; USER → canList + canView only.

### Definition of Done
- [ ] `module_wise_links` row seeded; `TenantDefaultDataService` seeds RBAC rows.
- [ ] All write endpoints return 403 for USER token.
- [ ] Frontend hides write-action buttons/dropdowns for USER role.
- [ ] Direct URL to create/edit as USER → list redirect.
- [ ] Integration test: USER token + POST /products → 403.
- [ ] Tested in browser with ADMIN and USER accounts.

---

## Cross-Cutting Rules (All Stories)

### Audit Fields
Every create and update records:
- `created_by` (userId from JWT), `created_at` (server-side UTC)
- `updated_by`, `updated_at` on every edit

### Toast Notifications
- **Success** — green, checkmark icon, 3.5s auto-dismiss.
- **Error** — red, error icon, 3.5s auto-dismiss.
- **Info** — brand colour, info icon. Used for "processing" states.

### Production Order Integration
- Edit + View pages both show "Create Production Order" → `/production-orders/create?productId={id}`.
- View page also shows all linked production orders via `GET /products/{id}/production-orders`.
- No API changes needed on the Products side — the Production module owns production order data.

### File Upload Standard
- Image upload: `POST /cedge/api/v1/upload/image` → returns stored value.
- Attachment upload: `POST /cedge/api/v1/upload/file` → returns stored value.
- S3 key: `{tenantCode}/STYLE_PRODUCT/{timestamp}!_!{sanitisedFileName}`
- DB stores only the stored value (not the full URL).
- `FileUrlUtil.buildUrl(storedValue)` constructs the CloudFront URL at read time.
- Never store or send full CloudFront URLs in the DB.

### Status Lifecycle

```
DRAFT ──► ACTIVE ──► HOLD
  ▲           │        │
  │           ▼        ▼
  └────── ARCHIVED ◄───┘
  (restore always sets back to DRAFT)
```

Invalid transitions rejected with `422 Unprocessable Entity` and message: "Invalid status transition from {current} to {requested}."
