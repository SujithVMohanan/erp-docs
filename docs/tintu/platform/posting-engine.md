# Accounting Posting Engine

> **Simplify. Manage. Grow.**

Sales, Purchase, and Inventory **must not** write ledger rows directly. They call a posting service with a business event. Account ids come from **configurable rules**, never from hardcoded numbers in a sales service.

Phase 5 implements the general ledger. This page is the **contract** those modules depend on.

Related: [Calculation Engine](calculation-engine.md) · [Audit](audit.md)

---

## 1. Flow

```text
Sales Invoice posted
        ↓
Accounting Posting Service
        ↓
Journal Entry + Journal Lines
        ↓
AR / AP / tax outstanding updates
```

The posting call runs in the **same database transaction** as inventory movements when both apply (see [Document Engine](document-engine.md) atomic posting). If GL is not enabled yet, the service records a `posting_outbox` row and still commits inventory only when company setting allows “inventory without GL” (Phase 1 companies). Default for production: both or neither.

---

## 2. Event types

| Event | Typical entry |
| ----- | ------------- |
| `sales_invoice_posted` | Dr AR · Cr Revenue · Cr Output tax |
| `customer_payment_posted` | Dr Bank/Cash · Cr AR |
| `sales_return_posted` | Reverse revenue/tax; Dr AR credit |
| `vendor_bill_posted` | Dr Expense/Inventory · Dr Input tax · Cr AP |
| `vendor_payment_posted` | Dr AP · Cr Bank/Cash |
| `purchase_return_posted` | Reverse bill as configured |
| `stock_receipt_posted` | Dr Inventory · Cr GRNI / clearing |
| `stock_issue_posted` | Dr COGS / expense · Cr Inventory |
| `delivery_posted` (when invoice later) | Optional COGS at delivery vs invoice — company policy |
| `stock_adjustment_posted` | Dr/Cr Inventory · opposite Adjustment expense/gain |

---

## 3. Rule catalog

### `account_posting_rules`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `company_id` | BIGINT | |
| `event_type` | VARCHAR(50) | |
| `line_role` | VARCHAR(40) | `ar` · `revenue` · `output_tax` · `ap` · `inventory` · `cogs` · `input_tax` · `discount` · `round_off` · `adjustment` · `grni` · `bank` |
| `account_id` | BIGINT | Tenant ledger account |
| `product_category_id` | BIGINT NULL | Optional dimension |
| `warehouse_id` | BIGINT NULL | |
| `tax_type_id` | BIGINT NULL | |
| `status` | VARCHAR(30) | |

Resolution: most specific matching rule wins (category + warehouse + tax) then company default. **Missing required role → posting fails** (do not guess Cash-in-Hand id 8).

Roles the spec requires to be configurable:

```text
Sales Account
Purchase Account
Inventory Account
COGS Account
Input Tax Account
Output Tax Account
Inventory Adjustment Account
Discount Account
Round-Off Account
AR, AP, GRNI/clearing, Bank/Cash
```

---

## 4. Journal shape (Phase 5)

### `journal_entries` / `journal_lines`

| Header | Lines |
| ------ | ----- |
| `document_type`, `document_id` | `account_id`, `debit`, `credit`, `tax_amount` |
| `entry_date`, `currency`, `exchange_rate` | `product_id`, `party_id` optional analytics |
| Unique posted document | Balanced Σ debit = Σ credit |

One journal per posted document. Void creates a reversing journal linked via `document_links` / `reverses_journal_id`.

---

## 5. Service API (internal)

```text
PostingService.post(event_type, document, calculated_totals, inventory_cost)
PostingService.reverse(document)
```

Returns `journal_id` or raises. Idempotent on `document_id` + `event_type` (second post returns existing journal).

---

## 6. What modules send

Not account ids. Example invoice payload:

```text
event: sales_invoice_posted
customer_id, currency, grand_total
lines: product, taxable, tax_breakdown, revenue_amount, discount
```

The posting service maps to accounts via rules.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
