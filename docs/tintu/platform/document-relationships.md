# Document Relationship Engine

> **Simplify. Manage. Grow.**

Documents convert into other documents, often **partially** and **many-to-many**. Relationships are stored as links, not as a single `parent_id` on the header.

Related: [Document Engine](document-engine.md) · [Workflow](workflow-engine.md)

---

## 1. Why links

```text
SO-1001  qty 100
   ├── Delivery-1001  qty 40
   ├── Delivery-1002  qty 30
   └── Delivery-1003  qty 30
```

The system must know: ordered 100, delivered 100, remaining 0.

One order line can feed several deliveries. One invoice can bill several deliveries. One delivery can invoice in two bills (rare but allowed if remaining qty exists).

---

## 2. Table `document_links`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `public_id` | UUID UNIQUE | |
| `tenant_id` | BIGINT | |
| `company_id` | BIGINT | |
| `source_document_type` | VARCHAR(50) | e.g. `sales_order` |
| `source_document_id` | BIGINT | Header id |
| `source_line_id` | BIGINT NULL | Line id; null = header-only link |
| `target_document_type` | VARCHAR(50) | e.g. `sales_delivery` |
| `target_document_id` | BIGINT | |
| `target_line_id` | BIGINT NULL | |
| `quantity` | DECIMAL(18,6) | Qty of **source UOM** converted |
| `created_by` | BIGINT | |
| `created_at` | TIMESTAMP | |

Indexes: source header, source line, target header, target line.

Do not delete links when a target is cancelled; mark the target cancelled and exclude cancelled qty from remaining calculations.

---

## 3. Remaining quantity

For a source line:

```text
converted = SUM(document_links.quantity)
            where source_line_id = this line
              and target header status not in (CANCELLED, VOID, DRAFT*)

* Draft targets may optionally reserve remaining qty (company setting).
  Default: DRAFT counts toward remaining so two users cannot over-convert.

remaining = source_line.quantity − converted
```

Over-conversion (`remaining < 0`) is rejected unless the document type allows over-delivery (setting).

Header remaining is the sum of line remaining (or max line remaining for mixed services).

---

## 4. Allowed conversions

| Source | Target |
| ------ | ------ |
| Quotation | Sales order |
| Sales order | Reservation, delivery, invoice (direct bill) |
| Delivery | Sales invoice, sales return |
| Sales invoice | Customer payment, sales return |
| Requisition | Purchase order |
| Purchase order | Purchase receipt, vendor bill (direct) |
| Purchase receipt | Vendor bill, purchase return |
| Vendor bill | Vendor payment, purchase return |
| Stock count | Stock adjustment |

Illegal pairs (quotation → vendor bill) are rejected.

---

## 5. Traceability API

```text
GET /api/v1/documents/{type}/{id}/trace
```

Returns upstream sources and downstream targets with quantities, statuses, and remaining.

UI: a “Document history” panel, not a technical link table.

---

## 6. Example

```text
source SO-1001 line 1  qty 100

link  Delivery-1001 line 1  40
link  Delivery-1002 line 1  30
link  Delivery-1003 line 1  30

ordered   = 100
delivered = 100
remaining = 0  → order line fulfilled
```

Partial:

```text
Ordered = 100
Delivered = 40
Remaining = 60
```

---

## 7. Convert actions

```text
POST /api/v1/sales/orders/{id}/convert-to-delivery
{ "lines": [ { "source_line_id": 1, "quantity": 40 } ], "warehouse_id": … }

POST /api/v1/purchases/orders/{id}/convert-to-receipt
```

Creates the target as DRAFT, writes `document_links`, copies product/qty/price snapshots, runs calculation. User reviews and posts.

AI may call convert only to **draft** targets.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
