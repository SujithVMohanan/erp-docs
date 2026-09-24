# Document Engine

> **Simplify. Manage. Grow.**

Niyanthra business records are **explicit documents**, not rows in one generic voucher table. A sales order is a sales order. A stock transfer is a stock transfer. Shared behaviour (numbering, status, lines, isolation, idempotency, audit) lives in this engine.

Frontend must never show internal type codes, posting flags, or legacy entry ids.

Related: [Workflow](workflow-engine.md) · [Relationships](document-relationships.md) · [Audit](audit.md)

---

## 1. Why not a generic voucher

A single voucher engine (one table, a type id, stock/accounting derived from that id) collapses unrelated lifecycles. Quotations do not post stock. Deliveries do. Invoices post AR. Stock counts do not.

```text
WRONG                         RIGHT
─────                         ─────
voucher (type=??)             sales_quotations
  ├── stock?                    sales_orders
  └── accounts?                 sales_deliveries
                                sales_invoices
                                stock_transfers
                                purchase_orders
                                …
```

Each document type has its own header and line tables, plus a `document_type` value used by workflow, links, audit, and events.

---

## 2. Document types

| Domain | `document_type` | User-facing name |
| ------ | --------------- | ---------------- |
| Sales | `sales_quotation` | Quotation |
| | `sales_order` | Sales Order |
| | `sales_delivery` | Delivery / Shipment |
| | `sales_invoice` | Sales Invoice |
| | `sales_return` | Sales Return |
| | `customer_payment` | Customer Payment |
| Purchase | `purchase_requisition` | Purchase Requisition |
| | `purchase_order` | Purchase Order |
| | `purchase_receipt` | Purchase Receipt |
| | `vendor_bill` | Vendor Bill |
| | `purchase_return` | Purchase Return |
| | `vendor_payment` | Vendor Payment |
| Inventory | `stock_receipt` | Stock Receipt |
| | `stock_issue` | Stock Issue |
| | `stock_transfer` | Stock Transfer |
| | `stock_adjustment` | Stock Adjustment |
| | `stock_reservation` | Stock Reservation |
| | `stock_count` | Stock Count |

Do not expose these codes in normal UI copy. Show the user-facing name and the human document number (`SO-2026-00041`).

---

## 3. Shared header fields

Every tenant document header includes:

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | Internal |
| `public_id` | UUID UNIQUE | API id |
| `tenant_id` | BIGINT | Isolation |
| `organization_id` | BIGINT | From session |
| `company_id` | BIGINT | From session |
| `branch_id` | BIGINT | Working branch |
| `document_type` | VARCHAR(50) | From the table above |
| `document_no` | VARCHAR(40) | Unique per company + type |
| `status` | VARCHAR(40) | Workflow status |
| `doc_date` | DATE | Business date |
| `currency_code` | VARCHAR(10) | Default company currency |
| `notes` | TEXT NULL | |
| `idempotency_key` | VARCHAR(80) NULL UNIQUE per company | See §7 |
| `created_by_type` | VARCHAR(10) | `HUMAN` · `AI` |
| `ai_assistance` | BOOLEAN | True if AI drafted or suggested |
| `ai_action_id` | UUID NULL | Correlation to AI audit |
| `created_by` | BIGINT | Master user |
| `created_at` | TIMESTAMP | |
| `updated_by` | BIGINT NULL | |
| `updated_at` | TIMESTAMP | |
| `submitted_by` | BIGINT NULL | |
| `submitted_at` | TIMESTAMP NULL | |
| `approved_by` | BIGINT NULL | |
| `approved_at` | TIMESTAMP NULL | |
| `rejected_by` | BIGINT NULL | |
| `rejected_at` | TIMESTAMP NULL | |
| `cancelled_by` | BIGINT NULL | |
| `cancelled_at` | TIMESTAMP NULL | |
| `posted_by` | BIGINT NULL | |
| `posted_at` | TIMESTAMP NULL | |
| `deleted_at` | TIMESTAMP NULL | Soft delete |

Server fills tenant/org/company from the authenticated context. The client must not supply a database name or an arbitrary tenant id.

---

## 4. Shared line fields

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `public_id` | UUID UNIQUE | |
| `header_id` | BIGINT FK | Parent document |
| `line_no` | INTEGER | 1-based display order |
| `product_id` | BIGINT NULL | Null for text-only / service lines where allowed |
| `variant_id` | BIGINT NULL | |
| `description` | VARCHAR(255) | Snapshot of name/description |
| `uom_id` | BIGINT | Unit |
| `quantity` | DECIMAL(18,6) | |
| `unit_price` | DECIMAL(18,6) | After pricing engine |
| `price_source` | VARCHAR(80) NULL | Explainable price reason |
| `discount_percent` | DECIMAL(8,4) | |
| `discount_amount` | DECIMAL(18,6) | |
| `tax_amount` | DECIMAL(18,6) | |
| `line_total` | DECIMAL(18,6) | |
| `warehouse_id` | BIGINT NULL | Inventory lines |
| `batch_id` | BIGINT NULL | |
| `serial_ids` | JSONB NULL | If serial-tracked |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

Module tables add domain columns (customer, promised date, reason code). They do not omit the shared set.

---

## 5. Numbering

| Rule | Detail |
| ---- | ------ |
| Scope | Unique per `(company_id, document_type, document_no)` |
| Format | Prefix + year + sequence, e.g. `SO-2026-00041` |
| Prefix | Configurable per company/document type (`SO`, `INV`, `PO`, `TRF`) |
| Allocation | Assigned on first save as DRAFT or on submit — company setting. Default: on first save |
| Gaps | Allowed after cancelled drafts; never reuse a posted number |

---

## 6. Isolation

Every read/write filters by `tenant_id` and `company_id` from the session. Branch-scoped users only see documents for assigned branches unless the permission is company-wide.

```text
Request
  → Auth
  → Company / tenant resolution (Master DB)
  → Permission
  → Document service (tenant DB)
```

---

## 7. Idempotency

Financial and inventory **creates** and **posts** accept header `Idempotency-Key`.

| First request | Second request (same key, same company, same body hash) |
| ------------- | -------------------------------------------------------- |
| Create document | Return the existing document |
| Post movements / journal | Return the existing posting result |

If the key is reused with a **different** body, reject with a conflict error. Do not create a second stock movement or journal.

Store keys on the header (`idempotency_key`) and optionally in `idempotency_records` for non-document posts.

### `idempotency_records`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `company_id` | BIGINT | |
| `key` | VARCHAR(80) | |
| `request_hash` | VARCHAR(64) | |
| `response_document_id` | BIGINT NULL | |
| `document_type` | VARCHAR(50) NULL | |
| `created_at` | TIMESTAMP | |

Unique `(company_id, key)`.

---

## 8. API conventions

Base path: `/api/v1/`. Django REST Framework.

| Concern | Convention |
| ------- | ---------- |
| Pagination | `?page=&page_size=` — envelope `{ count, next, previous, results }` |
| Filtering | Query params: `status`, `branch_id`, `date_from`, `date_to`, `search` |
| Sorting | `?ordering=-doc_date,document_no` |
| Errors | `{ "code", "message", "details": [] }` |
| Auth | Session/JWT + company context |
| Permissions | Resource.action, e.g. `sales.order.create` |
| Idempotency | Header `Idempotency-Key` on POST create/post |

Lifecycle actions are verbs, not generic PATCH of `status`:

```text
POST /api/v1/sales/orders/{id}/submit
POST /api/v1/sales/orders/{id}/approve
POST /api/v1/sales/orders/{id}/cancel
```

---

## 9. Domain events

After a successful commit, publish (do not block the transaction on consumers):

`{DocumentType}Created` · `Submitted` · `Approved` · `Rejected` · `Cancelled` · `Posted`

Examples: `SalesOrderApproved`, `DeliveryPosted`, `StockReceived`, `VendorBillPosted`.

Consumers (later): notifications, search index, AI analysis, forecasting.

---

## 10. Atomic posting

When a document posts inventory and/or accounting:

```text
BEGIN
  lock document
  validate transition
  write lines (already saved) / freeze totals
  write inventory_movements
  write journal (Phase 5)
  update outstanding / balances
  set status POSTED / FULFILLED
COMMIT
```

On any failure, roll back. Never leave stock moved without the document posted, or a journal without stock (when both are required).

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
