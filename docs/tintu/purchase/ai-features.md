# Purchase AI Features (Phase 7)

Not live in Phases 1–4. Drafts only. [AI Architecture](../platform/ai-architecture.md)

## Assistant

User: *We need 500 units of Product A next month.*

Analyze: on hand, reserved, incoming, open SO, consumption, lead time, MOQ, vendor price, reorder point.

Recommend supplier, qty, ETA, cost **with factors and confidence**. User confirms → **draft** PR or PO.

## Demand forecast

Inputs: sales history, seasonality, stock, open SO/PO, lead time. Output: forecast, reorder point, suggested qty, stock-out date. Show confidence and data period. Not certain.

## Supplier comparison

Price, lead time, MOQ, on-time history, rejection rate, payment terms, currency. Transparent table. No hidden score.

## Document reader

Upload vendor invoice/DN → extract fields → match vendor, product, PO, receipt → **draft bill** with mismatch highlights. Three-way match, never silent accept.

## Safety

Same purchase permissions. No auto-post receipt or payment.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
