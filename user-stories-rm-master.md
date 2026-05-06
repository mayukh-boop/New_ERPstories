# User Story: Raw Material Master
*Prepared: 06 May 2026 | Ticket: NE-30 | Project: NE*
*Reference HTML: masters-rawmaterial.html · masters-rawmaterial-add.html · masters-rawmaterial-edit.html · masters-rawmaterial-view.html*

---

## Ticket NE-30 — Raw Material Master

**Summary:** Raw Material Master — CRUD, Multi-UOM Denominations, Per-Location Inventory Limits, Scale-Wise Sizes, CSV Export/Import, Image Upload

**Story:**
As an Admin, I need to create and manage a catalogue of raw materials with commercial attributes, alternate UOM conversions, per-location inventory thresholds, and a reference image, so that procurement, production planning, and stock management all operate from a single authoritative record per material.

---

## Business Rules

| Rule | Detail |
|------|--------|
| **BOM Status auto-populate** | When RM Type is selected, BOM Status toggle pre-fills with `rm_type.default_bom_status`. User may override before saving. |
| **Locked fields after creation** | `rm_type_id`, `color_id`, `base_uom_id` cannot be changed after a record is created. Edit form displays them as read-only with a lock icon. |
| **Location locked in edit** | The primary location is locked after creation. Additional per-location limits are managed via the Inventory Limits tab on the edit screen. |
| **Scale Wise → Sizes dependency** | When `scale_wise = false`, `sizes` must be empty. When `scale_wise = true`, the size multi-select is enabled and selected sizes are saved. |
| **Multi-UOM denomination** | Each alternate UOM entry requires a denomination: how many base UOM units equal 1 alternate unit (e.g. 1 Cone = 1000 Grams → denomination = 1000). No duplicate `alt_uom_id` per RM allowed. |
| **Per-location inventory limits** | One RM can have limits for multiple locations. No duplicate `location_id` per RM. |
| **Soft delete** | Deactivate only (`is_active = false`). Hard delete is blocked if the RM is referenced in any open PO or BOM. |
| **Image upload** | Two-phase: frontend uploads file → `POST /cedge/api/v1/upload/image` → gets URL → sends URL in RM payload. Max 1 MB, JPG/PNG only. |
| **CSV locked columns** | On CSV import, `Color`, `Base UOM`, and `Location` columns are ignored even if changed. Only editable fields are applied. |
| **All tenant schema** | No `tenant_id` column needed on any table — schema-level isolation via `TenantContext`. |

---

## DB Schema

All Flyway scripts go in `db/migration/tenant/`.

---

### Table: `rm_type`

```sql
CREATE TABLE rm_type (
    id                 BIGSERIAL    PRIMARY KEY,
    name               VARCHAR(100) NOT NULL,
    default_bom_status BOOLEAN      NOT NULL DEFAULT TRUE,
    is_active          BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at         TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ  NOT NULL DEFAULT now(),
    CONSTRAINT uq_rm_type_name UNIQUE (name)
);
```

**Seed rows** (insert if not exists on tenant provisioning):

| Name | default_bom_status |
|------|--------------------|
| Yarn | true |
| Thread | true |
| Elastic | true |
| Interlining | true |
| Dye | true |
| Zipper | false |
| Button | false |
| Label | false |
| Packing Material | false |

---

### Table: `rm_master`

```sql
CREATE TABLE rm_master (
    id              BIGSERIAL      PRIMARY KEY,
    rm_type_id      BIGINT         NOT NULL REFERENCES rm_type(id),
    rm_name         VARCHAR(200)   NOT NULL,
    rm_code         VARCHAR(50),
    description     VARCHAR(500),
    shade_code      VARCHAR(50),
    brand_item      VARCHAR(100),
    color_id        BIGINT         NOT NULL REFERENCES color_master(id),
    base_uom_id     BIGINT         NOT NULL REFERENCES uom_master(id),
    hsn             VARCHAR(20)    NOT NULL,
    gst_rate        NUMERIC(5,2)   NOT NULL,
    price           NUMERIC(14,4),
    allowance_pct   NUMERIC(5,2),
    package_qty     NUMERIC(12,4),
    count_value     VARCHAR(50),
    gsm             NUMERIC(8,2),
    bom_status      BOOLEAN        NOT NULL DEFAULT TRUE,
    scale_wise      BOOLEAN        NOT NULL DEFAULT FALSE,
    image_url       VARCHAR(500),
    is_active       BOOLEAN        NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ    NOT NULL DEFAULT now(),
    CONSTRAINT uq_rm_name UNIQUE (rm_name),
    CONSTRAINT uq_rm_code UNIQUE (rm_code)
);
```

> `color_id` → `color_master(id)`, `base_uom_id` → `uom_master(id)` — both tables already exist in the tenant schema.
> Column named `count_value` (not `count`) to avoid SQL reserved-word conflict.

---

### Table: `rm_sizes`

Stores selected sizes per RM when `scale_wise = true`. Cascade-deleted with parent.

```sql
CREATE TABLE rm_sizes (
    id          BIGSERIAL   PRIMARY KEY,
    rm_id       BIGINT      NOT NULL REFERENCES rm_master(id) ON DELETE CASCADE,
    size_value  VARCHAR(30) NOT NULL,
    CONSTRAINT uq_rm_size UNIQUE (rm_id, size_value)
);
```

---

### Table: `rm_uom_denomination`

Alternate UOM entries with their conversion rate relative to base UOM.

```sql
CREATE TABLE rm_uom_denomination (
    id           BIGSERIAL      PRIMARY KEY,
    rm_id        BIGINT         NOT NULL REFERENCES rm_master(id) ON DELETE CASCADE,
    alt_uom_id   BIGINT         NOT NULL REFERENCES uom_master(id),
    denomination NUMERIC(12,4)  NOT NULL CHECK (denomination > 0),
    CONSTRAINT uq_rm_alt_uom UNIQUE (rm_id, alt_uom_id)
);
```

`denomination` = number of **base UOM units** per 1 alternate unit.
Example: Base UOM = Grams, Alt UOM = Cones → denomination = 1000 means 1 Cone = 1000 Grams.

---

### Table: `rm_inventory_limit`

Minimum stock threshold per warehouse location per RM.

```sql
CREATE TABLE rm_inventory_limit (
    id           BIGSERIAL      PRIMARY KEY,
    rm_id        BIGINT         NOT NULL REFERENCES rm_master(id) ON DELETE CASCADE,
    location_id  BIGINT         NOT NULL REFERENCES warehouse_location(id),
    min_qty      NUMERIC(12,4)  NOT NULL DEFAULT 0,
    CONSTRAINT uq_rm_inv_limit UNIQUE (rm_id, location_id)
);
```

> `warehouse_location(id)` already exists from Aisle/Inventory master.

---

## API Endpoints

Base path: `/cedge/api/v1/`
All endpoints require a valid JWT. Role guard: **ADMIN** = full CRUD; **USER** = GET only (returns `403` otherwise).

---

### 1. GET `/raw-materials` — Paginated List

Used by the list screen (`masters-rawmaterial.html`).

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | int | 0-based page index (default 0) |
| `size` | int | Page size (default 10) |
| `search` | string | Partial match on `rm_name` or `rm_code` |
| `rmTypeId` | long | Filter by RM type |
| `rmCode` | string | Partial match on `rm_code` |

**Response:**
```json
{
  "content": [
    {
      "id": 1,
      "rmName": "Grey Cotton Yarn",
      "rmCode": "RMY-001",
      "rmType": "Yarn",
      "rmTypeId": 1,
      "color": "Stone Grey",
      "colorCode": "GRY-005",
      "colorId": 3,
      "baseUom": "Kilograms",
      "baseUomId": 2,
      "sizesSummary": "40s, 60s",
      "bomStatus": true,
      "scaleWise": true,
      "imageUrl": null,
      "isActive": true,
      "createdAt": "2026-05-01T10:00:00Z"
    }
  ],
  "totalElements": 42,
  "totalPages": 5,
  "page": 0,
  "size": 10
}
```

> `sizesSummary` = comma-joined size values (e.g. `"40s, 60s"`). For display in the SIZE column only. Full `sizes` array is returned in the detail endpoint.

---

### 2. GET `/raw-materials/{id}` — Full Detail

Used by view screen and to prefill the edit form.

**Response:**
```json
{
  "id": 1,
  "rmTypeId": 1,
  "rmType": "Yarn",
  "rmName": "Grey Cotton Yarn",
  "rmCode": "RMY-001",
  "description": "100% cotton ring-spun yarn for knitting and weaving.",
  "shadeCode": "SHD-010",
  "brandItem": null,
  "colorId": 3,
  "color": "Stone Grey",
  "colorCode": "GRY-005",
  "baseUomId": 2,
  "baseUom": "Kilograms",
  "hsn": "52041100",
  "gstRate": 5.00,
  "price": 220.00,
  "allowancePct": 2.00,
  "packageQty": 50.00,
  "countValue": "40s",
  "gsm": null,
  "bomStatus": true,
  "scaleWise": true,
  "sizes": ["40s", "60s"],
  "imageUrl": null,
  "isActive": true,
  "createdAt": "2026-05-01T10:00:00Z",
  "updatedAt": "2026-05-01T10:00:00Z",
  "uomDenominations": [
    {
      "id": 1,
      "altUomId": 5,
      "altUom": "Grams",
      "denomination": 1000.0
    }
  ],
  "inventoryLimits": [
    {
      "id": 1,
      "locationId": 2,
      "location": "Raw Material Store",
      "minQty": 500.0
    },
    {
      "id": 2,
      "locationId": 1,
      "location": "Main Warehouse",
      "minQty": 200.0
    }
  ]
}
```

---

### 3. POST `/raw-materials` — Create

**Request body:**
```json
{
  "rmTypeId": 1,
  "rmName": "Grey Cotton Yarn",
  "rmCode": "RMY-001",
  "description": "100% cotton ring-spun yarn",
  "shadeCode": "SHD-010",
  "brandItem": null,
  "colorId": 3,
  "baseUomId": 2,
  "hsn": "52041100",
  "gstRate": 5.00,
  "price": 220.00,
  "allowancePct": 2.00,
  "packageQty": 50.00,
  "countValue": "40s",
  "gsm": null,
  "bomStatus": true,
  "scaleWise": true,
  "sizes": ["40s", "60s"],
  "imageUrl": null,
  "uomDenominations": [
    { "altUomId": 5, "denomination": 1000.0 }
  ],
  "inventoryLimits": [
    { "locationId": 2, "minQty": 500.0 }
  ]
}
```

**Validation rules:**

| Field | Rule |
|-------|------|
| `rmTypeId` | Required. Must exist and be active in `rm_type`. Return `404` if not found. |
| `colorId` | Required. Must exist in `color_master`. Return `404` if not found. |
| `baseUomId` | Required. Must exist in `uom_master`. Return `404` if not found. |
| `rmName` | Required, max 200 chars, unique (case-insensitive). Return `409` if duplicate. |
| `rmCode` | Optional, max 50 chars, unique when provided. Return `409` if duplicate. |
| `hsn` | Required, non-empty. |
| `gstRate` | Required, must be `>= 0`. |
| `price`, `allowancePct`, `packageQty`, `gsm` | Optional, must be `>= 0` when provided. |
| `scaleWise = false` | `sizes` must be null or empty. Return `422` if sizes are provided. |
| `uomDenominations[].altUomId` | Must differ from `baseUomId`. Return `422` if same. No duplicate `altUomId` in the list. |
| `uomDenominations[].denomination` | Must be `> 0`. |
| `inventoryLimits[].locationId` | Must exist in `warehouse_location`. No duplicate `locationId` in the list. |
| `inventoryLimits[].minQty` | Must be `>= 0`. |

**On success:** `201 Created`. Returns full detail response (same as `GET /raw-materials/{id}`).

**Side effects (all in one transaction):**
1. Insert `rm_master` row.
2. Insert all `rm_sizes` rows (only when `scaleWise = true` and sizes provided).
3. Insert all `rm_uom_denomination` rows.
4. Insert all `rm_inventory_limit` rows.

---

### 4. PUT `/raw-materials/{id}` — Update

**Request body:** Same shape as POST.

**Locked fields:** `rmTypeId`, `colorId`, `baseUomId` — these are silently ignored even if sent. The stored values from the original creation are never overwritten.

**Child record strategy — full replace in one transaction:**
- Delete all existing `rm_sizes` rows for this `rm_id`, then re-insert from request.
- Delete all existing `rm_uom_denomination` rows for this `rm_id`, then re-insert from request.
- If `scaleWise = false`, all `rm_sizes` rows are deleted regardless of what the client sends.
- Sending an empty array `[]` for `uomDenominations` or `inventoryLimits` deletes all existing child rows.

> **Inventory limits on the edit form use a separate tab and a dedicated sub-API (see Section 8). The PUT endpoint also accepts `inventoryLimits[]` for convenience on full saves.**

**On success:** `200 OK`. Returns full detail response.

---

### 5. DELETE `/raw-materials/{id}` — Soft Delete

Sets `is_active = false`.

**Guard:** Before deactivating, check for references in any open purchase order lines or active BOM lines. If found, return:

```json
{
  "status": 400,
  "error": "Cannot deactivate. This raw material is referenced in 2 open purchase order(s)."
}
```

**On success:** `204 No Content`.

---

### 6. GET `/raw-materials/export` — CSV Export

Exports all active raw materials as a downloadable CSV.

**Response headers:**
```
Content-Type: text/csv; charset=UTF-8
Content-Disposition: attachment; filename="rawmaterial_master_export.csv"
```

**CSV columns (exact order):**
```
ID, RM Name, RM Code, RM Type, Color [LOCKED], Base UOM [LOCKED], Location [LOCKED],
Size, Shade Code, Brand Item, HSN Code, GST Rate %, Price (₹),
Allowance %, Package Qty, Count, GSM, Scale Wise, BOM Status, Inventory Limit
```

- `Color [LOCKED]` → format: `{color_name} ({color_code})` e.g. `Stone Grey (GRY-005)`
- `Size` → comma-joined sizes e.g. `40s, 60s`
- `Scale Wise`, `BOM Status` → `Yes` / `No`
- `Location [LOCKED]` → first inventory limit location name, or blank if none
- `Inventory Limit` → first inventory limit `min_qty`, or blank if none
- Use `StreamingResponseBody` to avoid loading all rows into memory.
- Prefix BOM with UTF-8 BOM (`﻿`) for Excel compatibility.
- Accepts the same filter query parameters as `GET /raw-materials` (no pagination — exports full filtered set).

---

### 7. POST `/raw-materials/import` — CSV Import

Accepts `multipart/form-data` with a single `file` field (CSV, max 2 MB).

**Processing rules:**
- Parse CSV; skip the header row and empty lines.
- Match each row by `ID` column to an existing `rm_master` record.
- **Locked columns are ignored** even if changed: `Color [LOCKED]`, `Base UOM [LOCKED]`, `Location [LOCKED]`.
- Apply only editable columns: `RM Name`, `RM Code`, `RM Type`, `Size`, `Shade Code`, `Brand Item`, `HSN Code`, `GST Rate %`, `Price (₹)`, `Allowance %`, `Package Qty`, `Count`, `GSM`, `Scale Wise`, `BOM Status`, `Inventory Limit`.
- `Scale Wise` and `BOM Status` values: `Yes` → `true`, `No` → `false` (case-insensitive).
- Numeric fields parsed as decimal; blank = null.
- `RM Type` resolved by name match in `rm_type` table (case-insensitive).
- Rows with unresolvable `RM Type` or missing/invalid `ID` are skipped with an error entry.
- Process all rows; do **not** abort on first failure.

**Response:**
```json
{
  "imported": 18,
  "updated": 3,
  "skipped": 1,
  "errors": [
    { "row": 5, "message": "RM Type 'Foam Padding' not found in master." },
    { "row": 9, "message": "ID 99 not found." }
  ]
}
```

**HTTP status:** `200 OK` even when some rows have errors (partial success). Return `400` only if the file cannot be parsed at all.

---

### 8. GET `/raw-materials/{id}/inventory-limits` — List Inventory Limits

Used by the Inventory Limits tab on the edit screen.

**Response:**
```json
[
  { "id": 1, "locationId": 2, "location": "Raw Material Store", "minQty": 500.0, "uom": "Kilograms" },
  { "id": 2, "locationId": 1, "location": "Main Warehouse",     "minQty": 200.0, "uom": "Kilograms" }
]
```

---

### 9. POST `/raw-materials/{id}/inventory-limits` — Add Inventory Limit

**Request body:**
```json
{ "locationId": 3, "minQty": 1000.0 }
```

**Validation:** `locationId` must exist. No duplicate `locationId` for the same RM. `minQty >= 0`.
**On success:** `201 Created`. Returns the created limit with `id`, `location`, `minQty`, `uom`.

---

### 10. PUT `/raw-materials/{id}/inventory-limits/{limitId}` — Update Inventory Limit

**Request body:**
```json
{ "locationId": 3, "minQty": 1500.0 }
```

**On success:** `200 OK`. Returns updated limit object.

---

### 11. DELETE `/raw-materials/{id}/inventory-limits/{limitId}` — Remove Inventory Limit

**On success:** `204 No Content`.

---

### 12. GET `/rm-types` — List RM Types

Used to populate the RM Type dropdown on create/edit form.

**Response:**
```json
[
  { "id": 1, "name": "Yarn",             "defaultBomStatus": true  },
  { "id": 2, "name": "Thread",           "defaultBomStatus": true  },
  { "id": 3, "name": "Zipper",           "defaultBomStatus": false },
  { "id": 4, "name": "Button",           "defaultBomStatus": false },
  { "id": 5, "name": "Elastic",          "defaultBomStatus": true  },
  { "id": 6, "name": "Interlining",      "defaultBomStatus": true  },
  { "id": 7, "name": "Label",            "defaultBomStatus": false },
  { "id": 8, "name": "Packing Material", "defaultBomStatus": false },
  { "id": 9, "name": "Dye",              "defaultBomStatus": true  }
]
```

---

### 13. POST `/rm-types` — Add RM Type (Inline)

Used from the "Add New RM Type" inline panel on the create/edit form.

**Request body:**
```json
{ "name": "Woven Tape", "defaultBomStatus": true }
```

**Validation:** `name` required, unique (case-insensitive). Return `409` if duplicate.
**On success:** `201 Created`. Returns the new `rm_type` object.

---

## Role-Based Access Summary

| Endpoint | ADMIN | USER |
|----------|-------|------|
| GET /raw-materials | ✓ | ✓ |
| GET /raw-materials/{id} | ✓ | ✓ |
| GET /raw-materials/export | ✓ | ✓ |
| POST /raw-materials | ✓ | ✗ 403 |
| PUT /raw-materials/{id} | ✓ | ✗ 403 |
| DELETE /raw-materials/{id} | ✓ | ✗ 403 |
| POST /raw-materials/import | ✓ | ✗ 403 |
| GET /raw-materials/{id}/inventory-limits | ✓ | ✓ |
| POST /raw-materials/{id}/inventory-limits | ✓ | ✗ 403 |
| PUT /raw-materials/{id}/inventory-limits/{limitId} | ✓ | ✗ 403 |
| DELETE /raw-materials/{id}/inventory-limits/{limitId} | ✓ | ✗ 403 |
| GET /rm-types | ✓ | ✓ |
| POST /rm-types | ✓ | ✗ 403 |

---

## Screen-by-Screen Field Reference

### List Screen (`masters-rawmaterial.html`)

**Table columns:** IMAGE · RM NAME (RM Code below) · RM TYPE (badge) · COLOR (swatch + name + code) · SIZE · BASE UOM · BOM STATUS (badge) · ACTION

**Filter bar:** RM Name (text search) · RM Type (dropdown) · RM Code (text search) · Reset button

**Toolbar:** Import CSV · Export CSV · Add Raw Material (→ create screen)

**Action buttons per row:** View (→ view screen) · Edit (→ edit screen) · Delete (opens confirm modal)

**Delete modal:** "Delete raw material `{rmName}`? All linked UOM and inventory data will be removed." → Cancel / Delete buttons

**Import modal (preview before apply):** Shows per-record, per-field diff table: Field | Current Value | New Value. Warning note: "Locked fields (Color, Base UOM, Location) are ignored even if changed in the file."

---

### Create Screen (`masters-rawmaterial-add.html`)

**Card 1 — Core Identification**

| Row | Layout | Fields |
|-----|--------|--------|
| 1 | g3 | RM Type (required, dropdown + inline add panel) · RM Name (required) · RM Code |
| 2 | g2 | Description · Shade Code |
| 3 | g3 | Color (required, dropdown + inline add) · Base UOM (required, dropdown + inline add) · Multiple UOM (dropdown + inline add) |
| — | below Row 3 | Multi-UOM denomination table: columns = Alternative UOM · Denomination (1 {altUom} = ___ {baseUom}) · Remove |
| — | inline flex | Scale Wise (toggle) · Size (closed multi-select dropdown, enabled only when Scale Wise = Yes) |

**RM Type inline add panel fields:** RM Type Name (required) · BOM Status toggle (default Yes)

**Color inline add panel fields:** Color Name (required) · Color Code

**UOM inline add panel fields:** UOM Name (required) — shared panel for Base UOM and Multiple UOM

**Size inline add panel fields:** Size Value (required, e.g. 4XL, 10cm)

**Size dropdown grouped options:**
- Garment Sizes: XS · S · M · L · XL · XXL · XXXL
- Numeric Sizes: 28 · 30 · 32 · 34 · 36 · 38 · 40 · 42 · 44
- RM Specific: 2cm · 3cm · 5cm · 18L · 20L · 24L · 22cm · 30cm · 40s · 60s · 80s · Free Size

**Card 2 — Commercial Details**

| Row | Layout | Fields |
|-----|--------|--------|
| 1 | g4 | HSN (required) · GST Rate % (required) · Price · Allowance % |
| 2 | g4 align-items:end | Package Qty · Count · GSM · BOM Status (toggle, auto-populated from RM Type) |

**Card 3 — Inventory & Media (split layout)**

| Side | Fields |
|------|--------|
| Left | Location (required, dropdown) · Inventory Limit (number) + Add button → dynamic list below |
| Right | Image Ref (drag-drop / click upload, JPG/PNG max 1 MB) |

**Form actions:** Back · Save & Add New · Save

---

### Edit Screen (`masters-rawmaterial-edit.html`)

**Tab 1 — Details** (same field layout as create but locked fields shown as read-only divs with lock icon)

Locked fields (displayed as `.locked-field`, not editable):
- RM Type (locked)
- Color (locked)
- Base UOM (locked)
- Location (locked — hint: "Use Inventory Limits tab to manage per-location limits")

Editable fields remain the same as the create form.

Card 3 shows: Location (locked display) · Inventory Limit (input) on left; image upload on right.

**Form action:** Back · Save Changes

**Tab 2 — Inventory Limits**

Table: LOCATION · INVENTORY LIMIT · UOM · ACTION (Edit / Remove)
Footer row: "+ Add Location Inventory Limit" (opens modal)

**Add/Edit Inventory Limit modal fields:** Location (required, dropdown) · Inventory Limit (required, number)

Duplicate location guard: "Limit for this location already exists. Edit it instead." toast on duplicate.

**Form action:** Back · Save Changes

---

### View Screen (`masters-rawmaterial-view.html`)

**Header:** Page title = RM Name · Edit button (top-right, links to edit screen)

**Card 1 — Core Identification**

| Row | Layout | Fields |
|-----|--------|--------|
| 1 | g3 | RM Type (`.type-chip` styled badge) · RM Name · RM Code (monospace) |
| 2 | g2 | Color (color dot swatch + name + code) · Base UOM |
| 3 | g2 | Description · Shade Code (monospace) |
| — | below | Multi-UOM denomination table (visible only if denominations exist): Alt UOM · Denomination |
| — | inline flex | Scale Wise (`.scale-badge`) · Size chips (flex-wrap, visible only if `scaleWise = true`) |

**Card 2 — Commercial Details**

| Row | Layout | Fields |
|-----|--------|--------|
| 1 | g4 | HSN Code (monospace) · GST Rate % · Price · Allowance % |
| 2 | g4 | Package Qty · Count · GSM · BOM Status (`.bom-badge` — green "Yes" / grey "No") |

**Card 3 — Location & Media (split layout)**

| Side | Content |
|------|---------|
| Left | Default Location · Inventory Limit (single primary values shown in g2) · "All Location Limits" table (shown only when multiple limits exist): LOCATION · LIMIT · UOM |
| Right | RM image if `imageUrl` present; placeholder icon if not |

---

## Data Model Summary

```
rm_type
  id, name, default_bom_status, is_active

rm_master                                              ← main record
  id
  rm_type_id      FK → rm_type         [LOCKED after creation]
  color_id        FK → color_master    [LOCKED after creation]
  base_uom_id     FK → uom_master      [LOCKED after creation]
  rm_name, rm_code (unique), description, shade_code, brand_item
  hsn, gst_rate, price, allowance_pct
  package_qty, count_value, gsm
  bom_status      boolean (default from rm_type.default_bom_status, overridable)
  scale_wise      boolean (controls whether sizes are enabled)
  image_url
  is_active, created_at, updated_at

rm_sizes                                               ← child, cascade-delete
  id, rm_id FK → rm_master, size_value
  UNIQUE (rm_id, size_value)
  Populated only when scale_wise = true

rm_uom_denomination                                    ← child, cascade-delete
  id, rm_id FK → rm_master, alt_uom_id FK → uom_master, denomination
  UNIQUE (rm_id, alt_uom_id)
  denomination = base UOM units per 1 alt unit

rm_inventory_limit                                     ← child, cascade-delete
  id, rm_id FK → rm_master, location_id FK → warehouse_location, min_qty
  UNIQUE (rm_id, location_id)
```

---

## Implementation Notes

1. **Locked field enforcement on PUT** — read `rmTypeId`, `colorId`, `baseUomId` from the DB before the update and ignore whatever the client sent for those fields. Never allow them to be overwritten.

2. **Child table full-replace** — in a single transaction: `DELETE FROM rm_sizes WHERE rm_id = ?`, then re-insert. Same for `rm_uom_denomination`. This keeps the logic simple and avoids complex diff logic.

3. **BOM Status default** — the backend does NOT auto-resolve BOM status from RM type. The frontend sends the already-resolved value. Backend simply stores the boolean.

4. **`count_value` column** — `count` is a reserved SQL keyword; use `count_value` in DB. DTO field = `countValue`.

5. **Uniqueness conflict response** — catch `DataIntegrityViolationException` in the service layer and map to `409 Conflict` with a message identifying which field caused the conflict (`rm_name` or `rm_code`).

6. **CSV streaming** — use Spring's `StreamingResponseBody` for `/export` to avoid OOM on large datasets.

7. **Soft delete guard** — query for open PO lines or BOM lines referencing `rm_id` before setting `is_active = false`. If found, return `400` with a count in the message.

8. **Image URL** — RM endpoints never accept file uploads. Frontend calls `POST /upload/image` first; the returned URL is passed in the RM create/update payload as `imageUrl` string.

9. **Import — partial success** — collect all row errors into a list; never throw mid-import. Return `200` with the error list so the client can display which rows failed.

10. **Swagger** — annotate with `@Tag(name = "Raw Material Master")`. All endpoints need `@Operation(summary = ...)` and `@ApiResponse` for `200/201/400/403/404/409/422`.

11. **Mandatory completion checklist** — per BACKEND CLAUDE.md: test cases (happy path + all error scenarios) · Swagger annotations · Postman collection update before marking this ticket done.

---

## Reference Files

| Screen | File |
|--------|------|
| List | `masters-rawmaterial.html` |
| Create | `masters-rawmaterial-add.html` |
| Edit (2-tab) | `masters-rawmaterial-edit.html` |
| View | `masters-rawmaterial-view.html` |

*Backend team: implement DB migration → service layer → controllers in that order. Frontend team: refer to the HTML files for exact field layout, locked field display, toggle behavior, and modal flows.*
