# Calculation and Pricing Engine

> **Simplify. Manage. Grow.**

The **backend** is the source of truth for amounts. Frontend math is for immediate UX only. Saving or posting always recalculates on the server and stores the result.

The same engine is used by Sales, Purchase, and any future billing document.

Related: tax configuration in [Login Flow](../../sujith/doc1-loginflow.md) · [Posting Engine](posting-engine.md)

---

## 1. Line pipeline

```text
Quantity × Rate
        ↓
Gross Amount
        ↓
Discount (percent and/or amount)
        ↓
Taxable Amount
        ↓
Tax (inclusive or exclusive; exemptions)
        ↓
Line charges (if any)
        ↓
Line Total
```

## 2. Header pipeline

```text
Sum of line totals (net of line tax per tax display mode)
        ↓
Header discount
        ↓
Header charges (shipping, packing)
        ↓
Document tax (if header-level)
        ↓
Withholding / TDS (where applicable)
        ↓
Round off
        ↓
Grand Total
```

Landed cost (purchase): allocate freight/duty onto receipt lines **after** bill/landed-cost document — used for inventory unit cost, not for vendor bill grand total unless configured.

---

## 3. Stored result

Persist both inputs and outputs so reprints match history.

| Field (line) | Meaning |
| ------------ | ------- |
| `quantity`, `unit_price` | Inputs |
| `gross_amount` | qty × rate |
| `discount_percent`, `discount_amount` | Inputs / computed |
| `taxable_amount` | |
| `tax_amount` | |
| `tax_breakdown` | JSONB: component → amount (CGST/SGST/IGST or VAT) |
| `line_total` | |
| `price_source` | Why this rate |

| Field (header) | Meaning |
| -------------- | ------- |
| `subtotal` | |
| `discount_total` | |
| `tax_total` | |
| `charges_total` | |
| `withholding_total` | |
| `round_off` | |
| `grand_total` | |
| `calculation_version` | Engine version id |

If the client sends totals, the server **ignores** them except for a warning when they differ beyond a small epsilon (UX stale form).

---

## 4. Tax

- Inclusive vs exclusive is a company / tax-rule setting.
- Jurisdiction from source/destination (company tax rules) — do not hardcode GST splits in the sales service.
- Exemptions: customer/vendor/product tax category.
- TDS/withholding: purchase/sales as configured; stored separately from output tax.

---

## 5. Currency

- Document currency vs company base currency.
- Rate dated `doc_date` from a FX table (or 1 if same).
- Store `exchange_rate` and `base_grand_total` on the header.

---

## 6. Pricing engine

Resolve `unit_price` before the amount pipeline.

```text
Product base price
       ↓
Customer / vendor price list
       ↓
Quantity / volume break
       ↓
Promotion (date-valid)
       ↓
Branch / company list override
       ↓
Manual override (permission)
       ↓
Final price
```

Each step records a candidate. The winner is stored as `unit_price` plus:

```text
price_source = "Customer Price List: Wholesale-2026"
```

UI example: *Price ₹92 applied from Customer Price List: Wholesale-2026.*

If AI or a user overrides, `price_source = "Manual override"` (or `"AI suggestion, confirmed"`) and `price_overridden = true`.

### `price_lists`

| Field | Type |
| ----- | ---- |
| `id` | BIGINT PK |
| `company_id` | BIGINT |
| `name` | VARCHAR(150) |
| `code` | VARCHAR(50) |
| `currency_code` | VARCHAR(10) |
| `valid_from` | DATE |
| `valid_to` | DATE NULL |
| `list_type` | VARCHAR(20) `selling` · `buying` |
| `status` | VARCHAR(30) |

### `price_list_items`

| Field | Type |
| ----- | ---- |
| `price_list_id` | BIGINT FK |
| `product_id` | BIGINT |
| `variant_id` | BIGINT NULL |
| `min_qty` | DECIMAL(18,6) |
| `unit_price` | DECIMAL(18,6) |

### `customer_price_lists` / `vendor_price_lists`

Assign a list to a party, optional priority.

Promotions: `promotions` + `promotion_items` with date range, percent or amount off, optional coupon. Evaluated after volume breaks.

---

## 7. Rounding

Company `rounding_method` and `decimal_precision` from financial settings (onboarding). Apply once at grand total unless tax law requires per-line rounding — then follow tax rules.

---

## 8. API

```text
POST /api/v1/calculations/preview
```

Body: document type + lines (qty, product, party, warehouse). Response: fully calculated lines and header **without** persisting. Editors call this on blur; save still recalculates.

---

## 9. What this engine is not

- Not frontend JavaScript formulas bound to voucher column ids.
- Not a stored formula string interpreter per voucher type.
- Tax **rates** live in the tax engine; this engine **applies** them.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
