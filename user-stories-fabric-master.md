# User Story: Fabric Master
*Prepared: 21 May 2026 | Ticket: NE-2 | Project: NE*
*Reference HTML: masters-fabric.html · masters-fabric-add.html · masters-fabric-edit.html · masters-fabric-view.html*

---

## Ticket NE-2 — Fabric Master

**Summary:** Fabric Master — CRUD, Multi-UOM Denominations, Per-Location Inventory Limits, Roll-Wise Toggle, Vendor Quick-Create, CSV Export/Import, Image Upload

**Story:**
As an Admin, I need to create and manage a catalogue of fabrics with physical attributes, alternate UOM conversions, per-location inventory thresholds, vendor linkage, and a reference image, so that procurement, production planning, and inventory management all operate from a single authoritative record per fabric.

---

## Business Rules

| Rule | Detail |
|------|--------|
| **Locked fields after creation** | `color_id`, `base_uom_id`, `location_id` (primary), `roll_wise` cannot be changed after creation. Edit form displays them as read-only with a lock icon. |
| **Color display format** | In dropdowns and list view: `{color_name} ({color_code})`. If no code, show name only — no empty brackets. |
| **Multi-UOM denomination** | Each alternate UOM entry requires a denomination: how many base UOM units equal 1 alternate unit (e.g. 1 Roll = 50 Meters → denomination = 50). No duplicate `alt_uom_id` per fabric. |
| **Roll-wise default** | `roll_wise` toggle is **enabled by default** on create. When enabled: inventory tracked per roll. When disabled: inventory tracked by UOM quantity only. Locked after creation. |
| **Per-location inventory limits** | One fabric can have limits for multiple locations (managed via Inventory Limits tab on edit screen). No duplicate `location_id` per fabric. |
| **Soft delete guard** | Delete blocked if fabric has available stock (`available_qty > 0`). List page shows disabled delete button with tooltip "Cannot delete — fabric has stock (N Meters)". |
| **Image upload** | Two-phase: frontend uploads file → `POST /cedge/api/v1/upload/image` → gets URL → sends URL in fabric payload. Max 1 MB, JPG/PNG only. |
| **Vendor quick-create** | Vendor can be created inline from the fabric create/edit form via a modal. Vendor Name is required; Code, Contact, Phone, GSTIN are optional. |
| **Construction Type quick-add** | New construction types added inline via a single text field. No separate master table — stored as free-text values in a `construction_type_master` lookup table. |
| **CSV locked columns** | On CSV import, `Color [LOCKED]`, `Base UOM [LOCKED]`, `Default Location [LOCKED]`, `Roll Wise [LOCKED]` columns are ignored even if changed. |
| **All tenant schema** | No `tenant_id` column needed on any table — schema-level isolation via `TenantContext`. |

---

## DB Schema

All Flyway scripts go in `db/migration/tenant/`. Suggested migration: `V{n}__create_fabric_master_tables.sql`

---

### Table: `fabric_masters`

```sql
CREATE TABLE fabric_masters (
    id                   BIGSERIAL       PRIMARY KEY,
    fabric_name          VARCHAR(200)    NOT NULL,
    fabric_code          VARCHAR(50),
    color_id             BIGINT          NOT NULL REFERENCES color_master(id),
    fabric_type          VARCHAR(100)    NOT NULL,
    base_uom_id          BIGINT          NOT NULL REFERENCES uom_master(id),
    location_id          BIGINT          NOT NULL REFERENCES warehouse_location(id),
    vendor_id            BIGINT          REFERENCES vendors_master(id),
    count_value          VARCHAR(50),
    construction         VARCHAR(100),
    construction_type_id BIGINT          REFERENCES construction_type_master(id),
    width                NUMERIC(8,2),
    gsm                  NUMERIC(8,2),
    fabric_content       VARCHAR(500),
    allowance_pct        NUMERIC(5,2),
    hsn                  VARCHAR(20),
    gst_rate             NUMERIC(5,2),
    price                NUMERIC(14,4),
    inventory_limit      NUMERIC(12,4),
    roll_wise            BOOLEAN         NOT NULL DEFAULT TRUE,
    remarks              VARCHAR(1000),
    image_url            VARCHAR(500),
    is_active            BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at           TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at           TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT uq_fabric_name UNIQUE (fabric_name),
    CONSTRAINT uq_fabric_code UNIQUE (fabric_code)
);
```

> `color_id` → `color_master(id)`, `base_uom_id` → `uom_master(id)`, `location_id` → `warehouse_location(id)` — all already exist in tenant schema.
> Column named `count_value` (not `count`) to avoid SQL reserved-word conflict.

---

### Table: `construction_type_master`

Lookup table for fabric construction types. Supports inline quick-add from the form.

```sql
CREATE TABLE construction_type_master (
    id         BIGSERIAL    PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    is_active  BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    CONSTRAINT uq_construction_type_name UNIQUE (name)
);

-- Seed rows
INSERT INTO construction_type_master (name) VALUES
    ('Plain'), ('Twill'), ('Satin'), ('Denim'),
    ('Interlock'), ('Crepe'), ('Jacquard'), ('Rib')
ON CONFLICT DO NOTHING;
```

---

### Table: `fabric_uom_denomination`

Alternate UOM entries with conversion rate relative to base UOM.

```sql
CREATE TABLE fabric_uom_denomination (
    id           BIGSERIAL      PRIMARY KEY,
    fabric_id    BIGINT         NOT NULL REFERENCES fabric_masters(id) ON DELETE CASCADE,
    alt_uom_id   BIGINT         NOT NULL REFERENCES uom_master(id),
    denomination NUMERIC(12,4)  NOT NULL CHECK (denomination > 0),
    CONSTRAINT uq_fabric_alt_uom UNIQUE (fabric_id, alt_uom_id)
);
```

`denomination` = number of **base UOM units** per 1 alternate unit.
Example: Base UOM = Meters, Alt UOM = Rolls → denomination = 50 means 1 Roll = 50 Meters.

---

### Table: `fabric_inventory_limit`

Minimum stock threshold per warehouse location per fabric.

```sql
CREATE TABLE fabric_inventory_limit (
    id          BIGSERIAL      PRIMARY KEY,
    fabric_id   BIGINT         NOT NULL REFERENCES fabric_masters(id) ON DELETE CASCADE,
    location_id BIGINT         NOT NULL REFERENCES warehouse_location(id),
    min_qty     NUMERIC(12,4)  NOT NULL DEFAULT 0,
    CONSTRAINT uq_fabric_inv_limit UNIQUE (fabric_id, location_id)
);
```

---

## API Endpoints

Base path: `/cedge/api/v1/`
All endpoints require a valid JWT. Role guard: **ADMIN** = full CRUD; **USER** = GET only (returns `403` otherwise).

---

### 1. GET `/fabrics` — Paginated List

Used by the list screen (`masters-fabric.html`).

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `page` | int | 0-based page index (default 0) |
| `size` | int | Page size (default 10) |
| `search` | string | Partial match on `fabric_name` |
| `colorId` | long | Filter by color |
| `fabricCode` | string | Partial match on `fabric_code` |
| `fabricType` | string | Exact match on `fabric_type` |

**Response:**
```json
{
  "content": [
    {
      "id": 1,
      "fabricName": "Premium Cotton Twill",
      "fabricCode": "FAB-001",
      "color": "Navy Blue",
      "colorCode": "NVY-002",
      "colorId": 3,
      "fabricType": "Woven",
      "baseUom": "Meters",
      "baseUomId": 2,
      "vendor": "Sunrise Fabrics Pvt. Ltd.",
      "rollWise": true,
      "availableQty": 320.0,
      "imageUrl": null,
      "isActive": true
    }
  ],
  "totalElements": 42,
  "totalPages": 5,
  "page": 0,
  "size": 10
}
```

> `availableQty` = current stock from `inventory_ledger` (SUM IN - SUM OUT). Read-only — not stored in `fabric_masters`.

---

### 2. GET `/fabrics/{id}` — Full Detail

Used by view screen and to prefill the edit form.

**Response:**
```json
{
  "id": 1,
  "fabricName": "Premium Cotton Twill",
  "fabricCode": "FAB-001",
  "colorId": 3,
  "color": "Navy Blue",
  "colorCode": "NVY-002",
  "fabricType": "Woven",
  "baseUomId": 2,
  "baseUom": "Meters",
  "locationId": 1,
  "location": "Main Warehouse",
  "vendorId": 5,
  "vendor": "Sunrise Fabrics Pvt. Ltd.",
  "countValue": "40s",
  "construction": "2/1 Twill",
  "constructionTypeId": 2,
  "constructionType": "Twill",
  "width": 58.0,
  "gsm": 280.0,
  "fabricContent": "100% Cotton",
  "allowancePct": 3.0,
  "hsn": "52081200",
  "gstRate": 5.0,
  "price": 185.50,
  "inventoryLimit": 500.0,
  "rollWise": true,
  "remarks": "Premium grade cotton for formal wear",
  "imageUrl": null,
  "isActive": true,
  "createdAt": "2026-05-01T10:00:00Z",
  "updatedAt": "2026-05-01T10:00:00Z",
  "uomDenominations": [
    {
      "id": 1,
      "altUomId": 5,
      "altUom": "Rolls",
      "denomination": 50.0
    }
  ],
  "inventoryLimits": [
    {
      "id": 1,
      "locationId": 1,
      "location": "Main Warehouse",
      "minQty": 500.0,
      "uom": "Meters"
    },
    {
      "id": 2,
      "locationId": 2,
      "location": "Store B",
      "minQty": 200.0,
      "uom": "Meters"
    }
  ]
}
```

---

### 3. POST `/fabrics` — Create

**Request body:**
```json
{
  "fabricName": "Premium Cotton Twill",
  "fabricCode": "FAB-001",
  "colorId": 3,
  "fabricType": "Woven",
  "baseUomId": 2,
  "locationId": 1,
  "vendorId": 5,
  "countValue": "40s",
  "construction": "2/1 Twill",
  "constructionTypeId": 2,
  "width": 58.0,
  "gsm": 280.0,
  "fabricContent": "100% Cotton",
  "allowancePct": 3.0,
  "hsn": "52081200",
  "gstRate": 5.0,
  "price": 185.50,
  "inventoryLimit": 500.0,
  "rollWise": true,
  "remarks": "Premium grade cotton",
  "imageUrl": null,
  "uomDenominations": [
    { "altUomId": 5, "denomination": 50.0 }
  ],
  "inventoryLimits": [
    { "locationId": 1, "minQty": 500.0 }
  ]
}
```

**Validation rules:**

| Field | Rule |
|-------|------|
| `fabricName` | Required, max 200 chars, unique (case-insensitive). Return `409` if duplicate. |
| `fabricCode` | Optional, max 50 chars, unique when provided. Return `409` if duplicate. |
| `colorId` | Required. Must exist in `color_master`. Return `404` if not found. |
| `fabricType` | Required, non-empty, max 100 chars. |
| `baseUomId` | Required. Must exist in `uom_master`. Return `404` if not found. |
| `locationId` | Required. Must exist in `warehouse_location`. Return `404` if not found. |
| `vendorId` | Optional. If provided, must exist in `vendors_master`. Return `404` if not found. |
| `gstRate` | Optional, must be `>= 0` and `<= 100` when provided. |
| `width`, `gsm`, `price`, `allowancePct`, `inventoryLimit` | Optional, must be `>= 0` when provided. |
| `uomDenominations[].altUomId` | Must differ from `baseUomId`. Return `422` if same. No duplicate `altUomId` in the list. |
| `uomDenominations[].denomination` | Must be `> 0`. |
| `inventoryLimits[].locationId` | Must exist in `warehouse_location`. No duplicate `locationId` in the list. |
| `inventoryLimits[].minQty` | Must be `>= 0`. |

**On success:** `201 Created`. Returns full detail response (same as `GET /fabrics/{id}`).

**Side effects (all in one `@Transactional`):**
1. Insert `fabric_masters` row.
2. Insert all `fabric_uom_denomination` rows.
3. Insert all `fabric_inventory_limit` rows.

---

### 4. PUT `/fabrics/{id}` — Update

**Request body:** Same shape as POST.

**Locked fields:** `colorId`, `baseUomId`, `locationId`, `rollWise` — silently ignored even if sent. Stored values from creation are never overwritten.

**Child record strategy — full replace in one transaction:**
- Delete all existing `fabric_uom_denomination` rows for this `fabric_id`, then re-insert from request.
- Sending an empty `[]` for `uomDenominations` deletes all existing denomination rows.
- `inventoryLimits[]` in the PUT body is accepted for convenience (full save). Tab-level sub-API (see sections 8–10) is preferred for incremental changes.

**On success:** `200 OK`. Returns full detail response.

---

### 5. DELETE `/fabrics/{id}` — Soft Delete

Sets `is_active = false`.

**Guard:** Before deactivating, check `available_qty > 0` via inventory ledger. If found, return:

```json
{
  "status": 400,
  "error": "Cannot delete. This fabric has 320 Meters of available stock."
}
```

**On success:** `204 No Content`.

---

### 6. GET `/fabrics/export` — CSV Export

Exports all active fabrics as a downloadable CSV.

**Response headers:**
```
Content-Type: text/csv; charset=UTF-8
Content-Disposition: attachment; filename="fabric_master_export.csv"
```

**CSV columns (exact order):**
```
ID, Fabric Name, Fabric Code, Color [LOCKED], Fabric Type, Base UOM [LOCKED],
Construction Type, Count, Construction, Width (inches), GSM,
Fabric Content, Allowance %, HSN Code, GST %, Price (₹),
Roll Wise [LOCKED], Default Location [LOCKED], Inventory Limit, Remarks, Available Qty
```

- `Color [LOCKED]` → format: `{color_name} ({color_code})` e.g. `Navy Blue (NVY-002)`
- `Roll Wise [LOCKED]` → `Yes` / `No`
- Use `StreamingResponseBody` to avoid OOM on large datasets.
- Prefix with UTF-8 BOM (`﻿`) for Excel compatibility.
- Accepts the same filter query parameters as `GET /fabrics` (no pagination — exports full filtered set).

---

### 7. POST `/fabrics/import` — CSV Import

Accepts `multipart/form-data` with a single `file` field (CSV, max 2 MB).

**Processing rules:**
- Parse CSV; skip the header row and empty lines.
- Match each row by `ID` column to an existing `fabric_masters` record.
- **Locked columns are ignored** even if changed: `Color [LOCKED]`, `Base UOM [LOCKED]`, `Default Location [LOCKED]`, `Roll Wise [LOCKED]`.
- Apply only editable columns: `Fabric Name`, `Fabric Code`, `Fabric Type`, `Construction Type`, `Count`, `Construction`, `Width (inches)`, `GSM`, `Fabric Content`, `Allowance %`, `HSN Code`, `GST %`, `Price (₹)`, `Inventory Limit`, `Remarks`.
- Numeric fields parsed as decimal; blank = null.
- `Construction Type` resolved by name match in `construction_type_master` (case-insensitive). Unresolvable = skip row with error.
- Process all rows; do **not** abort on first failure.

**Response:**
```json
{
  "imported": 15,
  "updated": 3,
  "skipped": 1,
  "errors": [
    { "row": 5, "message": "Construction Type 'Dobby' not found in master." },
    { "row": 9, "message": "ID 99 not found." }
  ]
}
```

**HTTP status:** `200 OK` even when some rows have errors. Return `400` only if the file cannot be parsed at all.

---

### 8. GET `/fabrics/{id}/inventory-limits` — List Inventory Limits

Used by the Inventory Limits tab on the edit screen.

**Response:**
```json
[
  { "id": 1, "locationId": 1, "location": "Main Warehouse", "minQty": 500.0, "uom": "Meters" },
  { "id": 2, "locationId": 2, "location": "Store B",        "minQty": 200.0, "uom": "Meters" }
]
```

---

### 9. POST `/fabrics/{id}/inventory-limits` — Add Inventory Limit

**Request body:**
```json
{ "locationId": 3, "minQty": 1000.0 }
```

**Validation:** `locationId` must exist. No duplicate `locationId` for the same fabric. `minQty >= 0`.
**On success:** `201 Created`. Returns the created limit with `id`, `location`, `minQty`, `uom`.

---

### 10. PUT `/fabrics/{id}/inventory-limits/{limitId}` — Update Inventory Limit

**Request body:**
```json
{ "locationId": 3, "minQty": 1500.0 }
```

**On success:** `200 OK`. Returns updated limit object.

---

### 11. DELETE `/fabrics/{id}/inventory-limits/{limitId}` — Remove Inventory Limit

**On success:** `204 No Content`.

---

### 12. GET `/construction-types` — List Construction Types

Used to populate the Construction Type dropdown on create/edit form.

**Response:**
```json
[
  { "id": 1, "name": "Plain"     },
  { "id": 2, "name": "Twill"     },
  { "id": 3, "name": "Satin"     },
  { "id": 4, "name": "Denim"     },
  { "id": 5, "name": "Interlock" },
  { "id": 6, "name": "Crepe"     },
  { "id": 7, "name": "Jacquard"  },
  { "id": 8, "name": "Rib"       }
]
```

---

### 13. POST `/construction-types` — Add Construction Type (Inline)

**Request body:**
```json
{ "name": "Terry" }
```

**Validation:** `name` required, unique (case-insensitive). Return `409` if duplicate.
**On success:** `201 Created`. Returns the new construction type object.

---

## Role-Based Access Summary

| Endpoint | ADMIN | USER |
|----------|-------|------|
| GET /fabrics | ✓ | ✓ |
| GET /fabrics/{id} | ✓ | ✓ |
| GET /fabrics/export | ✓ | ✓ |
| POST /fabrics | ✓ | ✗ 403 |
| PUT /fabrics/{id} | ✓ | ✗ 403 |
| DELETE /fabrics/{id} | ✓ | ✗ 403 |
| POST /fabrics/import | ✓ | ✗ 403 |
| GET /fabrics/{id}/inventory-limits | ✓ | ✓ |
| POST /fabrics/{id}/inventory-limits | ✓ | ✗ 403 |
| PUT /fabrics/{id}/inventory-limits/{limitId} | ✓ | ✗ 403 |
| DELETE /fabrics/{id}/inventory-limits/{limitId} | ✓ | ✗ 403 |
| GET /construction-types | ✓ | ✓ |
| POST /construction-types | ✓ | ✗ 403 |

---

## Spring Boot Controller

```java
@RestController
@RequestMapping("/api/v1/fabrics")
@Tag(name = "Fabric Master", description = "Fabric catalogue management")
@RequiredArgsConstructor
public class FabricMasterController {

    private final FabricMasterService service;

    @GetMapping
    @Operation(summary = "Paginated fabric list with filters")
    public ResponseEntity<ApiResponse<Page<FabricListDTO>>> list(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) Long colorId,
            @RequestParam(required = false) String fabricCode,
            @RequestParam(required = false) String fabricType,
            Pageable pageable) {
        return ResponseEntity.ok(ApiResponse.success(
            service.list(search, colorId, fabricCode, fabricType, pageable)));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Full fabric detail")
    public ResponseEntity<ApiResponse<FabricDetailDTO>> get(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(service.get(id)));
    }

    @PostMapping
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Create fabric")
    public ResponseEntity<ApiResponse<FabricDetailDTO>> create(
            @RequestBody @Valid CreateFabricRequest req) {
        return ResponseEntity.status(201).body(ApiResponse.success(service.create(req)));
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Update fabric (locked fields ignored)")
    public ResponseEntity<ApiResponse<FabricDetailDTO>> update(
            @PathVariable Long id,
            @RequestBody @Valid UpdateFabricRequest req) {
        return ResponseEntity.ok(ApiResponse.success(service.update(id, req)));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Soft-delete fabric (blocked if stock > 0)")
    public ResponseEntity<ApiResponse<Void>> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/export")
    @Operation(summary = "Export fabrics as CSV (StreamingResponseBody)")
    public ResponseEntity<StreamingResponseBody> export(
            @RequestParam(required = false) String search,
            @RequestParam(required = false) Long colorId,
            @RequestParam(required = false) String fabricType) {
        StreamingResponseBody body = service.exportCsv(search, colorId, fabricType);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=fabric_master_export.csv")
            .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
            .body(body);
    }

    @PostMapping("/import")
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Import fabrics from CSV")
    public ResponseEntity<ApiResponse<FabricImportResultDTO>> importCsv(
            @RequestParam("file") MultipartFile file) {
        return ResponseEntity.ok(ApiResponse.success(service.importCsv(file)));
    }

    @GetMapping("/{id}/inventory-limits")
    @Operation(summary = "List per-location inventory limits")
    public ResponseEntity<ApiResponse<List<FabricInventoryLimitDTO>>> listLimits(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(service.listLimits(id)));
    }

    @PostMapping("/{id}/inventory-limits")
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Add inventory limit for a location")
    public ResponseEntity<ApiResponse<FabricInventoryLimitDTO>> addLimit(
            @PathVariable Long id,
            @RequestBody @Valid FabricInventoryLimitRequest req) {
        return ResponseEntity.status(201).body(ApiResponse.success(service.addLimit(id, req)));
    }

    @PutMapping("/{id}/inventory-limits/{limitId}")
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Update an inventory limit")
    public ResponseEntity<ApiResponse<FabricInventoryLimitDTO>> updateLimit(
            @PathVariable Long id,
            @PathVariable Long limitId,
            @RequestBody @Valid FabricInventoryLimitRequest req) {
        return ResponseEntity.ok(ApiResponse.success(service.updateLimit(id, limitId, req)));
    }

    @DeleteMapping("/{id}/inventory-limits/{limitId}")
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Remove an inventory limit")
    public ResponseEntity<ApiResponse<Void>> deleteLimit(
            @PathVariable Long id, @PathVariable Long limitId) {
        service.deleteLimit(id, limitId);
        return ResponseEntity.noContent().build();
    }
}

@RestController
@RequestMapping("/api/v1/construction-types")
@Tag(name = "Fabric Master")
@RequiredArgsConstructor
class ConstructionTypeController {
    private final ConstructionTypeService service;

    @GetMapping
    @Operation(summary = "List all construction types")
    public ResponseEntity<ApiResponse<List<ConstructionTypeDTO>>> list() {
        return ResponseEntity.ok(ApiResponse.success(service.list()));
    }

    @PostMapping
    @PreAuthorize("hasAuthority('ADMIN') or hasAuthority('SUPER_ADMIN')")
    @Operation(summary = "Add a new construction type (inline quick-add)")
    public ResponseEntity<ApiResponse<ConstructionTypeDTO>> create(
            @RequestBody @Valid CreateConstructionTypeRequest req) {
        return ResponseEntity.status(201).body(ApiResponse.success(service.create(req)));
    }
}
```

---

## Native SQL — FabricMasterDAO

```java
@Repository
@RequiredArgsConstructor
public class FabricMasterDAO {

    private final EntityManager em;

    @SuppressWarnings("unchecked")
    public List<FabricListDTO> list(String search, Long colorId, String fabricCode,
                                    String fabricType, int page, int size) {
        String sql = """
            SELECT
                fm.id,
                fm.fabric_name,
                fm.fabric_code,
                cm.color_name,
                cm.color_code,
                cm.id        AS color_id,
                fm.fabric_type,
                um.uom_name  AS base_uom,
                um.id        AS base_uom_id,
                vm.vendor_name,
                fm.roll_wise,
                fm.image_url,
                fm.is_active,
                COALESCE(
                    (SELECT SUM(CASE WHEN il.movement_type = 'IN'  THEN il.quantity ELSE 0 END)
                           - SUM(CASE WHEN il.movement_type = 'OUT' THEN il.quantity ELSE 0 END)
                     FROM inventory_ledger il
                     WHERE il.item_type = 'FABRIC' AND il.item_id = fm.id), 0
                ) AS available_qty
            FROM fabric_masters fm
            JOIN color_master cm ON cm.id = fm.color_id
            JOIN uom_master um   ON um.id = fm.base_uom_id
            LEFT JOIN vendors_master vm ON vm.id = fm.vendor_id
            WHERE fm.is_active = true
              AND (:search     IS NULL OR fm.fabric_name ILIKE '%' || :search || '%')
              AND (:colorId    IS NULL OR fm.color_id    = :colorId)
              AND (:fabricCode IS NULL OR fm.fabric_code ILIKE '%' || :fabricCode || '%')
              AND (:fabricType IS NULL OR fm.fabric_type = :fabricType)
            ORDER BY fm.fabric_name ASC
            LIMIT :size OFFSET :offset
            """;

        return ((List<Object[]>) em.createNativeQuery(sql)
            .setParameter("search",     search)
            .setParameter("colorId",    colorId)
            .setParameter("fabricCode", fabricCode)
            .setParameter("fabricType", fabricType)
            .setParameter("size",       size)
            .setParameter("offset",     page * size)
            .getResultList()).stream().map(r -> FabricListDTO.builder()
                .id(((Number) r[0]).longValue())
                .fabricName((String) r[1])
                .fabricCode((String) r[2])
                .color((String) r[3])
                .colorCode((String) r[4])
                .colorId(((Number) r[5]).longValue())
                .fabricType((String) r[6])
                .baseUom((String) r[7])
                .baseUomId(((Number) r[8]).longValue())
                .vendor((String) r[9])
                .rollWise((Boolean) r[10])
                .imageUrl((String) r[11])
                .isActive((Boolean) r[12])
                .availableQty(r[13] != null ? ((Number) r[13]).doubleValue() : 0.0)
                .build()
        ).toList();
    }

    public long count(String search, Long colorId, String fabricCode, String fabricType) {
        String sql = """
            SELECT COUNT(*) FROM fabric_masters fm
            WHERE fm.is_active = true
              AND (:search     IS NULL OR fm.fabric_name ILIKE '%' || :search || '%')
              AND (:colorId    IS NULL OR fm.color_id    = :colorId)
              AND (:fabricCode IS NULL OR fm.fabric_code ILIKE '%' || :fabricCode || '%')
              AND (:fabricType IS NULL OR fm.fabric_type = :fabricType)
            """;
        return ((Number) em.createNativeQuery(sql)
            .setParameter("search",     search)
            .setParameter("colorId",    colorId)
            .setParameter("fabricCode", fabricCode)
            .setParameter("fabricType", fabricType)
            .getSingleResult()).longValue();
    }
}
```

---

## Service Logic — Key Methods

```java
@Service
@RequiredArgsConstructor
@Transactional
public class FabricMasterService {

    public FabricDetailDTO create(CreateFabricRequest req) {
        // Validate FK references
        colorMasterRepo.findById(req.getColorId())
            .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "Color not found"));
        uomMasterRepo.findById(req.getBaseUomId())
            .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "UOM not found"));
        warehouseLocationRepo.findById(req.getLocationId())
            .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "Location not found"));

        FabricMaster fabric = mapper.toEntity(req);
        fabric.setRollWise(req.getRollWise() != null ? req.getRollWise() : true);
        try {
            fabricRepo.save(fabric);
        } catch (DataIntegrityViolationException e) {
            throw new ApiException(HttpStatus.CONFLICT,
                e.getMessage().contains("uq_fabric_code") ? "Fabric code already exists" : "Fabric name already exists");
        }

        // Save child tables in same transaction
        if (req.getUomDenominations() != null) {
            for (var d : req.getUomDenominations()) {
                if (d.getAltUomId().equals(req.getBaseUomId()))
                    throw new ApiException(HttpStatus.UNPROCESSABLE_ENTITY,
                        "Alternate UOM must differ from base UOM");
                fabricUomDenomRepo.save(FabricUomDenomination.builder()
                    .fabricId(fabric.getId()).altUomId(d.getAltUomId()).denomination(d.getDenomination()).build());
            }
        }
        if (req.getInventoryLimits() != null) {
            for (var l : req.getInventoryLimits()) {
                fabricInventoryLimitRepo.save(FabricInventoryLimit.builder()
                    .fabricId(fabric.getId()).locationId(l.getLocationId()).minQty(l.getMinQty()).build());
            }
        }
        return toDetailDTO(fabric);
    }

    public FabricDetailDTO update(Long id, UpdateFabricRequest req) {
        FabricMaster fabric = fabricRepo.findById(id)
            .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "Fabric not found"));

        // Apply editable fields only — locked fields are never touched
        fabric.setFabricName(req.getFabricName());
        fabric.setFabricType(req.getFabricType());
        fabric.setVendorId(req.getVendorId());
        fabric.setCountValue(req.getCountValue());
        fabric.setConstruction(req.getConstruction());
        fabric.setConstructionTypeId(req.getConstructionTypeId());
        fabric.setWidth(req.getWidth());
        fabric.setGsm(req.getGsm());
        fabric.setFabricContent(req.getFabricContent());
        fabric.setAllowancePct(req.getAllowancePct());
        fabric.setHsn(req.getHsn());
        fabric.setGstRate(req.getGstRate());
        fabric.setPrice(req.getPrice());
        fabric.setInventoryLimit(req.getInventoryLimit());
        fabric.setRemarks(req.getRemarks());
        fabric.setImageUrl(req.getImageUrl());

        // Full replace for UOM denominations
        fabricUomDenomRepo.deleteAllByFabricId(id);
        if (req.getUomDenominations() != null) {
            for (var d : req.getUomDenominations()) {
                if (d.getAltUomId().equals(fabric.getBaseUomId()))
                    throw new ApiException(HttpStatus.UNPROCESSABLE_ENTITY,
                        "Alternate UOM must differ from base UOM");
                fabricUomDenomRepo.save(FabricUomDenomination.builder()
                    .fabricId(id).altUomId(d.getAltUomId()).denomination(d.getDenomination()).build());
            }
        }
        return toDetailDTO(fabric);
    }

    public void delete(Long id) {
        FabricMaster fabric = fabricRepo.findById(id)
            .orElseThrow(() -> new ApiException(HttpStatus.NOT_FOUND, "Fabric not found"));
        double availQty = inventoryLedgerRepo.sumAvailableQtyByItemTypeAndId("FABRIC", id);
        if (availQty > 0)
            throw new ApiException(HttpStatus.BAD_REQUEST,
                String.format("Cannot delete. This fabric has %.0f %s of available stock.",
                    availQty, fabric.getBaseUomName()));
        fabric.setIsActive(false);
    }
}
```

---

## React Hooks — useFabricMaster.ts

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import api from '@/lib/api';

export interface FabricFilter {
  search?: string;
  colorId?: number;
  fabricCode?: string;
  fabricType?: string;
}

export function useFabricList(filter: FabricFilter, page: number) {
  return useQuery({
    queryKey: ['fabrics', 'list', filter, page],
    queryFn: () =>
      api.get('/fabrics', { params: { ...filter, page, size: 10 } }).then(r => r.data),
    keepPreviousData: true,
  });
}

export function useFabric(id: number | null) {
  return useQuery({
    queryKey: ['fabrics', id],
    queryFn: () => api.get(`/fabrics/${id}`).then(r => r.data),
    enabled: !!id,
  });
}

export function useCreateFabric() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (req: CreateFabricRequest) =>
      api.post('/fabrics', req).then(r => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['fabrics'] }),
  });
}

export function useUpdateFabric() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, req }: { id: number; req: UpdateFabricRequest }) =>
      api.put(`/fabrics/${id}`, req).then(r => r.data),
    onSuccess: (_, { id }) => {
      qc.invalidateQueries({ queryKey: ['fabrics', id] });
      qc.invalidateQueries({ queryKey: ['fabrics', 'list'] });
    },
  });
}

export function useDeleteFabric() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: number) => api.delete(`/fabrics/${id}`).then(r => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['fabrics', 'list'] }),
  });
}

export function useFabricInventoryLimits(id: number | null) {
  return useQuery({
    queryKey: ['fabrics', id, 'inventory-limits'],
    queryFn: () => api.get(`/fabrics/${id}/inventory-limits`).then(r => r.data),
    enabled: !!id,
  });
}

export function useAddFabricInventoryLimit() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, req }: { id: number; req: { locationId: number; minQty: number } }) =>
      api.post(`/fabrics/${id}/inventory-limits`, req).then(r => r.data),
    onSuccess: (_, { id }) => qc.invalidateQueries({ queryKey: ['fabrics', id, 'inventory-limits'] }),
  });
}

export function useUpdateFabricInventoryLimit() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, limitId, req }: { id: number; limitId: number; req: { locationId: number; minQty: number } }) =>
      api.put(`/fabrics/${id}/inventory-limits/${limitId}`, req).then(r => r.data),
    onSuccess: (_, { id }) => qc.invalidateQueries({ queryKey: ['fabrics', id, 'inventory-limits'] }),
  });
}

export function useDeleteFabricInventoryLimit() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, limitId }: { id: number; limitId: number }) =>
      api.delete(`/fabrics/${id}/inventory-limits/${limitId}`).then(r => r.data),
    onSuccess: (_, { id }) => qc.invalidateQueries({ queryKey: ['fabrics', id, 'inventory-limits'] }),
  });
}

export function useConstructionTypes() {
  return useQuery({
    queryKey: ['construction-types'],
    queryFn: () => api.get('/construction-types').then(r => r.data),
    staleTime: 10 * 60 * 1000,
  });
}

export function useAddConstructionType() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (name: string) =>
      api.post('/construction-types', { name }).then(r => r.data),
    onSuccess: () => qc.invalidateQueries({ queryKey: ['construction-types'] }),
  });
}

export function useImportFabrics() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (file: File) => {
      const form = new FormData();
      form.append('file', file);
      return api.post('/fabrics/import', form, {
        headers: { 'Content-Type': 'multipart/form-data' },
      }).then(r => r.data);
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: ['fabrics'] }),
  });
}
```

---

## Zod Schema — fabricSchema.ts

```typescript
import { z } from 'zod';

export const fabricUomDenominationSchema = z.object({
  altUomId:    z.number({ required_error: 'UOM is required' }),
  denomination: z.number().positive('Denomination must be > 0'),
});

export const fabricInventoryLimitSchema = z.object({
  locationId: z.number({ required_error: 'Location is required' }),
  minQty:     z.number().min(0, 'Min qty must be ≥ 0'),
});

export const createFabricSchema = z.object({
  fabricName:         z.string().min(1, 'Fabric name is required').max(200),
  fabricCode:         z.string().max(50).optional(),
  colorId:            z.number({ required_error: 'Color is required' }),
  fabricType:         z.string().min(1, 'Fabric type is required').max(100),
  baseUomId:          z.number({ required_error: 'Base UOM is required' }),
  locationId:         z.number({ required_error: 'Location is required' }),
  vendorId:           z.number().optional(),
  countValue:         z.string().optional(),
  construction:       z.string().optional(),
  constructionTypeId: z.number().optional(),
  width:              z.number().min(0).optional(),
  gsm:                z.number().min(0).optional(),
  fabricContent:      z.string().optional(),
  allowancePct:       z.number().min(0).optional(),
  hsn:                z.string().optional(),
  gstRate:            z.number().min(0).max(100).optional(),
  price:              z.number().min(0).optional(),
  inventoryLimit:     z.number().min(0).optional(),
  rollWise:           z.boolean().default(true),
  remarks:            z.string().optional(),
  imageUrl:           z.string().optional(),
  uomDenominations:   z.array(fabricUomDenominationSchema).optional(),
  inventoryLimits:    z.array(fabricInventoryLimitSchema).optional(),
});

export const updateFabricSchema = createFabricSchema.omit({
  colorId: true, baseUomId: true, locationId: true, rollWise: true,
});

export type CreateFabricRequest = z.infer<typeof createFabricSchema>;
export type UpdateFabricRequest = z.infer<typeof updateFabricSchema>;
```

---

## Integration Tests — FabricMasterControllerTest

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class FabricMasterControllerTest {

    @Autowired MockMvc mvc;
    @Autowired ObjectMapper om;

    // T1: create with required fields — returns 201 and generated id
    @Test
    void create_requiredFields_returns201() throws Exception {
        var req = Map.of(
            "fabricName", "Test Cotton", "colorId", seedColor(),
            "fabricType", "Woven", "baseUomId", seedUom(), "locationId", seedLocation()
        );
        mvc.perform(post("/api/v1/fabrics")
                .header("Authorization", "Bearer " + adminToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.data.id").isNumber())
            .andExpect(jsonPath("$.data.rollWise").value(true)); // default
    }

    // T2: duplicate fabric name → 409
    @Test
    void create_duplicateName_returns409() throws Exception {
        seedFabric("Duplicate Fabric");
        var req = Map.of(
            "fabricName", "Duplicate Fabric", "colorId", seedColor(),
            "fabricType", "Woven", "baseUomId", seedUom(), "locationId", seedLocation()
        );
        mvc.perform(post("/api/v1/fabrics")
                .header("Authorization", "Bearer " + adminToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isConflict());
    }

    // T3: altUomId same as baseUomId → 422
    @Test
    void create_altUomSameAsBase_returns422() throws Exception {
        Long uomId = seedUom();
        var req = buildCreateReq(uomId, List.of(Map.of("altUomId", uomId, "denomination", 50.0)));
        mvc.perform(post("/api/v1/fabrics")
                .header("Authorization", "Bearer " + adminToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isUnprocessableEntity());
    }

    // T4: uomDenominations saved correctly
    @Test
    void create_withUomDenominations_savedCorrectly() throws Exception {
        Long altUomId = seedAltUom();
        var req = buildCreateReqWithDenomination(altUomId, 50.0);
        var res = mvc.perform(post("/api/v1/fabrics")
                .header("Authorization", "Bearer " + adminToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andReturn().getResponse().getContentAsString();
        List<?> denoms = JsonPath.read(res, "$.data.uomDenominations");
        assertThat(denoms).hasSize(1);
    }

    // T5: locked fields (color, baseUom, location, rollWise) not changed on update
    @Test
    void update_lockedFieldsIgnored() throws Exception {
        Long fabricId = seedFabricWithColor(seedColor1());
        Long differentColorId = seedColor2();
        var req = Map.of("fabricName", "Updated Name", "colorId", differentColorId,
            "fabricType", "Knit", "baseUomId", seedUom(), "locationId", seedLocation());
        mvc.perform(put("/api/v1/fabrics/{id}", fabricId)
                .header("Authorization", "Bearer " + adminToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isOk());
        FabricMaster saved = fabricRepo.findById(fabricId).orElseThrow();
        assertThat(saved.getColorId()).isEqualTo(seedColor1()); // unchanged
        assertThat(saved.getFabricName()).isEqualTo("Updated Name"); // changed
    }

    // T6: delete with available stock → 400
    @Test
    void delete_withStock_returns400() throws Exception {
        Long fabricId = seedFabricWithStock(100.0);
        mvc.perform(delete("/api/v1/fabrics/{id}", fabricId)
                .header("Authorization", "Bearer " + adminToken()))
            .andExpect(status().isBadRequest());
    }

    // T7: delete with zero stock → 204, is_active = false
    @Test
    void delete_zeroStock_softDeletes() throws Exception {
        Long fabricId = seedFabricWithStock(0.0);
        mvc.perform(delete("/api/v1/fabrics/{id}", fabricId)
                .header("Authorization", "Bearer " + adminToken()))
            .andExpect(status().isNoContent());
        assertThat(fabricRepo.findById(fabricId).orElseThrow().getIsActive()).isFalse();
    }

    // T8: add inventory limit, duplicate location → 409
    @Test
    void addInventoryLimit_duplicateLocation_returns409() throws Exception {
        Long fabricId = seedFabric("Fabric Inv Test");
        Long locId = seedLocation();
        addLimitDirect(fabricId, locId, 500.0);
        var req = Map.of("locationId", locId, "minQty", 300.0);
        mvc.perform(post("/api/v1/fabrics/{id}/inventory-limits", fabricId)
                .header("Authorization", "Bearer " + adminToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isConflict());
    }

    // T9: list filter by fabricType returns only matching
    @Test
    void list_filterByFabricType_returnsOnlyMatching() throws Exception {
        seedFabricWithType("Woven Fabric", "Woven");
        seedFabricWithType("Knit Fabric", "Knit");
        var res = mvc.perform(get("/api/v1/fabrics").param("fabricType", "Woven")
                .header("Authorization", "Bearer " + adminToken()))
            .andExpect(status().isOk())
            .andReturn().getResponse().getContentAsString();
        List<?> content = JsonPath.read(res, "$.data.content");
        assertThat(content).allMatch(f -> ((Map<?,?>) f).get("fabricType").equals("Woven"));
    }

    // T10: USER role on create → 403
    @Test
    void create_userRole_returns403() throws Exception {
        var req = Map.of("fabricName", "Test", "colorId", 1, "fabricType", "Woven",
            "baseUomId", 1, "locationId", 1);
        mvc.perform(post("/api/v1/fabrics")
                .header("Authorization", "Bearer " + userToken())
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(req)))
            .andExpect(status().isForbidden());
    }
}
```

---

## Screen-by-Screen Field Reference

### List Screen (`masters-fabric.html`)

**Table columns:** IMAGE · FABRIC NAME (Vendor below) · COLOR (swatch + name + code) · FABRIC CODE · FABRIC TYPE (badge) · BASE UOM · AVAIL. QTY · ROLL WISE (badge) · ACTION

**Filter bar:** Fabric Name (text search) · Color (dropdown) · Fabric Code (text search) · Fabric Type (dropdown) · Reset button

**Toolbar:** Import CSV · Export CSV · Add Fabric (→ create screen)

**Action buttons per row:** View (→ view screen) · Edit (→ edit screen) · Delete (opens confirm modal, disabled if stock > 0)

**Delete modal:** "Delete fabric `{fabricName}`? All linked UOM and inventory data will be removed."

**Import modal:** Shows per-record, per-field diff table: Field | Current Value | New Value. Warning: "Locked fields (Color, Base UOM, Location, Roll Wise) are ignored even if changed in the file."

---

### Create Screen (`masters-fabric-add.html`)

**Card 1 — Core Identification (g3 grid)**

| Row | Fields |
|-----|--------|
| 1 | Fabric Name (required) · Color (required, dropdown + inline add: Color Name*, Color Code) · Fabric Type (required, text) |
| 2 | Base UOM (required, dropdown + inline add: UOM Name*) · Multiple UOM (dropdown + inline add, shared UOM panel) · Fabric Code |
| — | Dynamic Multi-UOM denomination table: Alternative UOM · `1 {altUom} = ___ {baseUom}` denomination input · Remove |

**Card 2 — Physical & Vendor Details (g4 grid)**

| Row | Fields |
|-----|--------|
| 1 | Count · Construction · Construction Type (dropdown + inline add: Name*) · Width (numeric, inches) |
| 2 | GSM (numeric) · Fabric Content (text) · Allowance % (numeric) · Vendors (dropdown + Create New Vendor modal) |

**Vendor creation modal fields:** Vendor Name*, Vendor Code, Contact Person, Phone, GSTIN

**Card 3 — Commercial + Roll Wise (g4 grid, align-items:end)**

| Fields |
|--------|
| HSN · GST % · Price · Update Roll Wise (toggle, default ON — "Inventory tracked per roll"; OFF → "Inventory tracked by UOM only") |

**Card 4 — Location + Inventory Limit + Remarks | Image (split layout)**

| Side | Fields |
|------|--------|
| Left | Location (required, dropdown) · Inventory Limit (numeric, + button adds to dynamic list) · Dynamic per-location list (Location · Limit · Remove) · Remarks (textarea) |
| Right | Image upload (drag-drop / click, JPG/PNG max 1 MB) |

**Form actions:** Back · Save & Add New · Save

---

### Edit Screen (`masters-fabric-edit.html`)

**Tab 1 — Details** (same layout as create, locked fields shown as `.locked-field` read-only divs)

Locked fields (read-only with lock icon, cannot be changed):
- Color
- Base UOM
- Location (hint: "Use Inventory Limits tab to manage per-location limits")
- Roll Wise (toggle shown as read-only display)

All other fields remain editable. Vendor can be changed.

**Form actions:** Back · Save Changes

**Tab 2 — Inventory Limits**

Table: LOCATION · INVENTORY LIMIT · UOM · ACTION (Edit / Remove)

Footer: "+ Add Location Inventory Limit" (opens modal)

**Add/Edit modal fields:** Location (required, dropdown) · Inventory Limit (required, numeric)

Duplicate location guard: "Limit for this location already exists. Edit it instead." toast on attempt.

---

### View Screen (`masters-fabric-view.html`)

**Header:** Page title = Fabric Name · Edit button (top-right)

**Card 1 — Core Identification**

| Row | Fields |
|-----|--------|
| 1 | Fabric Name · Fabric Code (monospace) · Fabric Type |
| 2 | Color (color dot swatch + name + code) · Base UOM |
| — | Multi-UOM denomination table (if denominations exist): Alt UOM · Denomination |

**Card 2 — Physical & Vendor Details**

| Row | Fields |
|-----|--------|
| 1 | Count · Construction · Construction Type · Width |
| 2 | GSM · Fabric Content · Allowance % · Vendor |

**Card 3 — Commercial**

| Row | Fields |
|-----|--------|
| 1 | HSN Code · GST % · Price · Roll Wise (badge: green "Roll Wise" / grey "Non-Roll") |

**Card 4 — Location & Media (split)**

| Side | Content |
|------|---------|
| Left | Default Location · Inventory Limit · All Location Limits table (when multiple) · Remarks |
| Right | Fabric image or placeholder icon |

**Form actions:** Back only (read-only page)

---

## Implementation Notes

1. **Locked field enforcement on PUT** — read `colorId`, `baseUomId`, `locationId`, `rollWise` from DB before update and ignore whatever the client sent.
2. **UOM denomination full-replace** — `DELETE FROM fabric_uom_denomination WHERE fabric_id = ?`, then re-insert. Same pattern as RM Master.
3. **Color display** — always format as `{name} ({code})` in dropdown options. If `color_code` is null/blank, show name only — no empty brackets.
4. **Available qty** — computed from `inventory_ledger` (SUM IN - SUM OUT for `item_type='FABRIC'`). Never stored in `fabric_masters`. **Deferred — see § Inventory Ledger Integration below.**
5. **Delete guard** — query `inventory_ledger` for `item_type='FABRIC'` and `item_id=?` before soft-delete. If net qty > 0, return HTTP 400. **Deferred — see § Inventory Ledger Integration below.**
6. **Uniqueness conflict** — catch `DataIntegrityViolationException`, map to HTTP 409 identifying which field caused the conflict.
7. **CSV streaming** — use `StreamingResponseBody` for `/export` endpoint.
8. **Image URL** — fabric endpoints never accept file uploads. Frontend calls `POST /upload/image` first; URL passed in payload as `imageUrl` string.
9. **Construction Type** — stored as FK to `construction_type_master`. Inline quick-add via `POST /construction-types`. New type immediately available in dropdown.
10. **Vendor quick-create** — `POST /vendors` from the fabric form. On success, new vendor is selected automatically in the dropdown.
11. **Swagger** — `@Tag(name = "Fabric Master")`. All endpoints need `@Operation(summary = ...)` and `@ApiResponse` for 200/201/400/403/404/409/422.

---

## Inventory Ledger Integration (Deferred — do when Inventory Module is built)

The following two features **cannot be implemented until the `inventory_ledger` table exists**. They are intentionally skipped in the current implementation and must be completed as part of the Inventory Module epic.

---

### What is deferred

| Feature | Current state | What to build |
|---------|--------------|---------------|
| `availableQty` on Fabric list/detail | Field absent from response; list shows `—` | Compute from `inventory_ledger` and return in `FabricDTO` |
| Delete guard (stock check) | Hard delete allowed unconditionally | Block delete if net stock > 0; return HTTP 400 |

---

### When `inventory_ledger` is created, do the following

#### 1. Backend — `FabricServiceImpl`

**Add `availableQty` to `toDto()`:**

```java
// In FabricServiceImpl.toDto(), after building the DTO:
BigDecimal availableQty = inventoryLedgerRepository
    .getNetQtyByItemTypeAndItemId("FABRIC", fabric.getId());

return FabricDTO.builder()
    // ... existing fields ...
    .availableQty(availableQty != null ? availableQty : BigDecimal.ZERO)
    .build();
```

The repository query (native SQL):
```sql
SELECT COALESCE(
    SUM(CASE WHEN movement_type = 'IN'  THEN quantity ELSE 0 END) -
    SUM(CASE WHEN movement_type = 'OUT' THEN quantity ELSE 0 END),
0) AS net_qty
FROM inventory_ledger
WHERE item_type = 'FABRIC'
  AND item_id   = :itemId
```

**Add delete guard to `delete()`:**

```java
@Override
@Transactional
public ApiResponse<Void> delete(Long id) {
    FabricMaster fabric = fabricMasterRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Fabric", "id", id));

    BigDecimal stock = inventoryLedgerRepository
        .getNetQtyByItemTypeAndItemId("FABRIC", id);
    if (stock != null && stock.compareTo(BigDecimal.ZERO) > 0) {
        throw new BadRequestException(
            "Cannot delete fabric — it has " + stock.toPlainString()
            + " " + fabric.getUomName() + " in stock");
    }

    fabricMasterRepository.delete(fabric);
    return ApiResponse.success("Fabric deleted successfully", null);
}
```

#### 2. Backend — `FabricDTO`

Add the field:

```java
private BigDecimal availableQty;   // null until inventory_ledger exists
```

#### 3. Frontend — `FabricListPage.tsx`

Replace the `—` placeholder cell with the real value once the API returns it:

```tsx
// In the table row, add column:
<td className="px-6 py-4 text-right">
  <span className="text-xs font-semibold text-[#191c1f]">
    {f.availableQty != null
      ? `${f.availableQty} ${f.uomName}`
      : <span className="text-[#c7c4d8]">—</span>}
  </span>
</td>
```

Also add the column header `AVAIL. QTY` to `<thead>`.

#### 4. Frontend — Delete button guard

In `FabricListPage.tsx`, disable the delete button when `availableQty > 0`:

```tsx
{perms.canDelete && (
  <button
    onClick={() => f.availableQty && f.availableQty > 0 ? null : setConfirmId(f.id)}
    disabled={!!f.availableQty && f.availableQty > 0}
    title={f.availableQty && f.availableQty > 0
      ? `Cannot delete — fabric has ${f.availableQty} ${f.uomName} in stock`
      : 'Delete'}
    className={cn(
      "w-8 h-8 flex items-center justify-center rounded-lg transition-colors",
      f.availableQty && f.availableQty > 0
        ? "text-[#c7c4d8] cursor-not-allowed"
        : "hover:bg-red-50 text-red-500"
    )}
  >
    <span className="material-symbols-outlined text-[18px]">delete</span>
  </button>
)}
```

---

### Checklist for when Inventory Module is built

- [ ] `inventory_ledger` table created in Flyway migration (`item_type VARCHAR`, `item_id BIGINT`, `movement_type VARCHAR`, `quantity NUMERIC`)
- [ ] `InventoryLedgerRepository.getNetQtyByItemTypeAndItemId()` native query added
- [ ] `FabricDTO.availableQty` field added
- [ ] `FabricServiceImpl.toDto()` populates `availableQty` from ledger
- [ ] `FabricServiceImpl.delete()` stock guard added (HTTP 400 if stock > 0)
- [ ] `FabricListPage.tsx` — AVAIL. QTY column shows real value
- [ ] `FabricListPage.tsx` — delete button disabled when stock > 0 with tooltip
- [ ] `FabricDTO` `availableQty` field is `@JsonInclude(NON_NULL)` safe (returns `0` not `null` once ledger exists)

---

## Definition of Done

- [ ] Flyway migration: `V{n}__create_fabric_master_tables.sql` — `fabric_masters`, `construction_type_master` (with seed data), `fabric_uom_denomination`, `fabric_inventory_limit`
- [ ] `FabricMasterController` — 11 endpoints + `ConstructionTypeController` — 2 endpoints
- [ ] `FabricMasterService` — create (locked fields set at creation), update (locked fields ignored), delete (stock guard), importCsv (partial success), exportCsv (StreamingResponseBody)
- [ ] `FabricMasterDAO` — native SQL list with 4 filters + available_qty subquery *(available_qty deferred — requires inventory_ledger)*
- [x] All write endpoints `@PreAuthorize("hasAnyAuthority('SUPER_ADMIN','ORG_ADMIN')")` ✅ done
- [ ] Locked fields (`color_id`, `base_uom_id`, `location_id`, `roll_wise`) never overwritten on PUT
- [ ] React: 12 hooks in `useFabricMaster.ts`, `keepPreviousData: true` on list
- [ ] Zod schema: `createFabricSchema` + `updateFabricSchema` (locked fields omitted)
- [ ] 10 integration tests passing
- [ ] Swagger docs + Postman collection updated

---

## Reference Files

| Screen | File |
|--------|------|
| List | `masters-fabric.html` |
| Create | `masters-fabric-add.html` |
| Edit (2-tab) | `masters-fabric-edit.html` |
| View | `masters-fabric-view.html` |

*Backend team: implement DB migration → service layer → controllers in that order. Frontend team: refer to the HTML files for exact field layout, locked field display, toggle behavior, and modal flows.*
