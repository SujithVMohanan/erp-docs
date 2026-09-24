# Sales State Machine

[Workflow engine](../platform/workflow-engine.md).

## Quotation

```text
DRAFT → SUBMITTED → APPROVED → CLOSED   (converted / expired)
DRAFT → CANCELLED
```

Converted qty tracked via links; remaining 0 → CLOSED.

## Sales order / delivery

```text
DRAFT → SUBMITTED → APPROVED → PROCESSING → PARTIALLY_FULFILLED → FULFILLED → CLOSED
```

Cancel rules: no posted downstream, or compensating docs.

## Invoice

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
sales.quotation.create / send / convert
sales.order.create / edit / submit / approve / cancel
sales.order.convert-to-delivery
sales.delivery.create / post
sales.invoice.create / post
sales.return.create / post
sales.payment.create / post
```

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
