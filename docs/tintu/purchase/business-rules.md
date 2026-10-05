# Purchase Business Rules

1. Vendor required on PO, receipt (goods), bill, payment.
2. Converted qty cannot exceed source remaining (unless over-receipt setting).
3. Receipt post → inventory IN + `PurchaseReceiptPosted`.
4. Bill post → AP via posting service; inventory/expense per product `track_inventory` (GRNI clear vs expense).
5. Three-way match results are stored on the bill; mismatches are visible; optional hard block.
6. Duplicate `vendor_invoice_no` per vendor/company is rejected.
7. Payment cannot exceed bill outstanding.
8. Services: bill without receipt when `requires_receipt = false` on product.
9. Returns need original link when stock was received.
10. Prices from [pricing engine](../platform/calculation-engine.md) (buying lists); explainable `price_source`.
11. AI drafts only; no auto-approve PO.
12. Tenant/company/branch isolation; idempotent create/post.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
