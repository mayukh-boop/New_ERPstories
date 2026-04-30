# User Stories: GRN Excess Approval & QC Inspection
*Prepared: 29 Apr 2026 | Reference pages: grn-approval.html, grn-qc.html, po-receipts.html*

---

## Ticket NE-13 — GRN Excess Quantity Approval Workflow

**Summary:** GRN Excess Quantity Approval Workflow

**Story:**
As a Procurement Manager / Approver, I need a dedicated approval workstation to review and act on GRN lines where received quantity exceeds the PO quantity beyond the allowed tolerance, so that excess goods do not enter inventory without authorisation and every over-receipt is audited.

---

### Business Context

When a GRN is submitted and any line has `receivedQty > poQty × (1 + tolerancePct/100)`, the GRN status is set to **EXCESS_PENDING**. The GRN is held from inventory posting until an authorised approver acts.

Two excess severity levels exist:
- **Amber** — within tolerance but over remaining PO balance. Can be posted after review without line-by-line approval.
- **Red** — over tolerance %. Mandatory per-line approver decision required.

---

### API Acceptance Criteria

#### GET /api/grn/pending-approval
Returns paginated list of GRNs in `EXCESS_PENDING` status.

Response per item:
```json
{
  "id": 3,
  "grnNum": "GRN-2026-0003",
  "poNum": "PO-2026-0042",
  "vendor": "Sunrise Fabrics Pvt. Ltd.",
  "submittedBy": "Ravi Kumar",
  "submittedAt": "2026-04-28T09:15:00Z",
  "totalExcessQty": 195,
  "lineCount": 3,
  "daysWaiting": 1
}
```

#### GET /api/grn/:id/approval-detail
Returns full GRN + PO context for the approval workstation detail panel.

Response:
```json
{
  "grnNum": "GRN-2026-0003",
  "poNum": "PO-2026-0042",
  "vendor": "Sunrise Fabrics Pvt. Ltd.",
  "receivedBy": "Ravi Kumar",
  "receivedDate": "2026-04-28",
  "submitterRemarks": "Supplier sent extra, charged same rate",
  "tolerancePct": 5,
  "lines": [
    {
      "id": "l1",
      "item": "Elastic Webbing 25mm",
      "uom": "Metres",
      "poQty": 500,
      "prevReceivedQty": 0,
      "receivedQty": 525,
      "excessQty": 25,
      "excessPct": 5.0,
      "withinTolerance": true
    }
  ],
  "trail": [
    { "actor": "Ravi Kumar", "role": "Store", "action": "submitted", "timestamp": "2026-04-28T09:15:00Z", "remarks": "" }
  ]
}
```

#### POST /api/grn/:id/approve-excess
Body:
```json
{
  "lineDecisions": [
    { "lineId": "l1", "decision": "accept", "allowedQty": 525 },
    { "lineId": "l2", "decision": "accept", "allowedQty": 1050 },
    { "lineId": "l3", "decision": "reject", "allowedQty": 0 }
  ],
  "globalAction": "approve",
  "approverRemarks": "Accepted within budget, line 3 rejected"
}
```

Rules:
- `allowedQty` must be ≤ `receivedQty` for that line.
- `decision: reject` → `allowedQty` must be 0 or omitted.
- `globalAction: approve` → posts `allowedQty` to inventory for each accepted line. GRN status → `APPROVED`.
- `globalAction: reject` → no inventory movement. GRN status → `EXCESS_REJECTED`. Notifies original receiver.
- Appends to approval trail with actor, role, timestamp, decision, remarks.
- Role guard: only `APPROVER` or `ADMIN` role. Returns `403` for others.

#### GET /api/grn/approval-stats
```json
{
  "pending": 4,
  "approvedToday": 2,
  "rejectedToday": 1,
  "totalExcessUnitsPending": 870
}
```

---

### Data Model Notes

```
GRN.status: DRAFT | SUBMITTED | EXCESS_PENDING | APPROVED | EXCESS_REJECTED | POSTED
GRN.tolerancePct: decimal (company-level default overridable per PO)
GrnLine.allowedQty: nullable decimal — set by approver when < receivedQty
ApprovalTrail: grnId, actor, role, action, timestamp, remarks (append-only log)
```

---

### Reference Files
- **UI prototype:** `grn-approval.html` (two-pane workstation — queue left, detail right with per-line Accept/Reject + override qty input, sticky action bar, confirmation modal)
- **GRN entry:** `po-receipts.html` (excess detection logic, two-tier colour coding)

---

---

## Ticket NE-14A — QC Inline Reject at GRN (Scenario A)

**Summary:** QC — Inline Reject at GRN Entry (Accepted Qty to Inventory)

**Story:**
As a Store / Receiving User, when inspecting goods at the point of receipt, I need to mark individual line items as partially rejected directly in the GRN form, so that only accepted quantities enter inventory and rejected quantities are tracked for vendor follow-up without requiring a separate QC step.

---

### Business Context (Scenario A)

Used when:
- The receiving team performs inline inspection at the dock.
- Rejected quantity does not need a separate QC lab process.
- Accepted qty → immediately posted to inventory when GRN is approved.
- Rejected qty → stays in `REJECTED` state, flagged for return-to-vendor.

---

### API Acceptance Criteria

#### GRN Line fields (extended for inline QC)
Each GRN line must support:
```json
{
  "lineId": "l1",
  "item": "Cotton Poplin 44\"",
  "receivedQty": 1000,
  "acceptedQty": 950,
  "rejectedQty": 50,
  "rejectionReason": "Colour variation beyond tolerance",
  "lotNumber": "LOT-2026-0041"
}
```

Rules:
- `acceptedQty + rejectedQty` must equal `receivedQty`. Backend validates; returns 422 if not.
- `rejectedQty > 0` requires `rejectionReason` (non-empty string). Returns 422 if missing.
- `lotNumber` required if `company.lotTracking = true` and `acceptedQty > 0`.

#### Inventory Posting (on GRN Approve)
- `acceptedQty` for each line → posted to inventory with LOT if enabled.
- `rejectedQty` → status `REJECTED`, no inventory movement. Linked to vendor for return note.

#### GET /api/grn/:id (GRN detail — extended)
Returns line-level QC fields: `acceptedQty`, `rejectedQty`, `rejectionReason`, `lotNumber`.

#### Notifications
- If any line has `rejectedQty > 0`, auto-generate a **Vendor Return Request** record (linked to GRN + PO) that procurement can issue to vendor.
- Optionally trigger email/WhatsApp notification to vendor contact.

---

### Data Model Notes

```
GrnLine.acceptedQty: decimal
GrnLine.rejectedQty: decimal
GrnLine.rejectionReason: text (nullable, required when rejectedQty > 0)
GrnLine.lotNumber: varchar (nullable, required when company.lotTracking & acceptedQty > 0)
GrnLine.qcMode: INLINE | QC_HOLD  ← distinguishes Scenario A vs B
```

---

### Reference Files
- **UI prototype:** `po-receipts.html` — GRN entry form, QC toggle per line (Accept/Reject radio + reject qty sub-row + reason input + LOT input)

---

---

## Ticket NE-14B — QC Inspection Page — Full QC Hold Flow (Scenario B)

**Summary:** QC Inspection Workstation — QC Hold → Inspect → Inventory / Vendor Return + Debit Note

**Story:**
As a Quality Control Inspector, I need a dedicated QC Inspection workstation to inspect goods that were received in full into QC Hold (without inline acceptance/rejection at the dock), so that I can record pass, fail, and hold quantities per line, automatically post passed quantities to inventory, and trigger a return-to-vendor workflow with an auto-generated debit note for failed quantities.

---

### Business Context (Scenario B)

Used when:
- Full received quantity is parked in `QC_HOLD` status at GRN (no inline QC).
- A separate QC Inspector reviews the goods in the QC area.
- Inspection result per line: **Pass**, **Fail**, **Hold** (pending re-inspection).
- Passed qty → inventory (with LOT if enabled).
- Failed qty → Vendor Return record + Debit Note against the original invoice.
- Hold qty → remains in QC_HOLD for subsequent re-inspection cycle.

---

### API Acceptance Criteria

#### GET /api/qc/pending
Returns list of GRNs in `QC_HOLD` status.

Response per item:
```json
{
  "id": 101,
  "grnNum": "GRN-2026-0101",
  "poNum": "PO-2026-0042",
  "vendor": "Sunrise Fabrics Pvt. Ltd.",
  "receivedBy": "Ravi Kumar",
  "receivedDate": "2026-04-28",
  "invoiceNum": "INV-SF-2026-1193",
  "invoiceDate": "2026-04-27",
  "totalQty": 2500,
  "lineCount": 3,
  "daysInHold": 1
}
```

#### GET /api/qc/:grnId/detail
Returns GRN header + PO header + all lines with full quantities.

#### GET /api/qc/stats
```json
{
  "inHold": 4,
  "inspectedToday": 2,
  "passedUnitsToday": 1840,
  "failedUnitsToday": 120,
  "debitNotesRaisedToday": 1
}
```

#### POST /api/qc/:grnId/submit
Body:
```json
{
  "lines": [
    {
      "lineId": "l1",
      "passQty": 950,
      "failQty": 50,
      "holdQty": 0,
      "lotNumber": "LOT-2026-0041",
      "failReason": "Colour shade variation"
    },
    {
      "lineId": "l2",
      "passQty": 800,
      "failQty": 0,
      "holdQty": 0,
      "lotNumber": "LOT-2026-0042",
      "failReason": null
    }
  ],
  "inspectorRemarks": "Batch 1 minor colour issue on roll 4"
}
```

Validation rules:
- `passQty + failQty + holdQty` must equal line `receivedQty`. Returns 422 if not.
- `failQty > 0` requires `failReason` (non-empty). Returns 422 if missing.
- `lotNumber` required when `company.lotTracking = true` and `passQty > 0`. Returns 422 if missing.
- All lines must be submitted in one call (atomic). Partial line submission not allowed.

Side effects on success:
1. **passQty** → inventory movement: increase stock for item/warehouse/LOT. Record `InventoryMovement` with type `QC_PASS`, source `QC_INSPECTION_ID`.
2. **failQty** → create `VendorReturn` record: lines with failQty, failReason, linked to original GRN + PO + Invoice.
3. **Debit Note** → auto-generate `DebitNote` against `invoiceNum`: line items = failQty × unitRate from PO. Status `DRAFT` (procurement to review before sending).
4. **holdQty** → line remains in `QC_HOLD`. GRN status stays `QC_HOLD` until all lines have holdQty = 0.
5. If all lines resolved (holdQty = 0 everywhere): GRN status → `QC_COMPLETED`.
6. Append `QcInspectionTrail` entry: inspector, timestamp, action, remarks.

---

### Debit Note Data Model

```
DebitNote:
  id, debitNoteNum (auto-generated DN-YYYY-NNNN), grnId, vendorId, invoiceRef
  status: DRAFT | ISSUED | SETTLED
  lines: [ { itemId, uom, qty, unitRate, amount, failReason } ]
  totalAmount: decimal
  createdAt, issuedAt, settledAt

VendorReturn:
  id, grnId, poId, vendorId, status: PENDING | DISPATCHED | SETTLED
  lines: [ { itemId, uom, returnQty, failReason } ]
  linkedDebitNoteId
```

---

### LOT Tracking (when company.lotTracking = true)

- LOT number entered per line at QC pass stage (not at GRN stage for Scenario B).
- `InventoryMovement` records store `lotNumber`.
- LOT format: `LOT-YYYY-NNNN` (system-generated prefix, inspector can override).
- If `passQty = 0`, LOT field is ignored.

---

### Re-inspection Flow (holdQty > 0)

When a previous QC submission left `holdQty > 0` on any line:
- GRN remains in `QC_HOLD` and appears in the inspection queue.
- Detail panel shows previous pass/fail counts and the remaining holdQty as the new "receivedQty" for re-inspection.
- Inspector submits a new `POST /api/qc/:grnId/submit` for only the held lines.
- Process repeats until holdQty = 0 on all lines.

---

### Reference Files
- **UI prototype:** `grn-qc.html` (two-pane: queue left, inspection entry right with per-line Pass/Fail/Hold inputs, LOT input, fail reason, auto-generated return table and debit note preview, confirmation modal)
- **GRN entry:** `po-receipts.html` (QC Hold mode toggle — when selected, receivedQty goes to QC_HOLD instead of inventory)

---

---

## Summary Table

| Ticket  | Feature | Key Endpoint | GRN Status Flow |
|---------|---------|-------------|-----------------|
| NE-13   | Excess Approval Workstation | POST /api/grn/:id/approve-excess | EXCESS_PENDING → APPROVED \| EXCESS_REJECTED |
| NE-14A  | Inline QC at GRN | GRN line fields: acceptedQty, rejectedQty | SUBMITTED → APPROVED (partial inventory) |
| NE-14B  | QC Hold + Inspection Page | POST /api/qc/:grnId/submit | QC_HOLD → QC_COMPLETED (+ DebitNote + VendorReturn) |

---

*HTML UI files are in the project root of New_ERP1. Share grn-approval.html and grn-qc.html with backend team for full field-level reference.*
