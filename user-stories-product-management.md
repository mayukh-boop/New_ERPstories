# User Stories: Product Management Module
*Prepared: 06 May 2026 | Reference prototypes: product-list.html, product-create.html, product-edit.html, product-view.html*
*Architecture basis: BACKEND CLAUDE.md (Spring Boot 3 / Java 21 / PostgreSQL) · FRONTEND CLAUDE.md (React 19 / TypeScript / TanStack Query)*

---

## Module Overview

**Module name:** Style Management  
**Nav link:** Products  
**Route prefix (frontend):** `/style/products`  
**API prefix:** `/cedge/api/v1/products`  
**Menu code (file uploads):** `STYLE_PRODUCT`

The Product Management module allows users to define sellable garment products with full specification: colours, sizes, Bill of Materials (BOM), costing, and compliance data. Each saved product can become the basis of a Production Order.

---

## Jira Epic: NE-PM — Product Management

| Ticket | Page / Feature | Priority |
|--------|---------------|----------|
| NE-PM-01 | DB Schema & Flyway Migrations | P0 |
| NE-PM-02 | Product List Page (API + UI) | P0 |
| NE-PM-03 | Create Product (API + UI) | P0 |
| NE-PM-04 | Edit Product (API + UI) | P0 |
| NE-PM-05 | View Product (API + UI) | P0 |
| NE-PM-06 | Archive / Restore Product | P1 |
| NE-PM-07 | Bulk Import via Excel/CSV | P2 |
| NE-PM-08 | RBAC — Page-level Permission Wiring | P0 |

---

---

## NE-PM-01 — DB Schema & Flyway Migrations

### Summary
Create all tenant-schema tables required by the Product Management module via a Flyway migration script.

### Migration File
`db/migration/tenant/V9__create_product_tables.sql`

> **Rule:** All tables below live in the **tenant schema**. No `@Table(schema = "public")`. No `tenant_id` column. Schema routing is handled by `TenantConnectionProvider` via the JWT `tenantSchema` claim.

---

### Table: `products`

```sql
CREATE TABLE products (
    id                BIGSERIAL       PRIMARY KEY,
    product_code      VARCHAR(30)     NOT NULL UNIQUE,     -- auto-generated if product_name blank
    product_name      VARCHAR(100),                        -- nullable; auto-generated code used as display if null
    color_id          BIGINT,                              -- FK → color_master.id (nullable; snapshot at save time)
    color_name        VARCHAR(60),                         -- denormalised snapshot for list display
    color_hex         VARCHAR(10),                         -- denormalised hex for swatch display
    uom_id            BIGINT          NOT NULL,            -- FK → uoms_master.id
    uom_code          VARCHAR(20)     NOT NULL,            -- denormalised
    category          VARCHAR(60),
    brand_id          BIGINT,                              -- FK → brand_masters.id
    brand_name        VARCHAR(80),                         -- denormalised
    hsn_code          VARCHAR(20),
    gst_pct           NUMERIC(5,2)    CHECK (gst_pct >= 0 AND gst_pct <= 100),
    standard_cost     NUMERIC(12,2),
    mrp               NUMERIC(12,2),
    moq               INT             CHECK (moq > 0),
    remarks           VARCHAR(500),
    product_image     VARCHAR(255),                        -- stored as {timestamp}!_!{filename} per upload standard
    status            VARCHAR(20)     NOT NULL DEFAULT 'DRAFT',   -- DRAFT | ACTIVE | HOLD | ARCHIVED
    created_by        BIGINT          NOT NULL,            -- FK → app_user.id
    created_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_by        BIGINT,
    updated_at        TIMESTAMPTZ,

    CONSTRAINT chk_products_status CHECK (status IN ('DRAFT','ACTIVE','HOLD','ARCHIVED'))
);

CREATE INDEX idx_products_status       ON products(status);
CREATE INDEX idx_products_category     ON products(category);
CREATE INDEX idx_products_created_at   ON products(created_at DESC);
CREATE INDEX idx_products_color_id     ON products(color_id);
CREATE INDEX idx_products_brand_id     ON products(brand_id);
```

---

### Table: `product_customer_names`
Stores multiple customer-facing names per product (prepopulated from `product_name`; user can add more).

```sql
CREATE TABLE product_customer_names (
    id             BIGSERIAL    PRIMARY KEY,
    product_id     BIGINT       NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    customer_name  VARCHAR(100) NOT NULL,
    display_order  INT          NOT NULL DEFAULT 0
);

CREATE INDEX idx_prod_cust_names_product ON product_customer_names(product_id);
```

---

### Table: `product_descriptions`
Stores multiple description lines per product (multi-entry field on the UI).

```sql
CREATE TABLE product_descriptions (
    id             BIGSERIAL    PRIMARY KEY,
    product_id     BIGINT       NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    description    VARCHAR(200) NOT NULL,
    display_order  INT          NOT NULL DEFAULT 0
);

CREATE INDEX idx_prod_desc_product ON product_descriptions(product_id);
```

---

### Table: `product_sizes`
One row per size per product, with optional quantity.

```sql
CREATE TABLE product_sizes (
    id           BIGSERIAL    PRIMARY KEY,
    product_id   BIGINT       NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    size_code    VARCHAR(20)  NOT NULL,
    size_name    VARCHAR(60),
    quantity     INT,
    display_order INT         NOT NULL DEFAULT 0
);

CREATE UNIQUE INDEX idx_prod_sizes_unique ON product_sizes(product_id, size_code);
CREATE INDEX idx_prod_sizes_product       ON product_sizes(product_id);
```

---

### Table: `product_bom`
Bill of Materials lines per product.

```sql
CREATE TABLE product_bom (
    id                   BIGSERIAL    PRIMARY KEY,
    product_id           BIGINT       NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    material_type        VARCHAR(60)  NOT NULL,   -- Fabric, Thread, Button, Accessory, etc.
    material_name        VARCHAR(100) NOT NULL,   -- existing master name or free-text (new)
    is_existing_material BOOLEAN      NOT NULL DEFAULT TRUE,
    consumption          NUMERIC(10,3),
    uom_id               BIGINT,                 -- FK → uoms_master.id
    uom_code             VARCHAR(20),            -- denormalised
    display_order        INT          NOT NULL DEFAULT 0
);

CREATE INDEX idx_prod_bom_product ON product_bom(product_id);
```

---

### Table: `product_attachments`
Uploaded files linked to a product (all types: PDF, DOCX, XLSX, images, etc.).

```sql
CREATE TABLE product_attachments (
    id            BIGSERIAL    PRIMARY KEY,
    product_id    BIGINT       NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    file_name     VARCHAR(255) NOT NULL,  -- stored as {timestamp}!_!{sanitisedOriginalName}
    original_name VARCHAR(255) NOT NULL,
    file_type     VARCHAR(50),
    file_size_kb  INT,
    uploaded_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    uploaded_by   BIGINT       NOT NULL  -- FK → app_user.id
);

CREATE INDEX idx_prod_att_product ON product_attachments(product_id);
```

---

### Product Code Auto-Generation Sequence

```sql
CREATE SEQUENCE product_code_seq START 1 INCREMENT 1;
```

Code format: `PROD-{YYYY}-{SEQ padded to 4 digits}` e.g. `PROD-2026-0001`.  
Generated in `ProductService.generateProductCode()` using:
```java
String year = String.valueOf(LocalDate.now().getYear());
long seq     = jdbcTemplate.queryForObject("SELECT nextval('product_code_seq')", Long.class);
return "PROD-" + year + "-" + String.format("%04d", seq);
```

---

### Acceptance Criteria (DB)
- [ ] Flyway migration runs cleanly on a fresh tenant schema
- [ ] All indices present and named consistently
- [ ] `status` check constraint rejects any value outside the allowed set
- [ ] `product_code` is unique within the tenant schema
- [ ] Cascade deletes verified: deleting a `products` row removes all child rows in all five child tables

---

---

## NE-PM-02 — Product List Page

**Story:**
As a user with `canList` permission on the Products link, I need a paginated, filterable list of all products so I can quickly locate, edit, view, or archive any product.

**Prototype:** `product-list.html`

---

### API: GET /cedge/api/v1/products

**Auth:** Any authenticated user with `canList` on the Products menu link  
**Query params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | int | 0 | Zero-based page index |
| `size` | int | 10 | Items per page |
| `sortBy` | String | `createdAt` | Field to sort by |
| `sortDir` | String | `desc` | `asc` or `desc` |
| `search` | String | — | Partial match on `product_name` OR `product_code` |
| `status` | String | — | Filter by status (DRAFT / ACTIVE / HOLD) |
| `category` | String | — | Filter by category |
| `dateFrom` | LocalDate | — | `created_at >= dateFrom` |
| `dateTo` | LocalDate | — | `created_at <= dateTo` |

**Response `200 OK`:**
```json
{
  "status": "SUCCESS",
  "message": "Products retrieved",
  "data": {
    "content": [
      {
        "id": 1,
        "productCode": "PROD-2026-0001",
        "productName": "Polo Shirt",
        "colorName": "Blue",
        "colorHex": "#3b82f6",
        "category": "Polo",
        "status": "ACTIVE",
        "productImageUrl": "https://cdn.example.com/acme_corp_1234/STYLE_PRODUCT/1712345678901!_!polo_blue.jpg",
        "mrp": 899.00,
        "moq": 50,
        "createdBy": "Ravi Kumar",
        "createdAt": "2026-04-10T09:30:00+05:30"
      }
    ],
    "totalElements": 42,
    "totalPages": 5,
    "size": 10,
    "number": 0
  }
}
```

**Implementation notes:**
- Use `ProductRepositoryCustom` (custom repository pattern per Rule 2) for the multi-filter query using `CriteriaBuilder`
- `productImageUrl` is built via `FileUrlUtil.buildCloudFrontUrl(tenantCode, "STYLE_PRODUCT", storedValue)` — never stored in DB
- Default filter excludes ARCHIVED products; ARCHIVED are served by a separate `GET /products/archived` endpoint
- Service method: `@Transactional(readOnly = true)`

---

### API: GET /cedge/api/v1/products/archived

**Auth:** `ADMIN` or `USER` with `canList`

Returns paginated list of archived (`status = 'ARCHIVED'`) products. Same shape as the list endpoint. Supports `search` and `category` filters only.

---

### UI Acceptance Criteria (product-list.html)

**Columns displayed:**
- # (row number)
- Product Image (thumbnail — click to fullscreen)
- Product Name (clickable → opens quick-view modal)
- Product Code
- Category
- Colors (colour swatch chips using `color_hex`)
- Creation Date
- Status (pill — DRAFT / ACTIVE / HOLD)
- Created By
- Action: **Edit** (first), **View** (second), Delete (archive)

**Filters:**
- Search (product name or code, debounced 300ms)
- Status dropdown (All / Active / Draft / Hold)
- Category dropdown (populated from distinct categories in tenant data)
- Days range selector (7 / 14 / 30 / 90 / Custom date range) — default 30 days
- Configured days range persists in `localStorage` key `cedge-product-list-days`

**Page header:**
- "New Product" button → `/style/products/create`
- Import button → opens Import modal (Excel/CSV)
- Export button → triggers `GET /products/export`
- "View Archived Products" link (below header, right-aligned) → opens Archive overlay modal

**Status count summary bar:** Draft count | Active count | Hold count (filtered to current visible set)

**Archive modal:**
- Table: Product Name | Code | Category | Archived On | Restore action
- Restore calls `PUT /products/{id}/status` with `{ status: "DRAFT" }`

---

---

## NE-PM-03 — Create Product

**Story:**
As a user with `canCreate` permission, I need to create a product with full specification including colours, sizes, BOM, images, and pricing so the product is ready for production order creation.

**Prototype:** `product-create.html`

---

### Multi-Colour Expansion Rule

When the user selects **N colours**, the backend creates **N separate product records** — one per colour — with all other data identical. All N records share the same `product_name`, BOM, sizes, pricing, and attachments. Each gets its own `product_code`, `color_id`, `color_name`, and `color_hex`.

The frontend shows a confirmation modal before submitting when N > 1 (already implemented in prototype).

---

### API: POST /cedge/api/v1/products

**Auth:** Any authenticated user with `canCreate`  
**Content-Type:** `multipart/form-data` — product data as JSON part `"product"`, image file as part `"image"` (optional), attachment files as parts `"attachments[]"` (optional, multiple).

> Alternatively, accept JSON only if the client has already uploaded the image and attachments separately via `POST /upload/image` and `POST /upload/file` and has stored values. Both flows should be supported.

**Request body (JSON part `product`):**
```json
{
  "productName": "Polo Shirt",
  "colorIds": [1, 3],
  "customerNames": ["Arrow Polo", "Polo Classic"],
  "descriptions": ["Premium pique polo with 3-button placket.", "220 GSM cotton fabric."],
  "category": "Polo",
  "brandId": 5,
  "sizes": [
    { "sizeCode": "S", "sizeName": "Small",  "quantity": 50 },
    { "sizeCode": "M", "sizeName": "Medium", "quantity": 80 }
  ],
  "uomId": 1,
  "hsnCode": "61051000",
  "gstPct": 12.00,
  "standardCost": 420.00,
  "mrp": 899.00,
  "moq": 50,
  "remarks": "Pre-approved for bulk.",
  "bom": [
    {
      "materialType": "Fabric",
      "materialName": "Cotton Twill",
      "isExistingMaterial": true,
      "consumption": 2.5,
      "uomId": 2
    },
    {
      "materialType": "Button",
      "materialName": "Pearl Button 15mm",
      "isExistingMaterial": true,
      "consumption": 6,
      "uomId": 1
    }
  ],
  "productImageStoredValue": "1712345678901!_!polo_blue.jpg",
  "attachments": [
    { "storedValue": "1712345678902!_!spec_sheet.pdf", "originalName": "spec_sheet.pdf", "fileType": "PDF", "fileSizeKb": 240 }
  ],
  "status": "DRAFT"
}
```

**Response `201 Created`:**
```json
{
  "status": "SUCCESS",
  "message": "2 products created successfully",
  "data": {
    "createdProducts": [
      { "id": 1, "productCode": "PROD-2026-0001", "colorName": "Blue" },
      { "id": 2, "productCode": "PROD-2026-0002", "colorName": "Red"  }
    ]
  }
}
```

---

### Service Logic: `ProductService.createProduct(CreateProductRequest req, Long userId)`

```
1. Validate UOM exists in uoms_master
2. Validate all colorIds exist in color_master
3. Validate brandId (if provided) exists in brand_masters
4. Validate each BOM uomId (if provided) exists in uoms_master
5. For each colorId in colorIds:
   a. Generate productCode via generateProductCode()
   b. Fetch Color entity (color_name, color_hex) from color_master
   c. Fetch UOM code from uoms_master
   d. Fetch Brand name from brand_masters (if provided)
   e. Build Product entity; set created_by = userId, created_at = NOW()
   f. Save product (flush to get id)
   g. Save product_customer_names (in order)
   h. Save product_descriptions (in order)
   i. Save product_sizes (with display_order)
   j. Save product_bom rows (with display_order)
   k. Save product_attachments
6. Return list of { id, productCode, colorName } for each created product
```

- Entire operation is `@Transactional`
- If `productName` is blank/null, `productCode` becomes the display name in list views
- If `colorIds` is empty, create one product with `color_id = null`, `color_name = null`

---

### Product Image Upload Flow

Frontend calls `POST /cedge/api/v1/upload/image` with the file before submitting the product form. The upload endpoint returns `storedValue` (e.g. `1712345678901!_!polo_blue.jpg`). Frontend passes this as `productImageStoredValue` in the product JSON.

S3 key format: `{tenantCode}/STYLE_PRODUCT/{storedValue}`

Database column `products.product_image` stores only `storedValue`. Full CloudFront URL is constructed at read time via `FileUrlUtil`.

---

### Attachment Upload Flow

Same pattern. Frontend calls `POST /cedge/api/v1/upload/file` for each attachment. Returns `storedValue`. Frontend accumulates all storedValues and sends in the `attachments[]` array.

S3 key format: `{tenantCode}/STYLE_PRODUCT/{storedValue}`

---

### Validation Rules (Server-Side)

| Field | Rule |
|-------|------|
| `uomId` | Required; must exist in `uoms_master` |
| `colorIds` | If provided, all IDs must exist in `color_master` |
| `productName` | If provided, max 100 chars |
| `remarks` | Max 500 chars |
| `gstPct` | 0–100 inclusive |
| `moq` | If provided, must be > 0 |
| `sizes[].sizeCode` | Max 20 chars; unique within the request |
| `bom[].materialType` | Required if BOM row provided |
| `bom[].materialName` | Required if BOM row provided; max 100 chars |
| `descriptions[].description` | Max 200 chars |
| `customerNames[].customerName` | Max 100 chars |

---

### UI Acceptance Criteria (product-create.html)

**Section 1 — Product Information:**
- **Product Name** — `<input maxlength="100">`, mandatory asterisk shown, auto-generates code on save if blank
- **Color** — multi-select dropdown mapped from `GET /cedge/api/v1/masters/color` (list endpoint); shows colour swatch per option; "+ Create New Color" shortcut opens inline create panel; selected colours shown as chips; multiple selection → separate product creation confirmed by modal before save
- **Customer Product Name** — pre-filled with Product Name value; user can edit; "Add another customer name" (+ button) adds a new row; each row has a delete button (hidden when only one row); max 100 chars per entry
- **Product Description** — multi-entry field (same component as Customer Product Name); "Add description line" adds a row; max 200 chars per entry
- **Product Category** — dropdown from tenant category data; "+ Create New Category" inline panel
- **Brand** — dropdown from `GET /cedge/api/v1/masters/brand`; "+ Create New Brand" inline panel
- **Product Image** — drag-and-drop or click upload area; "From Sample" link opens sample import modal; preview shown after selection with click-to-fullscreen; upload calls `POST /upload/image` immediately on file select

**Section 2 — Size & Quantity:**
- **Sizes** — multi-select dropdown from size masters; "+ Create New Size" inline panel; selected sizes appear in a table (one row per size) with a **Quantity** input beside each; table is hidden when no sizes selected
- **MOQ** (Minimum Order Quantity) — single numerical input displayed below the size-qty table; applies to the product overall, not per size

**Section 3 — Pricing & Compliance:**
- **UOM** — mandatory dropdown from `GET /cedge/api/v1/masters/uom`; "+ Create New UOM" inline panel
- **HSN Code** — numerical input
- **GST (%)** — numerical input, frontend clamps to 100
- **Standard Cost** — currency input with ₹ prefix
- **MRP** — currency input with ₹ prefix

**Section 4 — BOM Allocation:**
- Radio: **Use Existing Materials** (default) | **Add New Materials**
- Existing mode: Material Type dropdown (Fabric, Thread, Button, etc.) + Material Name dropdown (filtered by type from master data); Consumption; UOM
- New mode: Material Type dropdown + Material Name free text; Consumption; UOM
- "Import from Sample" link → opens Sample Import modal; populates BOM rows from selected sample
- "+ Add Row" adds another BOM line below; delete button on each row

**Section 5 — Remarks:** single `<textarea>` 500 char limit

**Section 6 — Attachments:** multi-file drop zone; shows attached file list with download and delete buttons; all file types accepted

**Action Bar:**
- Cancel → back to list
- Save as Draft → saves with `status = DRAFT`
- Save Product → saves with `status = ACTIVE` (or the status inferred from button choice)
- No status dropdown visible on create page (status is set by button choice)

---

---

## NE-PM-04 — Edit Product

**Story:**
As a user with `canEdit` permission, I need to edit all fields of an existing product including its status, so I can keep product data accurate and control its lifecycle.

**Prototype:** `product-edit.html`

---

### API: GET /cedge/api/v1/products/{id}

**Auth:** Any authenticated user with `canView`  
**Response `200 OK`:**
```json
{
  "status": "SUCCESS",
  "data": {
    "id": 1,
    "productCode": "PROD-2026-0001",
    "productName": "Polo Shirt",
    "colorId": 1,
    "colorName": "Blue",
    "colorHex": "#3b82f6",
    "customerNames": [
      { "id": 10, "customerName": "Arrow Polo", "displayOrder": 0 },
      { "id": 11, "customerName": "Polo Blue Edition", "displayOrder": 1 }
    ],
    "descriptions": [
      { "id": 20, "description": "Premium pique polo with 3-button placket.", "displayOrder": 0 },
      { "id": 21, "description": "220 GSM cotton fabric.", "displayOrder": 1 }
    ],
    "category": "Polo",
    "brandId": 5,
    "brandName": "Arrow",
    "sizes": [
      { "id": 30, "sizeCode": "S", "sizeName": "Small",  "quantity": 50 },
      { "id": 31, "sizeCode": "M", "sizeName": "Medium", "quantity": 80 }
    ],
    "uomId": 1,
    "uomCode": "PCS",
    "hsnCode": "61051000",
    "gstPct": 12.00,
    "standardCost": 420.00,
    "mrp": 899.00,
    "moq": 50,
    "remarks": "Pre-approved for bulk.",
    "productImageUrl": "https://cdn.example.com/...",
    "bom": [
      { "id": 40, "materialType": "Fabric", "materialName": "Cotton Twill", "isExistingMaterial": true, "consumption": 2.5, "uomId": 2, "uomCode": "MTR" }
    ],
    "attachments": [
      { "id": 50, "originalName": "spec_sheet.pdf", "fileType": "PDF", "fileSizeKb": 240, "fileUrl": "https://cdn.example.com/..." }
    ],
    "status": "ACTIVE",
    "createdBy": "Ravi Kumar",
    "createdAt": "2026-04-10T09:30:00+05:30",
    "updatedBy": "Priya Menon",
    "updatedAt": "2026-05-02T14:10:00+05:30"
  }
}
```

---

### API: PUT /cedge/api/v1/products/{id}

**Auth:** Any authenticated user with `canEdit`  
**Request body:** Same shape as create request, with the addition of `status` field.

```json
{
  "productName": "Polo Shirt",
  "colorId": 1,
  "customerNames": [...],
  "descriptions": [...],
  "sizes": [...],
  "uomId": 1,
  "status": "ACTIVE",
  ...
}
```

Note: On edit, `colorId` is a **single value** (not an array). Multi-colour expansion only happens at create time.

**Response `200 OK`:**
```json
{
  "status": "SUCCESS",
  "message": "Product updated successfully",
  "data": { "id": 1, "productCode": "PROD-2026-0001" }
}
```

**Service Logic: `ProductService.updateProduct(Long id, UpdateProductRequest req, Long userId)`**
```
1. Load Product by id; throw ResourceNotFoundException if not found or ARCHIVED
2. Validate UOM, color, brand (same rules as create)
3. Update scalar fields on entity
4. Replace child collections: delete all existing rows for product_customer_names, product_descriptions, product_sizes, product_bom; re-insert from request
5. For attachments: only delete attachments whose id is NOT in the request's attachment list (client sends back kept attachments); insert new ones
6. Update updated_by = userId, updated_at = NOW()
7. Save and return summary
```

---

### Status Transitions

Valid transitions enforced in `ProductService`:

| From | To (allowed) |
|------|-------------|
| DRAFT | ACTIVE, HOLD, ARCHIVED |
| ACTIVE | HOLD, ARCHIVED, DRAFT |
| HOLD | ACTIVE, ARCHIVED, DRAFT |
| ARCHIVED | DRAFT (restore only — via dedicated restore endpoint) |

Attempting an invalid transition returns `HTTP 422 Unprocessable Entity` with a `ValidationException`.

---

### UI Acceptance Criteria (product-edit.html)

- Page loaded via `GET /products/{id}`; URL parameter `?id=PROD-2026-0001` (or numeric id)
- Meta strip shows: Created by, Created on, Last modified by, Last modified
- All fields editable (same as create page); colour is now a **single-select** (not multi-select)
- **Status dropdown** visible in action bar: Draft / Active / Hold / Archive (Archive moves to ARCHIVED status)
- Action bar buttons: Cancel | Update Product | **Create Production Order** (blue `#2563eb`)
- "Create Production Order" navigates to `/production-orders/create?productId={id}` — production order create page pre-fills with current product data
- After successful update, toast is shown and page redirects to View page

---

---

## NE-PM-05 — View Product

**Story:**
As a user with `canView` permission, I need a read-only detail view of a product with all its specifications so I can review it without accidentally modifying data.

**Prototype:** `product-view.html`

---

### API
Reuses `GET /cedge/api/v1/products/{id}` (same endpoint as Edit uses to load data).

---

### UI Acceptance Criteria (product-view.html)

- All fields displayed in read-only format using `.rf` / `.rl` / `.rv` CSS classes from design system
- Product image shown full-width (click to fullscreen overlay)
- Pricing displayed as three card tiles: Standard Cost | MRP | MOQ
- BOM displayed in a table with colour-coded material type badges (Fabric = blue, Thread = green, Button = orange, Other = grey)
- Attachments listed with Download and View (new tab) buttons per file
- Action bar (top and bottom): Print | Edit (→ edit page) | **Create Production Order** (green `.btn-success`) | Back to List
- Print button triggers `window.print()`

---

---

## NE-PM-06 — Archive & Restore Product

### API: PUT /cedge/api/v1/products/{id}/status

**Auth:** `ADMIN` or `USER` with `canEdit`  
**Body:**
```json
{ "status": "ARCHIVED" }
```
**Response `200 OK`:**
```json
{
  "status": "SUCCESS",
  "message": "Product archived successfully"
}
```

- Sets `status = 'ARCHIVED'` and `updated_by`, `updated_at`
- Archived products are excluded from the default list endpoint
- Archived products can be restored by setting `status = 'DRAFT'` via the same endpoint

**Acceptance criteria:**
- [ ] Archived products do not appear in `GET /products` (default list)
- [ ] Archived products DO appear in `GET /products/archived`
- [ ] Restore from Archive modal sets status back to `DRAFT`
- [ ] Archived products cannot be selected as the base for a new Production Order (validate in Production Order service)

---

---

## NE-PM-07 — Bulk Import via Excel / CSV

### API: POST /cedge/api/v1/products/import

**Auth:** `ADMIN` only  
**Content-Type:** `multipart/form-data`  
**File part:** `file` (`.xlsx`, `.xls`, or `.csv`)

**Processing:**
1. Parse rows from the uploaded file
2. Validate each row (required fields, value constraints, lookup IDs by name where applicable)
3. Products that pass validation are created in batch; failures are collected
4. Return a summary: created count, skipped count, array of row-level errors

**Response `200 OK`:**
```json
{
  "status": "SUCCESS",
  "data": {
    "created": 18,
    "skipped": 2,
    "errors": [
      { "row": 5, "field": "uomCode", "message": "UOM 'BOLT' not found in master" },
      { "row": 9, "field": "productName", "message": "Exceeds 100 characters" }
    ]
  }
}
```

### API: GET /cedge/api/v1/products/import/template

Returns a downloadable Excel template file with column headers matching the import format.

### API: GET /cedge/api/v1/products/export

Returns all currently visible (non-archived) products as an Excel file. Respects the same query filters as the list endpoint (`status`, `category`, `dateFrom`, `dateTo`, `search`).

---

---

## NE-PM-08 — RBAC — Page-Level Permission Wiring

### Menu Registration

The Products link must be registered in `public.module_wise_links` so that RBAC permissions can be assigned per role.

**SQL (run as part of a public-schema Flyway migration):**
```sql
-- Assumes the Style Management module row already exists in public.modules
INSERT INTO public.module_wise_links (module_id, link_name, link_code, link_url, is_active)
SELECT m.id, 'Products', 'STYLE_PRODUCT', '/style/products', true
FROM   public.modules m
WHERE  m.module_code = 'STYLE_MGMT'
ON CONFLICT DO NOTHING;
```

### Permission Matrix (default seeded by `TenantDefaultDataService`)

| Role | canList | canView | canCreate | canEdit | canDelete |
|------|---------|---------|-----------|---------|-----------|
| ADMIN | ✅ | ✅ | ✅ | ✅ | ✅ |
| USER | ✅ | ✅ | ❌ | ❌ | ❌ |

### Endpoint Security Annotations

```java
// ProductController.java
@GetMapping               → hasAuthority('ADMIN') || hasAuthority('USER')
@GetMapping("/{id}")      → hasAuthority('ADMIN') || hasAuthority('USER')
@PostMapping              → hasAuthority('ADMIN')
@PutMapping("/{id}")      → hasAuthority('ADMIN')
@PutMapping("/{id}/status") → hasAuthority('ADMIN')
@GetMapping("/archived")  → hasAuthority('ADMIN') || hasAuthority('USER')
@PostMapping("/import")   → hasAuthority('ADMIN')
@GetMapping("/export")    → hasAuthority('ADMIN') || hasAuthority('USER')
```

---

---

## Integration Points

### 1. Color Master
- Dropdown data: `GET /cedge/api/v1/masters/color` → `List<ColorDTO>` (`id`, `colorName`, `colorHex`)
- Create New Color (inline): `POST /cedge/api/v1/masters/color` → saves to `color_master` table

### 2. Brand Master
- Dropdown data: `GET /cedge/api/v1/masters/brand` → `List<BrandDTO>` (`id`, `brandName`)
- Create New Brand (inline): `POST /cedge/api/v1/masters/brand`

### 3. UOM Master
- Dropdown data: `GET /cedge/api/v1/masters/uom` → `List<UomDTO>` (`id`, `uomCode`, `uomName`)
- Create New UOM (inline): `POST /cedge/api/v1/masters/uom` → saves to `uoms_master`

### 4. Size Master (new — if not already exists)
- Table: `size_master` (`id`, `size_code`, `size_name`)
- Dropdown: `GET /cedge/api/v1/masters/size`
- Create new: `POST /cedge/api/v1/masters/size`

> **Note:** If `size_master` does not yet exist, add it to a new Flyway migration `V9__create_product_tables.sql` along with the product tables.

```sql
CREATE TABLE size_master (
    id          BIGSERIAL   PRIMARY KEY,
    size_code   VARCHAR(20) NOT NULL UNIQUE,
    size_name   VARCHAR(60),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

INSERT INTO size_master (size_code, size_name) VALUES
  ('XS','Extra Small'),('S','Small'),('M','Medium'),('L','Large'),
  ('XL','Extra Large'),('XXL','Double Extra Large'),
  ('28','28'),('30','30'),('32','32'),('34','34'),('36','36'),('38','38');
```

### 5. Sample Import (BOM / Image)
- `GET /cedge/api/v1/samples` — returns sample list for the "Import from Sample" modal
- `GET /cedge/api/v1/samples/{id}/bom` — returns BOM lines of a sample to pre-fill product BOM
- `GET /cedge/api/v1/samples/{id}/image` — returns the sample's image stored value; product image field is pre-filled with this value

### 6. Production Order (outbound link)
- From Edit and View pages: "Create Production Order" link navigates to `/production-orders/create?productId={id}`
- Production Order create page calls `GET /cedge/api/v1/products/{id}` to pre-fill: product name, sizes, BOM, UOM, colour
- **No API changes required on the Product side** — the Production Order module consumes the existing product detail endpoint

---

---

## DTO Reference

### CreateProductRequest
```java
public record CreateProductRequest(
    String                   productName,          // nullable
    List<Long>               colorIds,             // nullable; empty = no colour
    List<CustomerNameDto>    customerNames,
    List<DescriptionDto>     descriptions,
    String                   category,
    Long                     brandId,
    List<SizeDto>            sizes,
    @NotNull Long            uomId,
    String                   hsnCode,
    BigDecimal               gstPct,
    BigDecimal               standardCost,
    BigDecimal               mrp,
    Integer                  moq,
    String                   remarks,
    List<BomLineDto>         bom,
    String                   productImageStoredValue,
    List<AttachmentDto>      attachments,
    String                   status                // DRAFT | ACTIVE
) {}
```

### UpdateProductRequest
Same as `CreateProductRequest` but replaces `colorIds: List<Long>` with `colorId: Long` (single value on edit).

### ProductListDTO
```java
public record ProductListDTO(
    Long     id,
    String   productCode,
    String   productName,
    String   colorName,
    String   colorHex,
    String   category,
    String   status,
    String   productImageUrl,   // CloudFront URL, built at read time
    BigDecimal mrp,
    Integer  moq,
    String   createdBy,
    String   createdAt
) {}
```

### ProductDetailDTO
Full detail including all child lists (customer names, descriptions, sizes, BOM, attachments). See GET response body in NE-PM-04 above.

---

---

## File Upload Standard (Mandatory Reference)

Per BACKEND CLAUDE.md §File & Image Upload Standard:

| Concern | Rule |
|---------|------|
| S3 key format | `{tenantCode}/STYLE_PRODUCT/{timestamp}!_!{sanitisedFileName}` |
| DB column value | Only `{timestamp}!_!{sanitisedFileName}` — never full path or URL |
| CloudFront URL | Built at read time by `FileUrlUtil.buildCloudFrontUrl(tenantCode, "STYLE_PRODUCT", storedValue)` |
| Upload sequence | Upload → S3 success confirmed → save storedValue to DB; never reverse order |
| Product image column | `products.product_image` |
| Attachment column | `product_attachments.file_name` |

---

---

## Completion Checklist (per BACKEND CLAUDE.md mandatory checklist)

For every API endpoint implemented under this epic:

- [ ] **Unit tests** — `ProductServiceTest` covering: create single colour, create multi-colour, blank name auto-generates code, invalid UOM throws, GST > 100 throws, update happy path, update invalid status transition, archive, restore, list with each filter, list excludes archived
- [ ] **Integration tests** — `ProductControllerTest` with `MockMvc` + real repository layer (no mocked DB): create → get → update → archive → restore flow
- [ ] **Swagger/OpenAPI** — `@Tag(name = "Products")` on controller; `@Operation(summary = "...")` on every endpoint; `@ApiResponse` for 200, 201, 400, 403, 404, 422
- [ ] **Postman Collection** — `Products` folder with: Create (single colour), Create (multi-colour), Get Detail, Update, Archive, Restore, List with filters, Import, Export; environment variable `{{productId}}` set from create response
- [ ] **CLAUDE.md update** — add `products`, `product_customer_names`, `product_descriptions`, `product_sizes`, `product_bom`, `product_attachments`, `size_master` to the tenant-schema table list

---

---

## Frontend Implementation Notes

Per FRONTEND CLAUDE.md:

- **API calls:** via Axios instance at base URL `/cedge/api/v1`; `withCredentials: true`; 401 interceptor auto-refreshes token
- **Server state:** TanStack React Query for all `GET` calls (`useQuery`); mutations via `useMutation`; invalidate `['products']` query key on create/update/archive
- **Forms:** React Hook Form + Zod resolver on Create and Edit pages
- **Routing:**

  | Path | Component | Guard |
  |------|-----------|-------|
  | `/style/products` | `ProductListPage` | `canList` |
  | `/style/products/create` | `ProductCreatePage` | `canCreate` |
  | `/style/products/:id/edit` | `ProductEditPage` | `canEdit` |
  | `/style/products/:id` | `ProductViewPage` | `canView` |

- **Sidebar nav:** driven by `GET /api/v1/menu` response; Products link appears under Style Management group once the `public.module_wise_links` row is seeded
- **Icons:** Google Material Symbols Outlined (CDN, already in `index.html`) for sidebar nav; Lucide React for inline UI icons
- **Image upload:** call `POST /upload/image` on file select (before form submit); store returned `storedValue` in React Hook Form field; display CloudFront URL for preview
- **Multi-colour confirmation modal:** shown in `ProductCreatePage` when `colorIds.length > 1`; blocked until user confirms; on confirm calls `POST /products`
- **Create Production Order button:** on View and Edit pages; navigates to `/production-orders/create?productId={id}` using React Router `useNavigate`
