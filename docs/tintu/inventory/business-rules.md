# Inventory Business Rules

1. Ledger is authoritative; UI qty is a snapshot.
2. Issue/delivery/transfer-out requires `Available ≥ qty` unless `allow_negative_stock`.
3. Transfer posts paired OUT+IN; never one side only (except configured two-step, which is still two explicit posts).
4. Batch products: batch required on every movement.
5. Serial products: serial unique; qty 1 per serial movement.
6. Reservations do not move stock; they only change reserved qty.
7. Same available unit cannot be reserved twice (row lock on balance).
8. Posted documents are immutable; reverse via void/return/adjust.
9. Adjustment reasons are required on post.
10. Count posting creates adjustments; it does not rewrite balances directly.
11. Cost on OUT follows valuation policy, not a user-typed cost (unless standard-cost override permission).
12. All operations are company/tenant scoped; warehouse must belong to the document branch (or allowed cross-branch transfer with permission).
13. Idempotent POST with `Idempotency-Key`.
14. Atomic: movements + balance update (+ journal when GL on) or rollback.
15. Do not show movement internal ids as the user primary key; show document number.

---

## Events

`StockReceived` · `StockIssued` · `StockTransferred` · `StockAdjusted` · `StockReserved` · `StockReservationReleased` · `NegativeStockAttempt`

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
