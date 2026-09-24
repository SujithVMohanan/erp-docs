# Purchase Workflows

## Controlled

```text
Purchase Requisition
       ↓
Approval
       ↓
Purchase Order
       ↓
Purchase Receipt
       ↓
Vendor Bill
       ↓
Payment
```

## Direct

```text
Purchase Receipt
       ↓
Vendor Bill
       ↓
Payment
```

Also allowed: PO without requisition; bill without receipt if services (`receipt_not_required`).

## Partial receipts

```text
PO line qty 100
Receipt-1  70     remaining 30
Receipt-2  30     remaining 0  → PO FULFILLED
```

Links: [document_links](../platform/document-relationships.md). Over-receipt blocked unless company allows.

## Three-way match

```text
Purchase Order
      +
Purchase Receipt
      +
Vendor Bill
```

On bill post, compare qty and amount (and tax) against linked PO/receipt. Result: `MATCHED` · `QTY_MISMATCH` · `PRICE_MISMATCH` · `TAX_MISMATCH`. **Highlight** discrepancies. Company setting `block_on_mismatch` prevents post until resolved. Never silently accept.

## Returns

Return against receipt or bill → stock OUT (or reverse IN) + AP credit via posting service.

## Payments

Allocate to one or more bills (`document_links` qty/amount). Outstanding = bill grand total − allocated payments.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
