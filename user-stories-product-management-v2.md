# Product Management — User Stories
**Module:** Style Management → Products
**Prepared:** 06 May 2026
**Reference Prototypes:** product-list.html · product-create.html · product-edit.html · product-view.html
**Epic:** NE-PM

---

## Story Index

| Story ID | Title | Priority | Points |
|----------|-------|----------|--------|
| NE-PM-01 | Product List Page | High | 5 |
| NE-PM-02 | Create Product | High | 13 |
| NE-PM-03 | Edit Product | High | 8 |
| NE-PM-04 | View Product | Medium | 3 |
| NE-PM-05 | Archive & Restore Product | Medium | 3 |
| NE-PM-06 | Bulk Import & Export | Low | 5 |
| NE-PM-07 | Role-Based Access Control | High | 3 |

---

---

## NE-PM-01 — Product List Page

### Title
Product List Page with Search, Filters, Status Summary and Quick-View

---

### User Story Description
As a **merchandiser or planner**, I want to see all products in a structured, filterable list so that I can quickly locate any product, monitor its status, and navigate to edit, view, or create a production order from it without having to remember product codes.

---

### Acceptance Criteria

**AC-01 — Page Load & Layout**
- The Products list page is accessible from the sidebar under Style Management → Products.
- The page title reads "Product List" with the sub-heading "Manage all products — view, edit and create production orders".
- A "New Product" button in the top-right header bar navigates to the Create Product page.
- An "Import" button and an "Export" button appear in the page header (Import to the left of Export).
- A "View Archived Products" text link appears below the header, right-aligned, and opens the Archive overlay modal.

**AC-02 — Status Summary Bar**
- Three count tiles are displayed above the table: **Draft**, **Active**, and **Hold**.
- Each count reflects the number of records matching the currently applied filters (not the total count of all records).
- Counts update automatically whenever filters change.

**AC-03 — Filter Bar**
- The filter bar contains the following controls:
  - **Search** — text input; filters on product name OR product code (case-insensitive, partial match).
  - **Status** — dropdown: All / Active / Draft / Hold. Default: All.
  - **Category** — dropdown populated from all distinct categories in the tenant's product data. Default: All.
  - **Days Range** — dropdown: Last 7 / 14 / 30 (default) / 90 days / Custom range. Selecting "Custom range" reveals **From** and **To** date pickers.
  - **Clear** button — resets all filters to defaults.
- Filters are applied immediately on change (no submit button required).
- The configured days range preference is persisted across page sessions.

**AC-04 — Table Columns**
The data table displays the following columns in order:

| # | Column | Description |
|---|--------|-------------|
| 1 | # | Row number within the current page |
| 2 | Image | Product image thumbnail (48×48px). Clicking opens a fullscreen overlay. If no image, a placeholder icon is shown. |
| 3 | Product Name | Clickable — opens the quick-view modal |
| 4 | Product Code | Auto-generated code (e.g. PROD-2026-0001); monospace font |
| 5 | Category | Product category |
| 6 | Colors | Small colour swatch chips, one per colour. Tooltip shows colour name on hover. |
| 7 | Creation Date | Formatted as DD Mon YYYY |
| 8 | Status | Status pill — Draft (grey) / Active (green) / Hold (yellow) |
| 9 | Created By | Name of the user who created the record |
| 10 | Action | **Edit** button (first), **View** button (second), **Delete/Archive** icon button (third) |

**AC-05 — Action Buttons**
- **Edit** button navigates to the Edit Product page for that record.
- **View** button navigates to the View Product page for that record.
- **Delete** icon triggers a confirmation: "Archive product '...'? This will move it to archive and it will no longer appear in active lists." Confirming sets status to ARCHIVED.

**AC-06 — Quick-View Modal**
- Clicking a product name in the table opens a modal showing: Product Name, Product Code, Status pill, Category, UOM, Brand, MRP, MOQ, Colors, Sizes, Created By, Created Date, and product image (if present).
- The modal footer contains two buttons: **Edit** (navigates to Edit page) and **Full View** (navigates to View page).
- The modal closes when the user clicks the × button or clicks outside the modal.

**AC-07 — Pagination**
- The table paginates at 10 records per page by default.
- Pagination controls show page number buttons with previous/next arrows.
- A count label shows "Showing X–Y of Z products".
- Archived products are excluded from all counts and the table.

**AC-08 — Image Fullscreen**
- Clicking a product image thumbnail (in the table or in the quick-view modal) opens a fullscreen dark overlay with the full image and a close (×) button.

---

### Priority
**High** — This is the entry point to the entire Product module. No other product feature is usable without it.

---

### Story Points / Effort Estimate
**5 points**
- Backend: 1.5 days (paginated list API with multi-filter query, archived exclusion, archive action endpoint)
- Frontend: 1.5 days (table, filter bar, status pills, quick-view modal, image fullscreen)
- Testing: 0.5 days

---

### Assumptions / Notes
- Archived products are shown only via the "View Archived Products" overlay modal, not in the main table.
- The Category dropdown is populated dynamically from existing product data (distinct values), not from a separate category master table.
- Status pill colours match the design system: Draft = grey, Active = green, Hold = yellow.
- Colour swatches use the `color_hex` value stored on the product record at time of save (denormalised snapshot).
- The "Delete" action archives (soft-delete) the product — no hard delete exists.

---

### Definition of Done
- [ ] Product list API (`GET /cedge/api/v1/products`) implemented with pagination, search, status, category, and date range filters; ARCHIVED records excluded by default.
- [ ] Archive action API (`PUT /cedge/api/v1/products/{id}/status` with `status: ARCHIVED`) implemented.
- [ ] All table columns, status summary counts, and filter controls render and function correctly.
- [ ] Quick-view modal opens, displays correct data, and navigates correctly from its buttons.
- [ ] Image thumbnail and fullscreen overlay work.
- [ ] "View Archived Products" link opens the archive modal listing archived products.
- [ ] Unit tests written for list service method (all filter combinations, edge cases: no results, exactly 10 results, page boundary).
- [ ] Swagger annotations complete on all endpoints.
- [ ] Postman collection updated with list, filter, and archive requests.
- [ ] Tested in browser: all filters, pagination, modal open/close, fullscreen image.

---
---

## NE-PM-02 — Create Product

### Title
Create Product with Multi-Colour Expansion, BOM Allocation, Image Upload, and Auto-Code Generation

---

### User Story Description
As a **merchandiser**, I want to create a product with all its specifications — including colour, sizes, Bill of Materials, pricing, and attachments — so that the product is fully defined, available to downstream teams for production planning, and can be immediately used to raise a production order.

---

### Acceptance Criteria

**AC-01 — Page Access & Navigation**
- The Create Product page is accessible from the "New Product" button on the list page.
- Breadcrumb shows: Style Management > Products > New Product.
- A **Cancel** button in the action bar returns the user to the Product List without saving.

**AC-02 — Section 1: Product Information**

*Product Name*
- Text input, maximum 100 characters.
- Mandatory in validation logic — if left blank at save time, the system auto-generates a unique product code in the format `PROD-{YEAR}-{SEQUENCE}` (e.g. `PROD-2026-0001`) and uses it as the display name.
- The character counter is shown below the field.

*Color (mandatory)*
- Multi-select dropdown mapped from the Color Master (`GET /cedge/api/v1/masters/color`).
- Each option in the dropdown shows a colour swatch (hex-based) and the colour name.
- Selected colours appear as removable chip tags in the trigger area.
- A "+ Create New Color" option at the bottom of the dropdown opens an inline create panel with fields: Color Name (text) and Color Hex (colour picker). Saving adds the new colour to both the master and the current selection.
- An information banner reads: "Selecting multiple colours will create separate products per colour — all other data remains the same."
- At least one colour must be selected before saving (server-side validation).

*Customer Product Name*
- First entry auto-fills with the value typed in the Product Name field (updates on input).
- The user may edit this auto-filled value.
- A "+ Add another customer name" button adds a new text input row below.
- Each additional row has a delete (×) button. The delete button on the first row is hidden while it is the only row.
- Maximum 100 characters per entry.

*Product Description*
- Multi-entry field — same component pattern as Customer Product Name.
- "+ Add description line" button adds new rows.
- Maximum 200 characters per entry.
- Not mandatory.

*Product Category*
- Dropdown. Not mandatory.
- A "+" button opens an inline create panel for a new category name. Saved categories appear in the dropdown immediately.
- The category list is shared with the Product Type field on the Sample page.

*Brand*
- Dropdown mapped from Brand Master. Not mandatory.
- A "+" button opens an inline create panel. New brands are saved to the brand master immediately.

*Product Image*
- Drag-and-drop or click-to-upload area.
- A "From Sample" link opens a sample selection modal. Selecting a sample imports that sample's image into this field.
- On file selection, the image is uploaded immediately to storage and a preview is shown inline.
- The preview is clickable and opens a fullscreen overlay.
- A "✕ Remove" button on the preview removes the image.
- Accepted formats: PNG, JPG, JPEG.

**AC-03 — Section 2: Size & Quantity**

*Sizes*
- Multi-select dropdown from Size Master. Not mandatory.
- A "+" button opens an inline create panel (Size Code + Size Name fields).
- When sizes are selected, a table appears below the dropdown — one row per selected size — with:
  - Size badge (code label)
  - Quantity input (numerical, optional)
  - Delete row button
- Removing a size from the dropdown also removes its table row.

*MOQ (Minimum Order Quantity)*
- Single numerical input field displayed below the size-quantity table.
- Applies to the product overall, not per size.
- Not mandatory.

**AC-04 — Section 3: Pricing & Compliance**

| Field | Type | Rule |
|-------|------|------|
| UOM | Dropdown (UOM Master) + inline create | Mandatory |
| HSN Code | Numerical text | Optional |
| GST (%) | Decimal | Optional; maximum value 100; server rejects values > 100 |
| Standard Cost | Decimal with ₹ prefix | Optional |
| MRP | Decimal with ₹ prefix | Optional |

The "+ Create New UOM" inline panel (UOM Code + UOM Name) saves to the UOM Master immediately.

**AC-05 — Section 4: BOM Allocation**

- Two radio options: **Use Existing Materials** (default selected) | **Add New Materials**.

*Existing Materials mode:*
- Table with columns: #, Material Type (dropdown), Material Name (dropdown, filtered by selected Material Type), Consumption (decimal), UOM (dropdown from UOM Master), Delete.
- Material Type options include all RM types including Fabric.
- Selecting "Fabric" as Material Type filters Material Name to show only fabrics from the fabric master. Same logic applies to other RM types.
- An "+ Import from Sample" link opens a sample selection modal. Selecting a sample populates the BOM table with that sample's BOM data (user may then edit).
- A "+ Add Row" button adds a new blank row at the bottom of the table.

*New Materials mode:*
- Same table structure but Material Name becomes a free-text input (varchar) and Material Name shows colour in parentheses (e.g. "Cotton Twill (Blue)").
- Consumption: decimal input. UOM: dropdown from UOM Master.
- "+ Add Row" adds a new row.

- BOM is not mandatory. It may also be allocated or completed in the Edit page.

**AC-06 — Section 5: Remarks**
- Textarea, 500 character maximum. Not mandatory.

**AC-07 — Section 6: Attachments**
- Single multi-file drop zone.
- Accepted file types: PNG, JPG, JPEG, PDF, DOC, DOCX, XLS, XLSX, CSV, TXT, ZIP, PPT, PPTX.
- Multiple files can be selected or dragged in one go.
- Each uploaded file appears in a list showing: file icon, file name, file size, Download button, Delete button.
- Download opens the file. Delete removes it from the list (and from the server on save).

**AC-08 — Action Bar**
- **Cancel** — returns to the list without saving.
- **Save as Draft** — saves the product with status = DRAFT.
- **Save Product** — saves the product with status = ACTIVE.
- No status dropdown is shown on the Create page (status is controlled solely by button choice).

**AC-09 — Multi-Colour Expansion**
- When the user selects more than one colour and clicks Save, a confirmation modal appears: "You have selected {N} colours ({list}). Saving will create {N} separate products with the same details — one per colour."
- The modal has a **Confirm** and a **Cancel** button.
- On confirm, the backend creates N product records (one per colour). All records share the same product name, customer names, descriptions, category, brand, sizes, BOM, pricing, remarks, and attachments. Each gets its own product code, color_id, color_name, and color_hex.
- The success toast shows the count: "2 products created successfully" and lists each code.

**AC-10 — Auto-Code Generation**
- If Product Name is blank at save time, a code is auto-generated in the format `PROD-{YEAR}-{4-digit-sequence}`.
- The sequence is tenant-scoped and monotonically incrementing.
- The generated code is displayed in the list page and view page as the product identifier.

**AC-11 — Validation**
- UOM is mandatory; saving without it highlights the field in red and shows "UOM is required."
- At least one colour must be selected; saving without one highlights the colour field and shows "At least one colour is required."
- GST > 100 is clamped to 100 on input (client-side); server also rejects values above 100.
- All validation errors are shown inline beside the relevant field.
- A toast notification confirms success or reports failure.

---

### Priority
**High** — Core data entry screen. No product exists in the system until this is built.

---

### Story Points / Effort Estimate
**13 points**
- Backend: 3 days (create API with multi-colour expansion, auto-code generation, file upload wiring, BOM save, all child table inserts, validation)
- Frontend: 3.5 days (6 sections, multi-select dropdowns with inline create panels, BOM table with two modes, file upload with preview, multi-colour confirmation modal, form validation)
- Testing: 1 day

---

### Assumptions / Notes
- Image and attachment files are uploaded to cloud storage immediately when the user selects them (before form submission). The form submission sends only the stored file reference value, not the binary.
- The "From Sample" import is read-only from the sample — it does not link the product to the sample permanently; it is a one-time data copy.
- Categories created inline on the Create Product page are immediately available in all product forms and on the Sample page's Product Type field.
- "Save as Draft" and "Save Product" both call the same create API endpoint; only the `status` field value differs (`DRAFT` vs `ACTIVE`).
- If a user navigates away mid-form, no draft is auto-saved; the user must explicitly click "Save as Draft".
- New colour, UOM, brand, size, and category created via inline panels are persisted to their respective master tables immediately on clicking Save inside the panel, regardless of whether the main product form is eventually saved.

---

### Definition of Done
- [ ] `POST /cedge/api/v1/products` implemented with all validations, multi-colour expansion, auto-code generation, and all child table inserts (customer names, descriptions, sizes, BOM, attachments).
- [ ] Image upload flow: upload endpoint accepts file, returns stored value; stored value saved to `products.product_image`.
- [ ] Attachment upload flow: upload endpoint accepts files, stored values saved to `product_attachments`.
- [ ] Inline create endpoints working for Color, UOM, Brand, Size, Category masters.
- [ ] BOM "Import from Sample" modal fetches sample BOM and populates rows.
- [ ] Multi-colour confirmation modal appears when N > 1 colours selected.
- [ ] All form sections render and submit correctly including the two BOM modes.
- [ ] Validation errors shown inline; toast confirms success.
- [ ] Unit tests for: single-colour create, multi-colour create (3 colours → 3 records), blank name auto-generates code, GST > 100 rejected, missing UOM rejected.
- [ ] Integration test: full create → verify all child tables populated correctly.
- [ ] Swagger annotations complete.
- [ ] Postman collection updated.

---
---

## NE-PM-03 — Edit Product

### Title
Edit Product with Status Management and Create Production Order Shortcut

---

### User Story Description
As a **merchandiser or production planner**, I want to edit any field of an existing product — including its status and BOM — so that I can keep the product specification current and control whether it is available for production.

---

### Acceptance Criteria

**AC-01 — Page Load**
- The Edit page loads pre-populated with all current product data retrieved from the API.
- The page header shows "Edit: {Product Name}" and the product code beneath it.
- An "Edit Mode" badge is displayed next to the title.
- A meta strip below the page header shows: Created by, Created on, Last modified by, Last modified date.
- Two navigation buttons in the header: **View** (navigates to View page for the same product) and **Back to List**.

**AC-02 — Editable Fields**
- All fields from the Create page are editable: Product Name, Colour (single-select on edit — no multi-colour expansion), Customer Product Names (multi-entry), Product Description (multi-entry), Category, Brand, Sizes with quantities, UOM, HSN, GST, Standard Cost, MRP, MOQ, BOM rows, Remarks, Attachments, and Product Image.
- Colour is a **single-select** on the Edit page (multi-colour expansion is a create-only behaviour).
- Existing attached files are shown in the attachments list with Download and Delete options. New files can be added via the drop zone.
- Existing BOM rows are editable inline. New rows can be added with the "+" button. Rows can be deleted.

**AC-03 — Status Dropdown**
- The action bar contains a **Status** dropdown with options: Draft, Active, Hold, Archive.
- The status dropdown is visible **only** on the Edit page, not on the Create page.
- Selecting "Archive" and saving moves the product to archived status and redirects to the Product List.
- Valid status transitions are enforced by the server (e.g. an ARCHIVED product cannot be edited directly — only restored via the Archive modal).

**AC-04 — Action Bar**
- **Cancel** — returns to the product list without saving.
- **Update Product** (primary brand-colour button) — saves all changes; shows success toast; redirects to the View page.
- **Create Production Order** (blue `#2563eb` button) — navigates to the Production Order create page with the current product pre-selected and its details pre-populated. This button is always visible regardless of status.

**AC-05 — BOM Edit**
- The full BOM section (same as Create page) is available on Edit — Existing / New radio, material type, material name, consumption, UOM, add row, delete row.
- This allows users who skipped BOM on creation to complete it later.
- "Import from Sample" shortcut is also available on Edit.

**AC-06 — Image Replacement**
- The current product image is shown if one exists.
- The user may drag-and-drop or click-to-upload a new image to replace it.
- Clicking the existing image opens a fullscreen preview.
- The "✕ Remove" button removes the current image without replacing.

**AC-07 — Validation**
- Same validation rules as Create (UOM mandatory, Colour required, GST ≤ 100).
- Inline field-level error messages shown on failed submit.
- If no changes have been made, the Update Product button is still active (no dirty-state blocking).

**AC-08 — Unsaved Changes Warning**
- If the user clicks Cancel or navigates away with unsaved changes, a browser confirmation appears: "You have unsaved changes. Leave without saving?"

---

### Priority
**High** — BOM allocation, status changes (Hold, Archive), and image updates all depend on this screen being available.

---

### Story Points / Effort Estimate
**8 points**
- Backend: 2 days (GET product detail, PUT update with all child table replace-and-reinsert, status transition validation, file update logic)
- Frontend: 2 days (load and populate all form sections, status dropdown, Create Production Order button, unsaved-changes guard)
- Testing: 0.5 days

---

### Assumptions / Notes
- On update, all child rows (customer names, descriptions, sizes, BOM lines) are replaced in full — the client sends the complete current state, and the server deletes and re-inserts.
- For attachments, the client sends back the IDs of kept attachments alongside any new stored values. The server deletes rows whose IDs are absent from the request and inserts new rows.
- Changing colour on Edit does NOT trigger multi-colour expansion — it simply changes the colour of the existing product record.
- Only ADMIN users can change status. USER-role users can edit product fields but the status dropdown is hidden for them (enforced server-side via permission check).
- The "Create Production Order" button navigates to `/production-orders/create?productId={id}`. The Production Order page is responsible for consuming the product data; no additional API is needed on the Products side.
- Archived products redirect to the list page if accessed directly via URL — they cannot be edited without being restored first.

---

### Definition of Done
- [ ] `GET /cedge/api/v1/products/{id}` returns full product detail including all child lists (customer names, descriptions, sizes, BOM, attachments with download URLs).
- [ ] `PUT /cedge/api/v1/products/{id}` implemented with full child-table replace logic, status transition validation, updated_by / updated_at tracking.
- [ ] `PUT /cedge/api/v1/products/{id}/status` implemented separately for status-only updates.
- [ ] Edit page loads pre-populated from the API; all fields editable; meta strip shows audit data.
- [ ] Status dropdown (Draft / Active / Hold / Archive) visible on edit; hidden on create.
- [ ] Create Production Order button visible and navigates correctly.
- [ ] Unsaved-changes browser confirmation works on Cancel and back-navigation.
- [ ] Unit tests: update happy path, invalid status transition (e.g. ARCHIVED → HOLD) returns 422, missing UOM returns 400.
- [ ] Swagger annotations complete.
- [ ] Postman request for update added.

---
---

## NE-PM-04 — View Product

### Title
Read-Only Product View with Print and Create Production Order

---

### User Story Description
As a **production planner or quality reviewer**, I want to view the complete specification of a product on a read-only page so that I can review all details — including BOM, sizing, pricing, and attachments — and raise a production order directly without risk of accidentally modifying the product data.

---

### Acceptance Criteria

**AC-01 — Page Layout**
- The View page is accessible from the Edit button on the list, the Full View button in the quick-view modal, and from the Edit page's View button.
- The page header shows: Product Name, Product Code (sub-heading), current Status pill, and a "View Only" badge.
- A meta strip shows: Created by, Created on, Last modified by, Last modified.

**AC-02 — Section 1: Product Information (Read-Only)**
All fields displayed in labelled read-only format:
- Product Name and Product Code side by side.
- Colors: shown as swatch chips with colour name tooltip.
- Customer Product Name(s): listed one per line.
- Product Description: displayed as formatted text (preserving line breaks).
- Product Category and Brand side by side.
- Product Image: shown full-width on the right column; clicking opens fullscreen overlay. Placeholder icon shown if no image.

**AC-03 — Section 2: Size & Quantity (Read-Only)**
- Sizes displayed as a table: Size badge | Quantity.
- Header shows total size count.
- If no sizes, shows "No sizes selected" in muted italic text.

**AC-04 — Section 3: Pricing & Compliance (Read-Only)**
- UOM, HSN Code, and GST displayed as labelled fields in a 3-column grid.
- Standard Cost, MRP, and MOQ displayed as three pricing cards with labels, values (with ₹ prefix where applicable), and subtitles ("Manufacturing cost", "Max. retail price", "Minimum order quantity").

**AC-05 — Section 4: BOM (Read-Only)**
- BOM lines displayed as a read-only table: Material Type (with colour-coded badge: Fabric = blue, Thread = green, Button = orange, Other = grey) | Material Name | Consumption | UOM.
- Header shows total BOM line count.
- If no BOM, shows "No BOM data allocated" in muted italic text.

**AC-06 — Section 5: Remarks (Read-Only)**
- Remarks displayed as pre-formatted text.
- If empty, shows "No remarks".

**AC-07 — Section 6: Attachments (Read-Only)**
- Each attachment listed with: file icon, file name, file size, **Download** button (downloads the file), **View** button (opens file in new browser tab).
- Header shows attachment count.
- If no attachments, shows "No attachments".

**AC-08 — Action Bar (Top and Bottom)**
Both the top page header and the bottom action bar contain:
- **Print** — triggers browser print (`window.print()`).
- **Edit** — navigates to the Edit page for this product.
- **Create Production Order** (green `.btn-success`) — navigates to the Production Order create page with the current product pre-selected and details pre-populated.
- **Back to List** — navigates to the Product List.

---

### Priority
**Medium** — Required for read-only reviewers (USER role) and as the landing page after a successful save/update.

---

### Story Points / Effort Estimate
**3 points**
- Backend: 0.5 days (reuses the same `GET /products/{id}` endpoint as Edit; no new API needed)
- Frontend: 1.5 days (read-only display components, pricing cards, BOM badge colours, attachment Download/View buttons, print layout)
- Testing: 0.5 days

---

### Assumptions / Notes
- The View page reuses `GET /cedge/api/v1/products/{id}` — no separate view-specific API endpoint is needed.
- "Create Production Order" on the View page navigates to `/production-orders/create?productId={id}` (same as on Edit page).
- The Print action uses `window.print()`. Print-specific CSS (hiding sidebar, header, action bar) should be included in the page stylesheet so that only the product detail sections are printed.
- Users with only VIEW permission cannot see the Edit button (permission-gated in frontend).
- Download and View actions for attachments use the CloudFront URL constructed server-side.

---

### Definition of Done
- [ ] View page renders all 6 sections in read-only format using data from `GET /products/{id}`.
- [ ] Pricing cards (Standard Cost, MRP, MOQ) display correctly with ₹ prefix and subtitles.
- [ ] BOM table shows colour-coded material type badges.
- [ ] Each attachment row has a working Download and View button.
- [ ] Product image fullscreen overlay works on click.
- [ ] Print produces a clean output (sidebar, topbar, and action bar hidden via `@media print`).
- [ ] "Create Production Order" button navigates with correct `productId` query param.
- [ ] Status pill and "View Only" badge visible in page header.
- [ ] Tested with: a product that has all fields populated, a product with no image/BOM/attachments (empty states shown), and a product with multiple customer names and description lines.

---
---

## NE-PM-05 — Archive & Restore Product

### Title
Archive Product from List and Restore from Archive Modal

---

### User Story Description
As an **admin**, I want to archive products that are no longer in active use so that they are hidden from the main list without being permanently deleted, and I want to be able to restore any archived product back to Draft status when needed.

---

### Acceptance Criteria

**AC-01 — Archive from List Page**
- Clicking the Delete icon in a product row on the list page shows a confirmation: "Archive product '{name}'? This will move it to archive and it will no longer appear in active lists."
- Confirming archives the product (status = ARCHIVED) and removes it from the list immediately (optimistic or post-refetch).
- A success toast confirms: "'{name}' archived".

**AC-02 — Archive via Edit Page**
- On the Edit page, selecting "Archive" from the Status dropdown and clicking Update Product archives the product.
- After archiving, the user is redirected to the Product List page with a confirmation toast.

**AC-03 — Archived Products Are Hidden**
- Archived products do NOT appear in the main Product List.
- The Status filter on the list page does NOT include "Archived" — archived products are only accessible via the Archive modal.
- Archived products are excluded from all status counts (Draft, Active, Hold).

**AC-04 — View Archived Products Modal**
- The "View Archived Products" link below the list page header opens an overlay modal.
- The modal displays a table with columns: Product Name | Product Code | Category | Archived On | Restore button.
- An empty state message is shown if there are no archived products.

**AC-05 — Restore from Archive Modal**
- Each row in the archive modal has a **Restore** button (green-tinted).
- Clicking Restore sets the product status to DRAFT and removes it from the archive modal immediately.
- A success toast confirms: "'{name}' restored to active products".
- The restored product appears in the main list on the next load/filter.

**AC-06 — Archived Products Cannot Be Used for Production Orders**
- An archived product cannot be selected when creating a Production Order.
- The Production Order module validates that the selected productId belongs to a non-ARCHIVED product.

---

### Priority
**Medium** — Needed to maintain a clean product list without destroying data.

---

### Story Points / Effort Estimate
**3 points**
- Backend: 0.5 days (`PUT /products/{id}/status` endpoint, `GET /products/archived` endpoint, Production Order validation)
- Frontend: 1 day (archive confirmation, Archive modal with table and Restore button, toast feedback)
- Testing: 0.5 days

---

### Assumptions / Notes
- Archive is a soft-delete — the product record is never physically removed from the database. Status is set to `ARCHIVED`.
- Restoring always sets status to DRAFT (not back to the previous status), so the user can review and re-activate manually.
- Only ADMIN-role users can archive or restore products. USER-role users do not see the Delete icon on the list.
- No limit on how many products can be archived.

---

### Definition of Done
- [ ] `PUT /cedge/api/v1/products/{id}/status` handles ARCHIVED transition from DRAFT, ACTIVE, and HOLD.
- [ ] `GET /cedge/api/v1/products/archived` returns archived products with pagination and search.
- [ ] Default list endpoint (`GET /products`) excludes ARCHIVED records in all circumstances.
- [ ] Archive confirmation dialog shows on Delete icon click.
- [ ] Archive modal renders archived products table with Restore button.
- [ ] Restore sets status = DRAFT and removes item from modal immediately.
- [ ] Toast shown on archive and on restore.
- [ ] Unit tests: archive from each valid status, restore, verify list excludes archived.

---
---

## NE-PM-06 — Bulk Import & Export

### Title
Import Products from Excel/CSV and Export Product List

---

### User Story Description
As an **admin or merchandiser**, I want to import multiple product records from a spreadsheet at once so that I can onboard a large product catalogue quickly, and I want to export the current filtered list so that I can share product data with other teams or audit it offline.

---

### Acceptance Criteria

**AC-01 — Import Button**
- An "Import" button appears in the list page header (to the left of Export).
- Clicking opens the Import modal.

**AC-02 — Import Modal**
- The modal contains an icon, a title "Upload Excel / CSV", and a brief instruction.
- A file drop zone accepts `.xlsx`, `.xls`, or `.csv` files (single file per import).
- Once a file is selected, the file name is shown and the Import submit button becomes active.
- A **Download Template** button allows the user to download a blank template file with the correct column headers.
- **Cancel** closes the modal without importing.
- **Import** submits the file. On success, the modal closes and a toast shows the result: "Importing '{filename}' — products will appear shortly."

**AC-03 — Import Processing**
- The server parses each row and validates: Product Name (max 100 chars), UOM (must match an existing UOM code), GST ≤ 100, MOQ > 0 if provided.
- Rows that pass validation are created as new products.
- Rows that fail validation are skipped. The API response includes a list of row-level errors with row number, field name, and error message.
- Partial success is allowed — valid rows are saved even if some rows fail.
- Duplicate product names within the import file do not cause errors — each row creates a separate product record.

**AC-04 — Export**
- The "Export" button in the list page header triggers a file download.
- The export respects the currently applied filters (search, status, category, date range).
- The exported file is `.xlsx` format.
- Exported columns match the list page columns: Product Code, Product Name, Colour, Category, Brand, Sizes, UOM, GST%, Standard Cost, MRP, MOQ, Status, Created By, Created Date.

---

### Priority
**Low** — Useful for bulk onboarding but not blocking for initial launch.

---

### Story Points / Effort Estimate
**5 points**
- Backend: 2 days (import parsing, row-level validation, batch insert, export with filter support, template generation)
- Frontend: 1 day (Import modal with file upload UI, Download Template button, Export button handler)
- Testing: 0.5 days

---

### Assumptions / Notes
- Import does not support updating existing products — it is create-only. If a product with the same name already exists, a new record is still created.
- Import file size limit: 5 MB.
- Import is restricted to ADMIN role users only. Export is available to both ADMIN and USER.
- The import template will be a server-generated `.xlsx` file with headers and one example row.
- Large imports (>500 rows) may be processed asynchronously — the API returns a job ID and the frontend polls for completion (implementation detail for the backend team to decide).

---

### Definition of Done
- [ ] `POST /cedge/api/v1/products/import` accepts `.xlsx`/`.csv`, validates rows, creates valid records, returns success + error summary.
- [ ] `GET /cedge/api/v1/products/import/template` returns a downloadable `.xlsx` template.
- [ ] `GET /cedge/api/v1/products/export` returns `.xlsx` file respecting current filters.
- [ ] Import modal opens, accepts file, shows file name, enables submit button on selection, shows result toast.
- [ ] Download Template button downloads the file.
- [ ] Export button triggers file download.
- [ ] Unit tests: valid rows imported, invalid rows skipped with correct error messages, missing UOM returns row-level error.

---
---

## NE-PM-07 — Role-Based Access Control

### Title
RBAC — Separate Create, Edit, and View Permissions for Products

---

### User Story Description
As an **administrator**, I want to control which users can create, edit, and view products — separately — so that read-only users can browse product data without risk of modification, while only authorised users can make changes.

---

### Acceptance Criteria

**AC-01 — Page-Level Permission Matrix**

| Action | ADMIN | USER |
|--------|-------|------|
| View product list | ✅ | ✅ |
| View product detail (View page) | ✅ | ✅ |
| Create product | ✅ | ❌ |
| Edit product fields | ✅ | ❌ |
| Change product status | ✅ | ❌ |
| Archive / restore product | ✅ | ❌ |
| Import products | ✅ | ❌ |
| Export product list | ✅ | ✅ |

**AC-02 — UI-Level Enforcement**
- The "New Product" button in the list page header is hidden for USER-role users.
- The "Edit" action button in the table is hidden for USER-role users.
- The "Delete/Archive" action button in the table is hidden for USER-role users.
- The Status dropdown on the Edit page is hidden for USER-role users.
- The "Import" button is hidden for USER-role users.
- If a USER-role user attempts to navigate directly to the Create or Edit URL, they are redirected to the list page with an "Access denied" toast.

**AC-03 — API-Level Enforcement**
- All write endpoints (`POST /products`, `PUT /products/{id}`, `PUT /products/{id}/status`, `POST /products/import`) return `HTTP 403 Forbidden` if called by a USER-role token.
- Read endpoints (`GET /products`, `GET /products/{id}`, `GET /products/archived`, `GET /products/export`) are accessible to both roles.

**AC-04 — Sidebar Navigation**
- The Products link appears in the sidebar for both ADMIN and USER roles.
- It is shown only after the `public.module_wise_links` row for Products has been seeded and the role has at least `canView` permission assigned.

**AC-05 — Separate Page Access Flags**
- The `canCreate`, `canEdit`, and `canView` flags in the `role_module_menu` table are independently configurable, allowing fine-grained access control (e.g. a user who can view and edit but not create).
- The frontend reads these flags from the menu API response and conditionally shows/hides the New Product button, Edit buttons, and the Create page route.

---

### Priority
**High** — Access control must be in place before the module goes live to prevent unauthorised data modification.

---

### Story Points / Effort Estimate
**3 points**
- Backend: 1 day (endpoint security annotations, module_wise_links seed migration, permission seeding)
- Frontend: 0.5 days (conditional rendering of buttons and routes based on permission flags)
- Testing: 0.5 days

---

### Assumptions / Notes
- The Style Management module (`public.modules` row) is assumed to already exist from previous work. Only the Products link row needs to be added to `public.module_wise_links`.
- Default seeding via `TenantDefaultDataService`: ADMIN gets all permissions; USER gets canList + canView only.
- SUPER_ADMIN has implicit access to all endpoints and bypasses the `role_module_menu` check.
- Custom roles (if enabled for a tenant) can be granted any combination of the five flags through the Permissions management screen.

---

### Definition of Done
- [ ] `public.module_wise_links` row seeded for Products (link_code: `STYLE_PRODUCT`, link_url: `/style/products`).
- [ ] `TenantDefaultDataService` seeds `role_module_menu` rows for ADMIN (full) and USER (canList + canView) for the Products link.
- [ ] All write API endpoints reject USER-role tokens with HTTP 403.
- [ ] Frontend hides New Product, Edit, Delete/Archive, Import buttons and the status dropdown for USER-role sessions.
- [ ] Navigating directly to `/style/products/create` or `/style/products/{id}/edit` as a USER-role user redirects to the list page.
- [ ] Integration test: USER-role token calling `POST /products` returns 403.
- [ ] Tested in browser with both ADMIN and USER accounts.

---

---

## Cross-Cutting Notes (Applicable to All Stories)

### Audit Fields
Every product create and update must record:
- `created_by` (user ID from JWT) and `created_at` (server-side UTC timestamp)
- `updated_by` and `updated_at` on every edit

These are displayed on the Edit and View pages in the meta strip: Created by, Created on, Last modified by, Last modified.

### Toast Notifications
All create, update, archive, restore, import, and export actions must show a toast notification:
- **Success** — green, with a checkmark icon. Auto-dismisses after 3.5 seconds.
- **Error** — red, with an error icon. Auto-dismisses after 3.5 seconds.
- **Info** — brand colour, with an info icon. Used for "processing" messages.

### Production Order Integration
Both the Edit and View pages show a "Create Production Order" button that navigates to the Production Order create page with `?productId={id}` in the URL. The Production Order page is responsible for calling `GET /products/{id}` to pre-fill its form. No additional API changes are required on the Product side for this integration.

### File Upload Standard
All product images and attachments follow the storage convention:
- S3 key: `{tenantCode}/STYLE_PRODUCT/{timestamp}!_!{sanitisedFileName}`
- DB column stores only: `{timestamp}!_!{sanitisedFileName}`
- CloudFront URL is constructed at read time — never stored in the database.

### Status Lifecycle

```
DRAFT ──► ACTIVE ──► HOLD
  ▲           │        │
  │           ▼        ▼
  └────── ARCHIVED ◄───┘
  (restore sets back to DRAFT)
```
