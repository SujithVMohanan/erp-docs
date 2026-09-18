# Sales Business Rules

1. Customer required on order, delivery, invoice, payment.
2. Converted qty ≤ remaining source qty (over-delivery setting optional).
3. Delivery post: Available check, consume reservation, stock OUT, `DeliveryPosted`.
4. Invoice post: calculation freeze, AR posting, optional stock OUT if no delivery.
5. Credit limit: warn or block on approve/post (company setting).
6. Returns require a source delivery/invoice link for stocked items.
7. Payment ≤ invoice outstanding.
8. Prices from selling price lists; `price_source` stored; manual override needs `sales.order.edit-price`.
9. Tax from tax rules (origin/destination); not hardcoded in the sales service.
10. AI creates DRAFT only.
11. Isolation + idempotency + atomic post (lines + stock + journal-or-outbox).
12. UI shows document numbers and statuses, never internal voucher ids.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
