# Purchase Accounting

Modules call the [posting engine](../platform/posting-engine.md). No hardcoded purchase/AP account ids.

| Event | Typical journal |
| ----- | --------------- |
| Receipt posted (inventory item) | Dr Inventory · Cr GRNI |
| Vendor bill (inventory already received) | Dr GRNI · Dr Input tax · Cr AP (price variance if any) |
| Vendor bill (expense / no receipt) | Dr Expense · Dr Input tax · Cr AP |
| Payment | Dr AP · Cr Bank |
| Return | Reverse inventory/AP per links |

Rules configured per company: Purchase, Inventory, Input tax, AP, Discount, Round-off, GRNI.

Phase 5 implements journals; this module only emits events with calculated totals.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
