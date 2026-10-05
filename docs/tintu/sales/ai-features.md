# Sales AI Features (Phase 7)

Not implemented in Phases 1–4. [AI Architecture](../platform/ai-architecture.md)

## Sales assistant

*Create a sales order for ABC Traders for 50 of Product A and 20 of Product B, deliver to Kochi warehouse.*

→ Structured draft: customer, products, qty, warehouse, dates, price, discount, tax. User confirms. **No auto-post.**

## Quotation generation

Identify customer/products, current price, discounts, tax, assumptions listed, save **draft** quotation.

## Insights (after-the-fact, background)

Declining customers, frequent products, baskets, odd discounts, overdue, credit-limit proximity, reorder likelihood, trends, qty/margin anomalies. Show data source. Never silently edit documents.

## Reorder assistant

History → frequency → average qty → last purchase → *usually every 28–35 days; last order 31 days ago* → button **Create draft sales order**.

## Contextual Assist on the order screen

Suggest price, history, products, stock, delivery date, draft customer message.

## Safety

Permissions identical to the user. Ambiguous SKUs → choices, not guesses. No SQL. `created_by_type = AI`.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
