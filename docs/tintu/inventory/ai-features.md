# Inventory AI Features (Phase 7)

**Not implemented in this documentation phase.** Assistants create **drafts** or read-only answers. They never post stock.

Architecture: [AI Architecture](../platform/ai-architecture.md)

---

## Assistant questions (future)

- Which products may run out in two weeks?
- Why is Product A low?
- Which warehouse has excess?
- Slow-moving products?

Answers must cite actual balances, open SO/PO, and links to movements — no invented qty.

## Alerts (events, can be rules first)

Low stock · stock-out risk · excess · dead · slow/fast moving · expiry risk · negative stock attempt · unusual movement.

Example copy: *Product A may stock out in ~8 days given recent consumption and incoming POs.*

## Anomaly detection

Flag large adjustments, odd transfers, repeated corrections for **human review**. Do not accuse fraud.

## Batch / expiry intelligence

Expiring soon, slow-moving near expiry, suggest transfer or sales priority. Thresholds configurable.

## Warehouse recommendation (on delivery)

Customer location + availability + reserved + lead time → *Recommended: Kochi — available stock + shortest delivery.* User may override.

## Safety

Same permissions as the user. Draft stock adjustment only. No direct ledger writes.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
