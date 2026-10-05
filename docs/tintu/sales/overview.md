# Sales Overview

> **Simplify. Manage. Grow.**

Sales documents are explicit: quotations, orders, deliveries, invoices, returns, customer payments. Companies skip stages they do not need (retail: invoice → payment). Partial fulfilment is first-class.

Not a generic sales voucher keyed by entry type.

Related: [Document engine](../platform/document-engine.md) · [Workflow](../platform/workflow-engine.md) · [Relationships](../platform/document-relationships.md) · [Calculation](../platform/calculation-engine.md) · [Inventory ledger](../platform/inventory-ledger.md) · [Posting](../platform/posting-engine.md)

---

## Documents

```text
Sales
├── Quotations
├── Sales Orders
├── Deliveries / Shipments
├── Sales Invoices
├── Sales Returns
└── Customer Payments
```

## Header extras

| Document | Extra fields |
| -------- | ------------ |
| Quotation | `customer_id`, `valid_until` |
| Order | `customer_id`, `promised_date`, `warehouse_id` default, `price_list_id` |
| Delivery | `customer_id`, `warehouse_id`, `ship_to_address` |
| Invoice | `customer_id`, `due_date`, `payment_terms_id` |
| Return | `customer_id`, `warehouse_id` |
| Payment | `customer_id`, `amount`, `payment_method`, `bank_account_id` |

Customers: tenant `parties` (`party_type = customer`). Credit limit and outstanding used at order/invoice post.

---

## Pages

[Workflows](workflows.md) · [State machine](state-machine.md) · [Business rules](business-rules.md) · [API](api.md) · [Calculations](calculations.md) · [Accounting](accounting.md) · [AI](ai-features.md)

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
