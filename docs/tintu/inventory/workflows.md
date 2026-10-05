# Inventory Workflows

> **Simplify. Manage. Grow.**

## Stock receipt

```text
Receipt (DRAFT)
 → Warehouse, product, qty, batch/serial
 → Validation
 → POST → Inventory movement IN
 → Optional posting event stock_receipt_posted
```

Sources: purchase receipt conversion, opening stock, production, manual.

## Stock issue

```text
Issue (DRAFT)
 → Availability check (Available ≥ qty)
 → Batch/serial pick
 → POST → Inventory movement OUT
```

Sources: delivery, production consume, manual issue.

## Stock transfer

```text
Warehouse A  →  Transfer document  →  Warehouse B
```

On post, **two** movements, same `transaction_id`:

```text
A → OUT
B → IN
```

In-transit: optional status `PROCESSING` (OUT from A, not yet IN at B) if company uses two-step transfer. Default: single post, both movements together.

## Stock adjustment

Reasons (configurable `adjustment_reasons`): damage, expiry, theft, stock count difference, correction, write-off, other.

```text
Adjustment DRAFT → (optional APPROVE) → POST
 → IN or OUT movement
 → posting event stock_adjustment_posted
```

Large adjustments should flag [anomaly review](ai-features.md) after post (non-blocking).

## Stock reservation

```text
On Hand = 100, Reserved = 30, Available = 70
```

Sales orders (and optional production/transfer) create reservation lines. Delivery **consumes** reservation then posts OUT. Cancel **releases**.

## Stock count

```text
Count sheet (warehouse / location)
 → Enter counted qty
 → POST creates adjustment(s) for variance
 → Links: stock_count → stock_adjustment
```

## Batch / expiry

Receipts of batch items require batch code + expiry. Issues prefer FEFO unless user overrides (permission).

## Serial

Receipt registers serials; issue must list serials; transfer moves serial warehouse.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
