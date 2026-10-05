# Sales Accounting

[Posting engine](../platform/posting-engine.md) maps events to accounts.

| Event | Typical journal |
| ----- | --------------- |
| Invoice posted | Dr AR · Cr Revenue · Cr Output tax (Cr Discount if separate) |
| Payment | Dr Bank/Cash · Cr AR |
| Return | Reverse revenue/tax; credit AR |
| Delivery posted (policy: COGS at ship) | Dr COGS · Cr Inventory |
| Invoice posted (policy: COGS at bill, no delivery) | Dr COGS · Cr Inventory |

Company chooses COGS at delivery vs invoice. Sales services do not embed ledger ids.

Round-off and discount roles from posting rules.

Phase 5 writes `journal_entries`. Until then, events go to the posting outbox when GL is off.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
