# Workflow / Lifecycle Engine

> **Simplify. Manage. Grow.**

Every business document has an explicit lifecycle. Status changes go through this engine. Modules must not scatter `if status ==` checks that allow illegal transitions.

Related: [Document Engine](document-engine.md)

---

## 1. Purpose

- One definition of allowed statuses per `document_type`
- One function: `transition(document, action, user, reason)`
- Invalid transitions fail before any inventory or accounting work
- Approvals are first-class, not a boolean on the row

---

## 2. Standard fulfilment lifecycle

Used by quotations (subset), orders, deliveries, receipts, transfers, requisitions.

```text
DRAFT
  ↓
SUBMITTED
  ↓
APPROVED
  ↓
PROCESSING
  ↓
PARTIALLY_FULFILLED
  ↓
FULFILLED
  ↓
CLOSED
```

Alternate transitions:

```text
DRAFT → CANCELLED
SUBMITTED → REJECTED
SUBMITTED → CANCELLED
APPROVED → CANCELLED          (if nothing fulfilled)
PROCESSING → CANCELLED        (with compensating stock rules)
REJECTED → DRAFT              (optional rework)
```

`CLOSED` is terminal for operational work (short-close remaining qty). `CANCELLED` is terminal. Neither returns to `DRAFT`.

---

## 3. Invoice / bill / payment lifecycle

```text
DRAFT
  ↓
POSTED
  ↓
PARTIALLY_PAID
  ↓
PAID
```

Also:

```text
DRAFT → CANCELLED
POSTED → VOID                 (reversing journal + stock if required)
```

**Forbidden:** `PAID → DRAFT`, `POSTED → DRAFT`, `VOID → POSTED` without a new document.

---

## 4. Transition table

Store per company (defaults shipped by platform).

### `workflow_definitions`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `company_id` | BIGINT | |
| `document_type` | VARCHAR(50) | |
| `from_status` | VARCHAR(40) | |
| `to_status` | VARCHAR(40) | |
| `action` | VARCHAR(40) | `submit` · `approve` · `reject` · `cancel` · `post` · `fulfill` · `close` · `void` · `pay` |
| `permission_code` | VARCHAR(80) | e.g. `sales.order.approve` |
| `requires_reason` | BOOLEAN | Reject/cancel/void |
| `status` | VARCHAR(30) | `active` |

Unique `(company_id, document_type, from_status, action)`.

Engine algorithm:

```text
load definition(document_type, current_status, action)
if none → reject
if user lacks permission_code → 403
if requires_reason and reason empty → 400
run document-type guards (qty remaining, stock, match)
apply to_status + audit stamps
```

---

## 5. Guards (examples)

| Document | Guard |
| -------- | ----- |
| Any | Cannot edit header/lines in `POSTED`, `PAID`, `CANCELLED`, `VOID`, `CLOSED` except notes allowed by policy |
| Sales order | `approve` requires at least one line with qty > 0 |
| Delivery | `post` requires available stock (or allowed negative) and remaining order qty |
| Invoice | `post` requires totals from calculation engine; cannot `post` twice |
| Vendor bill | Three-way match warnings may block `post` if company setting `block_on_mismatch` |
| Stock issue | `post` requires Available ≥ qty |
| Payment | `pay` cannot exceed outstanding |

Guards are registered per `document_type`, not copied into every view.

---

## 6. Fulfilment status from quantities

For orders, POs, deliveries-from-order:

```text
remaining = ordered − sum(linked target qty)

remaining = ordered     → APPROVED or PROCESSING
0 < remaining < ordered → PARTIALLY_FULFILLED
remaining = 0           → FULFILLED
```

The relationship engine supplies the sums. Workflow applies the status. Do not let the UI set `FULFILLED` directly.

---

## 7. Optional stages

A company may skip stages. That is **configuration**, not missing statuses.

| Mode | Path |
| ---- | ---- |
| Retail sales | Create `sales_invoice` in DRAFT → POST (creates implied delivery/stock out per setting) → PAY |
| B2B sales | Quotation → Order → Delivery → Invoice → Payment |
| Direct purchase | Receipt → Vendor Bill → Payment |
| Controlled purchase | Requisition → Approve → PO → Receipt → Bill → Payment |

Skipped documents are not created. The next document may be entered without a source link. Links are used when a source exists.

---

## 8. API

```text
POST /api/v1/{domain}/{resources}/{id}/submit
POST /api/v1/{domain}/{resources}/{id}/approve
POST /api/v1/{domain}/{resources}/{id}/reject     { "reason": "…" }
POST /api/v1/{domain}/{resources}/{id}/cancel     { "reason": "…" }
POST /api/v1/{domain}/{resources}/{id}/post
POST /api/v1/{domain}/{resources}/{id}/close
```

Body may include `reason`. Response is the document with new `status` and audit stamps.

PATCH of `status` is rejected.

---

## 9. AI

AI may call `create` as DRAFT only (see [AI Architecture](ai-architecture.md)). AI must not call `approve`, `post`, `void`, or `pay` unless a future permission explicitly allows it (default: **denied**).

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
