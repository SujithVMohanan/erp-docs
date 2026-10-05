# Inventory Valuation

Valuation produces `unit_cost` / `total_cost` on [inventory movements](../platform/inventory-ledger.md). Sales and purchase modules do not choose account ids or invent cost.

---

## Methods (company setting)

| Method | OUT cost |
| ------ | -------- |
| `FIFO` | Oldest remaining receipt layers |
| `WEIGHTED_AVERAGE` | Moving average after each IN |
| `STANDARD` | Product standard cost; variance on IN |

Default for new companies: `WEIGHTED_AVERAGE` (or FIFO if configured at onboarding).

---

## Layers (FIFO)

### `inventory_cost_layers`

| Field | Type |
| ----- | ---- |
| `warehouse_id`, `product_id`, `variant_id`, `batch_id` | Dimensions |
| `qty_remaining` | |
| `unit_cost` | |
| `source_movement_id` | Receipt/adjustment IN |

OUT consumes layers in date/id order. Transfer: OUT consumes source layers; IN creates layers at the same unit cost (no markup).

---

## Weighted average

After IN: `new_avg = (on_hand_value + in_value) / (on_hand_qty + in_qty)`.

OUT uses current average. Store that average on the movement for reprint.

---

## Landed cost

Purchase freight/duty allocated to receipt lines updates layer or average **via a landed-cost document**, not by editing posted movements in place. Implementation detail in Phase 5; contract: extra IN value or adjustment IN.

---

## Reports

- Stock value = Σ (on hand × cost) per warehouse
- COGS = Σ OUT `total_cost` for sales issues in period

Inventory value changes only through movements.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
