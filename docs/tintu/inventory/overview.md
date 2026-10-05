# Inventory Overview

> **Simplify. Manage. Grow.**

Inventory is a domain of **explicit documents** (receipts, issues, transfers, adjustments, reservations, counts) plus masters (products, warehouses, batches, serials). Stock on hand is the [inventory ledger](../platform/inventory-ledger.md), not a single quantity column.

This is Phase 2 documentation. Accounting journals are Phase 5 via the [posting engine](../platform/posting-engine.md). AI assistants are Phase 7.

---

## Documents

```text
Inventory
├── Stock Receipts
├── Stock Issues
├── Stock Transfers
├── Stock Adjustments
├── Stock Reservations
├── Stock Counts
├── Batch Tracking
├── Serial Tracking
└── Inventory Valuation
```

There is no generic voucher type that represents all of these.

---

## Isolation

All tables are tenant-scoped. Headers carry `organization_id`, `company_id`, `branch_id`. Warehouses belong to a branch. Users with branch-scoped rights only post to their warehouses.

---

## Masters

### `warehouses`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `public_id` | UUID UNIQUE | |
| `company_id` | BIGINT | |
| `branch_id` | BIGINT | |
| `code` | VARCHAR(30) | Unique per company |
| `name` | VARCHAR(150) | |
| `allow_negative_stock` | BOOLEAN | Default false |
| `status` | VARCHAR(30) | |

### `products`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `company_id` | BIGINT | |
| `sku` | VARCHAR(80) | |
| `name` | VARCHAR(200) | |
| `uom_id` | BIGINT | |
| `track_inventory` | BOOLEAN | |
| `track_batch` | BOOLEAN | |
| `track_serial` | BOOLEAN | |
| `valuation_method` | VARCHAR(30) NULL | Override company default |
| `standard_cost` | DECIMAL(18,6) NULL | |
| `list_price` | DECIMAL(18,6) NULL | Base selling price |
| `status` | VARCHAR(30) | |

### `product_batches`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `product_id` | BIGINT | |
| `batch_no` | VARCHAR(80) | |
| `expiry_date` | DATE NULL | |
| `manufactured_on` | DATE NULL | |

### Document headers

`stock_receipts`, `stock_issues`, `stock_transfers`, `stock_adjustments`, `stock_counts` follow [shared header/line fields](../platform/document-engine.md). Transfers add `from_warehouse_id`, `to_warehouse_id`. Adjustments add `reason_code`.

Onboarding may seed a **Base Location**; Niyanthra treats that as the first warehouse of branch `HO`.

---

## Pages in this module

- [Workflows](workflows.md)
- [State machine](state-machine.md)
- [Business rules](business-rules.md)
- [API](api.md)
- [Valuation](valuation.md)
- [AI features](ai-features.md) (Phase 7)

Platform: [Document engine](../platform/document-engine.md) · [Workflow](../platform/workflow-engine.md) · [Ledger](../platform/inventory-ledger.md)

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
