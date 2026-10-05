# Purchase State Machine

[Workflow engine](../platform/workflow-engine.md).

## Requisition / PO

```text
DRAFT → SUBMITTED → APPROVED → PROCESSING → PARTIALLY_FULFILLED → FULFILLED → CLOSED
DRAFT → CANCELLED
SUBMITTED → REJECTED
APPROVED → CANCELLED   (zero receipts)
```

PO `PROCESSING` = approved, awaiting receipts. Remaining qty drives partial vs fulfilled.

## Receipt

```text
DRAFT → POSTED
```

Optional approve. POST creates stock IN. VOID reverses movements.

## Vendor bill

```text
DRAFT → POSTED → PARTIALLY_PAID → PAID
POSTED → VOID
```

`PAID → DRAFT` forbidden.

## Payment

```text
DRAFT → POSTED
```

---

## Permissions

```text
purchase.requisition.create / submit / approve
purchase.order.create / approve / cancel
purchase.receipt.create / post
purchase.bill.create / post
purchase.return.create / post
purchase.payment.create / post
```

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
