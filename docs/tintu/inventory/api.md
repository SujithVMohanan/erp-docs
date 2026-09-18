# Inventory API

Django REST `/api/v1/inventory/`. Conventions: [Document engine](../platform/document-engine.md).

## Receipts

```text
POST   /api/v1/inventory/receipts
GET    /api/v1/inventory/receipts
GET    /api/v1/inventory/receipts/{id}
PUT    /api/v1/inventory/receipts/{id}
POST   /api/v1/inventory/receipts/{id}/submit
POST   /api/v1/inventory/receipts/{id}/approve
POST   /api/v1/inventory/receipts/{id}/cancel
POST   /api/v1/inventory/receipts/{id}/post
```

## Issues

```text
POST/GET/PUT /api/v1/inventory/issues
POST /api/v1/inventory/issues/{id}/post
```

## Transfers

```text
POST/GET/PUT /api/v1/inventory/transfers
POST /api/v1/inventory/transfers/{id}/post
POST /api/v1/inventory/transfers/{id}/receive    # two-step only
```

## Adjustments

```text
POST/GET/PUT /api/v1/inventory/adjustments
POST /api/v1/inventory/adjustments/{id}/approve
POST /api/v1/inventory/adjustments/{id}/post
```

## Reservations

```text
POST/GET /api/v1/inventory/reservations
POST /api/v1/inventory/reservations/{id}/release
```

## Counts

```text
POST/GET/PUT /api/v1/inventory/counts
POST /api/v1/inventory/counts/{id}/post
```

## Balances and ledger (read)

```text
GET /api/v1/inventory/balances?warehouse_id=&product_id=
GET /api/v1/inventory/movements?product_id=&warehouse_id=&date_from=
```

No public POST to `movements`.

## Masters

```text
/api/v1/inventory/warehouses
/api/v1/inventory/products
/api/v1/inventory/batches
```

## Headers

`Idempotency-Key` on create and post.

## Permissions

Enforced on every route (`inventory.transfer.create`, `inventory.adjustment.approve`, …).

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
