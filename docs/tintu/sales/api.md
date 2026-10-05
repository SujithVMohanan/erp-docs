# Sales API

```text
POST/GET/PUT /api/v1/sales/quotations
POST /api/v1/sales/quotations/{id}/submit
POST /api/v1/sales/quotations/{id}/convert-to-order

POST/GET/PUT /api/v1/sales/orders
POST /api/v1/sales/orders/{id}/submit
POST /api/v1/sales/orders/{id}/approve
POST /api/v1/sales/orders/{id}/cancel
POST /api/v1/sales/orders/{id}/convert-to-delivery
POST /api/v1/sales/orders/{id}/convert-to-invoice

POST/GET/PUT /api/v1/sales/deliveries
POST /api/v1/sales/deliveries/{id}/post
POST /api/v1/sales/deliveries/{id}/convert-to-invoice

POST/GET/PUT /api/v1/sales/invoices
POST /api/v1/sales/invoices/{id}/post
POST /api/v1/sales/invoices/{id}/void

POST/GET /api/v1/sales/returns
POST /api/v1/sales/returns/{id}/post

POST/GET /api/v1/sales/payments
POST /api/v1/sales/payments/{id}/post
```

Convert body: `{ "lines": [ { "source_line_id", "quantity" } ], "warehouse_id"? }`.

`Idempotency-Key` on create/post. List filters: status, customer, branch, dates.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
