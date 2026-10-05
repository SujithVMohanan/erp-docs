# Inventory State Machine

Uses the [workflow engine](../platform/workflow-engine.md).

## Receipt, issue, transfer, adjustment

```text
DRAFT → SUBMITTED → APPROVED → POSTED
DRAFT → CANCELLED
SUBMITTED → REJECTED → DRAFT
APPROVED → CANCELLED   (if not posted)
```

Simple companies: DRAFT → POSTED (submit/approve skipped). Posted is terminal except VOID (reversing movements).

`POSTED → DRAFT` is forbidden.

## Two-step transfer (optional)

```text
DRAFT → APPROVED → PROCESSING (shipped) → FULFILLED (received)
```

PROCESSING means source OUT done; destination IN pending. Cancellation after PROCESSING requires a compensating receipt/issue, not a silent delete.

## Reservation

```text
DRAFT → ACTIVE → CONSUMED | RELEASED | CANCELLED
```

Partial consume: `PARTIALLY_CONSUMED` then `CONSUMED`.

## Count

```text
DRAFT → IN_PROGRESS → SUBMITTED → APPROVED → POSTED
```

POSTED means variance adjustments posted.

---

## Permissions

```text
inventory.receipt.create / submit / approve / post / cancel
inventory.issue.create / post
inventory.transfer.create / post
inventory.adjustment.create / approve / post
inventory.reservation.create / release
inventory.count.create / approve / post
inventory.warehouse.view
```

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
