# AI Architecture (Phase 7)

> **Simplify. Manage. Grow.**

AI reduces repetitive work. It does **not** replace permissions, workflow, tax, stock validation, or audit. This page is the target architecture. **It is not implemented in Phases 1–4.**

Related: [Document Engine](document-engine.md) · [Audit](audit.md) · module `ai-features.md` pages.

---

## 1. Principle

```text
Old ERP: many manual steps
Niyanthra: user states intent → AI drafts → user confirms → workflow posts
```

Consequential writes (orders, invoices, adjustments, payments) are **drafts** until a human (with permission) submits/approves/posts.

AI never:

- Bypasses company/tenant/branch isolation
- Runs arbitrary SQL
- Posts inventory or journals
- Approves its own drafts by default
- Silently changes prices, tax, or qty

---

## 2. Control flow

```text
User
  ↓
AI intent parser
  ↓
Permission check (same codes as UI)
  ↓
Structured tool call
  ↓
ERP service (validation)
  ↓
Result (draft or read-only answer)
```

Write path:

```text
AI → Create DRAFT → User confirmation → Workflow / permission → Post
```

---

## 3. Tools (examples)

Read: `search_customers`, `search_products`, `get_stock`, `get_open_orders`, `get_price`.

Write (draft only): `create_sales_order_draft`, `create_quotation_draft`, `create_purchase_requisition_draft`, `create_stock_adjustment_draft`.

Each tool is a server function with a JSON schema. The model may only call listed tools. No generic `execute_sql`.

---

## 4. Ambiguity

If the product or customer is not unique:

```text
Do not guess.
Return candidates: Product A, Product A - 1KG, Product A - 5KG
```

If required fields are missing, ask. Never invent SKUs or rates.

---

## 5. Explainable recommendations

Never only “Buy 500 units.” Include factors and confidence:

```text
Recommended purchase: 500
Average monthly demand: 380
Current available: 120
Open customer orders: 150
Lead time: 12 days
Safety stock: 100
Confidence: Medium
Data period: last 12 months
```

Confidence is documented (e.g. Medium = 6–12 months history, no strong seasonality model yet). Do not present forecasts as certain.

---

## 6. UX

AI is **contextual**, not a separate bolted-on chatbot only.

On a sales order: suggest price, customer history, stock, delivery warehouse.

On a PO: supplier compare, qty, unusual price.

On inventory: explain stock, stock-out risk, excess, transfer suggestion.

Natural language (“Show today’s sales”) is the same tool layer.

---

## 7. Execution

- LLM calls are **Celery** / async. Invoice save must not wait on insights.
- OCR (vendor bill reader) is background; user gets a draft bill to confirm.
- Insights and anomaly flags are events after post, never blocking post.

---

## 8. Safety checklist

| Control | AI must honor |
| ------- | ------------- |
| Permissions | Same resource.action as the user |
| Isolation | Session company/tenant/branch |
| Approval | Workflow engine |
| Accounting | Posting service only after human post |
| Inventory | Ledger validation |
| Tax | Calculation engine |
| Audit | `created_by_type = AI`, `ai_action_id` |

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
