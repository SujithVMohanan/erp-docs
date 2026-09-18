# Purchase API

```text
POST/GET/PUT /api/v1/purchases/requisitions
POST /api/v1/purchases/requisitions/{id}/submit
POST /api/v1/purchases/requisitions/{id}/approve
POST /api/v1/purchases/requisitions/{id}/convert-to-order

POST/GET/PUT /api/v1/purchases/orders
POST /api/v1/purchases/orders/{id}/submit
POST /api/v1/purchases/orders/{id}/approve
POST /api/v1/purchases/orders/{id}/cancel
POST /api/v1/purchases/orders/{id}/convert-to-receipt
POST /api/v1/purchases/orders/{id}/convert-to-bill

POST/GET/PUT /api/v1/purchases/receipts
POST /api/v1/purchases/receipts/{id}/post
POST /api/v1/purchases/receipts/{id}/convert-to-bill

POST/GET/PUT /api/v1/purchases/bills
POST /api/v1/purchases/bills/{id}/post
GET  /api/v1/purchases/bills/{id}/match

POST/GET /api/v1/purchases/returns
POST /api/v1/purchases/returns/{id}/post

POST/GET /api/v1/purchases/payments
POST /api/v1/purchases/payments/{id}/post
```

`Idempotency-Key` on POST create/post. Pagination/filter/sort per [document engine](../platform/document-engine.md).

Convert bodies: `{ "lines": [ { "source_line_id", "quantity" } ] }`.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
