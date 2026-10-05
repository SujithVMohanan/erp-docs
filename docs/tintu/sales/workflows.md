# Sales Workflows

## Full B2B

```text
Quotation
    ↓
Sales Order
    ↓
Reservation
    ↓
Delivery / Shipment
    ↓
Sales Invoice
    ↓
Payment
```

## Simple retail

```text
Sales Invoice
    ↓
Payment
```

Invoice post may create implied stock OUT (and optional implied delivery) when company setting `invoice_decrements_stock` is true and no delivery exists.

## Optional stages

Quotation may convert to order (partial lines). Order may invoice without delivery (services). Delivery may wait for invoice. All conversions use [document_links](../platform/document-relationships.md).

## Partial fulfilment

```text
Ordered = 100
Delivered = 40   Remaining = 60
Delivered = 30   Remaining = 30
Delivered = 30   Remaining = 0  → FULFILLED
```

## Reservation

On order approve (if `auto_reserve`): create reservation. Delivery post consumes reservation then ledger OUT. Cancel/reduce releases reserved qty.

## Returns

Against delivery or invoice → stock IN + AR credit.

## Payments

Allocate to invoices; outstanding drives PARTIALLY_PAID / PAID.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
