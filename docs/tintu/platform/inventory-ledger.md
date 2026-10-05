# Inventory Ledger and Reservations

> **Simplify. Manage. Grow.**

`product.stock_qty` is **not** the system of record. Every increase or decrease is an `inventory_movements` row. On-hand and available quantities are derived from movements (and maintained in balance snapshots for speed).

Related: [Document Engine](document-engine.md) · [Inventory module](../inventory/overview.md)

---

## 1. Movement ledger

### `inventory_movements`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `public_id` | UUID UNIQUE | |
| `tenant_id` | BIGINT | |
| `organization_id` | BIGINT | |
| `company_id` | BIGINT | |
| `branch_id` | BIGINT | |
| `warehouse_id` | BIGINT | |
| `product_id` | BIGINT | |
| `variant_id` | BIGINT NULL | |
| `batch_id` | BIGINT NULL | |
| `serial_id` | BIGINT NULL | |
| `transaction_type` | VARCHAR(50) | Document type that posted |
| `transaction_id` | BIGINT | Header |
| `transaction_line_id` | BIGINT NULL | Line |
| `quantity_in` | DECIMAL(18,6) | ≥ 0 |
| `quantity_out` | DECIMAL(18,6) | ≥ 0 |
| `unit_cost` | DECIMAL(18,6) | Valuation cost |
| `total_cost` | DECIMAL(18,6) | |
| `movement_date` | DATE | |
| `created_by` | BIGINT | |
| `created_at` | TIMESTAMP | |

Exactly one of `quantity_in` / `quantity_out` is non-zero per row (except a correction pair as two rows).

A **stock transfer** posts two movements with the same `transaction_id` (`stock_transfer`): source OUT, destination IN.

Never update a posted movement. Reverse with opposite rows tied to a void/return document.

---

## 2. Balance snapshots

Optional table updated in the same DB transaction as the movement (not a nightly job as the only source).

### `stock_balances`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `company_id` | BIGINT | |
| `warehouse_id` | BIGINT | |
| `product_id` | BIGINT | |
| `variant_id` | BIGINT | 0 if none |
| `batch_id` | BIGINT | 0 if none |
| `qty_on_hand` | DECIMAL(18,6) | Sum(in) − Sum(out) |
| `qty_reserved` | DECIMAL(18,6) | From reservation engine |
| `qty_incoming` | DECIMAL(18,6) | Open PO/receipt expected (not yet IN) |
| `qty_outgoing` | DECIMAL(18,6) | Open deliveries not yet OUT |
| `updated_at` | TIMESTAMP | |

Unique `(company_id, warehouse_id, product_id, variant_id, batch_id)`.

```text
Available = On Hand − Reserved
```

Incoming/outgoing are **informational** (supply/demand). They do not reduce Available unless reserved.

---

## 3. Quantity views

| View | Meaning |
| ---- | ------- |
| On Hand | Physically in the warehouse |
| Reserved | Allocated to orders/jobs, not yet issued |
| Available | Can still be promised |
| Incoming | Expected IN from open receipts/POs |
| Outgoing | Expected OUT from open deliveries |

---

## 4. Reservation engine

Reservations are **not** stock OUT. They only change `qty_reserved`.

### `stock_reservations` (header) + `stock_reservation_lines`

See [Inventory workflows](../inventory/workflows.md). Header `document_type = stock_reservation`.

| Event | Effect |
| ----- | ------ |
| Sales order approved (if auto-reserve) | Create/increase reservation |
| Delivery posted | Consume reservation (reserved ↓, then movement OUT) |
| Order cancelled / line reduced | Release reservation |
| Manual reservation | Same table, `source` = manual / production / transfer |

Priority (lower number wins when stock is scarce):

| Priority | Typical source |
| -------- | -------------- |
| 10 | Urgent customer order |
| 20 | Customer order |
| 30 | Production requirement |
| 40 | Transfer requirement |

The engine must not allocate the same available unit twice. Allocation is serialized per `(warehouse, product, variant, batch)` row (row lock on `stock_balances`).

Hard-reserve vs soft-reserve is a company setting. Default: hard (Available blocks oversell).

---

## 5. Batch and serial

- Batch-managed product: movement **requires** `batch_id`. Expiry lives on `product_batches`.
- Serial-managed: one serial per unit; `quantity_in/out` is 1 per serial movement (or qty 1 lines).
- Serial cannot exist in two warehouses on hand at once.

---

## 6. Cost on the movement

`unit_cost` comes from the [valuation](../inventory/valuation.md) policy (FIFO, weighted average, standard). The inventory service asks valuation; sales/purchase services do not pick a cost id.

---

## 7. Negative stock

Default: **reject** issue/delivery if Available < qty. Company/warehouse flag `allow_negative_stock` may permit it; still write the movement and raise an alert event `NegativeStockAttempt` if it would have failed.

---

## 8. Posting API (internal)

Not a public “insert movement” API for the UI.

```text
InventoryLedger.post_in(document, lines)
InventoryLedger.post_out(document, lines)
InventoryLedger.post_transfer(document)   # OUT + IN
InventoryLedger.reverse(document)
```

Public APIs are the inventory documents in [Inventory API](../inventory/api.md).

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
