# Purchase Overview

> **Simplify. Manage. Grow.**

Purchase uses **explicit documents**, not a generic purchase voucher. Stages are optional: a small company can start at receipt or bill; a controlled company uses requisition → approval → PO → receipt → bill → payment.

Related: [Workflow](../platform/workflow-engine.md) · [Relationships](../platform/document-relationships.md) · [Calculation](../platform/calculation-engine.md) · [Posting](../platform/posting-engine.md)

---

## Documents

```text
Purchase
├── Purchase Requisitions
├── Purchase Orders
├── Purchase Receipts
├── Vendor Bills
├── Purchase Returns
└── Vendor Payments
```

## Header extras (beyond shared document fields)

| Document | Extra fields |
| -------- | ------------ |
| Requisition | `needed_by`, `requester_id` |
| PO | `vendor_id`, `expected_date`, `price_list_id` |
| Receipt | `vendor_id`, `warehouse_id`, `po` links |
| Vendor bill | `vendor_id`, `vendor_invoice_no`, `due_date`, match status |
| Return | `vendor_id`, `warehouse_id` |
| Payment | `vendor_id`, `amount`, `payment_method`, `bank_account_id` |

Vendors live in tenant `parties` (`party_type = vendor`) or `vendors` — one party master is preferred, with roles.

---

## Pages

[Workflows](workflows.md) · [State machine](state-machine.md) · [Business rules](business-rules.md) · [API](api.md) · [Calculations](calculations.md) · [Accounting](accounting.md) · [AI](ai-features.md)

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
