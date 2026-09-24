# 📐 Niyanthra ERP — Module Implementation Sequence & Critical Path

**Version:** 1.0 (Engineering Release)  
**Date:** September 2026  
**Status:** Framework for Soft-Launch (3-5 Pilot Customers, Kerala)  
**Audience:** Engineering Lead, Backend Architects, Database Engineers, QA Lead, Implementation Manager

---

## Executive Summary

Niyanthra's soft-launch execution spans **9-12 weeks** across 5 core modules, built in a **strict dependency order** designed to maximize validation gates and minimize rollback risk. This document details:

1. **Build sequence** — which module first, which module can't start until predecessors are complete
2. **Critical path** — the longest dependency chain that determines launch readiness
3. **Interconnection points** — where data flows between modules and how consistency is enforced
4. **Pre-launch gates** — the 12 hard validation criteria that must pass before pilot sign-up

**Key Principle:** No module is "done" until it passes integration tests with its dependent modules. A "complete" Inventory module means nothing without Sales Order integration; a "complete" Financial module means nothing without GL posting validation.

---

## Table of Contents

1. [Module Dependency Map](#1-module-dependency-map)
2. [Build Sequence & Milestones](#2-build-sequence--milestones)
3. [Module Interconnectivity & Data Flow](#3-module-interconnectivity--data-flow)
4. [Database Binding & Foreign Key Contracts](#4-database-binding--foreign-key-contracts)
5. [Transaction Atomicity & Consistency Guarantees](#5-transaction-atomicity--consistency-guarantees)
6. [State Machines & Workflow Lifecycles](#6-state-machines--workflow-lifecycles)
7. [Soft-Launch Readiness & Validation Gates](#7-soft-launch-readiness--validation-gates)
8. [Go-Live Checklist & First Customer Onboarding](#8-go-live-checklist--first-customer-onboarding)

---

# 1. Module Dependency Map

## 1.1 Dependency Graph (Visual)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FOUNDATIONAL LAYER                               │
├─────────────────────────────────────────────────────────────────────┤
│  • Master Data (Chart of Accounts, Product Master, Customer Master) │
│  • Compliance Data (GST rates, HSN codes, Professional Tax slabs)   │
│  • Multi-tenant RLS policies & initialization                       │
│  • Authentication & Authorization framework                         │
└────────────────────┬────────────────────────────────────────────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
    ┌────▼──────┐          ┌────▼──────┐
    │ INVENTORY │          │ FINANCIAL │
    │  (Module) │          │  (Module) │
    └────┬──────┘          └────┬──────┘
         │ (Stock Master,       │ (GL accounts,
         │  Warehouses,         │  AR/AP ledgers)
         │  Stock Summary)      │
         │                      │
         ├──────────┬───────────┤
         │          │           │
    ┌────▼────┐ ┌──▼──┐ ┌──────▼──────┐
    │  SALES  │ │ POS │ │  PURCHASE   │
    │    &    │ │     │ │     &       │
    │ DISTRIB.│ │     │ │  PROCURE.   │
    └────┬────┘ └──┬──┘ └──────┬──────┘
         │         │          │
         │    ┌────┴──────────┘
         │    │
         └────┴────────────┬────────────┐
                           │            │
                    ┌──────▼──┐  ┌──────▼──────┐
                    │    CRM  │  │  REPORTING  │
                    │         │  │  & ANALYTICS│
                    └─────────┘  └─────────────┘
                    (Read-only   (Read-only
                     after       after
                     S&D setup)   GL posted)
```

## 1.2 Dependency Matrix

| Module | Build Order | Depends On | Blocks | Can Start When |
|--------|------------|-----------|--------|-----------------|
| **Inventory** | 1 | Foundational | Sales, POS, Purchase | Master data initialized, Stock summary schema ready |
| **Financial** | 2 | Foundational, Inventory (optional) | Sales, Purchase (invoicing), POS | GL schema, compliance data, AR/AP ledgers ready |
| **Sales & Distribution** | 3 | Inventory, Financial | CRM, Reports | SO → Invoice → GL posting proven with 10+ test invoices |
| **Purchase & Procurement** | 4 | Inventory, Financial | Reporting | PO → GRN → Bill → GL posting tested end-to-end |
| **Point of Sale** | 5 | Inventory, Financial, Sales (master data) | CRM | Retail counter billing + real-time inventory sync validated |
| **CRM** | 6 | Sales, Purchase (reference) | Reports | Customer master + order history populated from S&D, P&P |
| **Reporting & Analytics** | 7 | All transactional modules | None | All GL posting stable, daily reconciliation passing |

---

# 2. Build Sequence & Milestones

## 2.1 Phase 1: Foundational Layer (Weeks 1-2)

### Objectives
- Master data schemas and seeding
- Multi-tenant RLS policies enforced
- Authentication & role provisioning
- Compliance parameter table (tax rates, HSN, state codes)

### Deliverables

| Component | Spec | Owner | Acceptance Criteria |
|-----------|------|-------|-------------------|
| **Chart of Accounts (CoA)** | Part 2 §2.1 | Finance Lead | 150+ default GL accounts, state-specific Professional Tax slabs, mappings to tax categories |
| **Product Master** | Part 2 §2.2 | Inventory Lead | 500+ test SKUs, HSN codes, tax rates, UOM, re-order levels |
| **Customer Master** | Part 2 §2.3 | Sales Lead | 100+ test customers, GSTIN validation (optional at sign-up), credit limits |
| **Compliance Parameters** | Part 2 §6 | Finance Lead | GST rates (5%, 12%, 18%, 28%), HSN-to-tax mapping, state codes, Professional Tax slabs |
| **RLS Policies** | Part 2 §4 | Backend Lead | Row-level security enforced on all tenant tables, company_id isolation verified |
| **Auth & RBAC** | Part 4 | Backend Lead | 6 roles defined (Owner, Admin, Accountant, Manager, Cashier, Staff), invite flow working |

### Database Initializations

```sql
-- Tenant creation (shared-schema multi-tenant)
CREATE TABLE companies (
    company_id BIGINT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    state_code CHAR(2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Chart of Accounts seeding
INSERT INTO chart_of_accounts (company_id, account_code, account_name, account_type, created_at)
SELECT NULL, '1010', 'Accounts Receivable', 'ASSET', NOW()  -- Shared template
WHERE NOT EXISTS (SELECT 1 FROM chart_of_accounts WHERE account_code = '1010');

-- RLS Policy enforcement
ALTER TABLE sales_orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY company_isolation ON sales_orders
    USING (company_id = current_setting('app.current_company_id')::bigint);

-- Compliance parameters (read-only master)
INSERT INTO compliance_parameters (tax_type, rate, hsn_code_from, hsn_code_to, effective_from, effective_to)
VALUES ('GST', 18.00, '3201', '3209', '2020-01-01', '9999-12-31');
```

### Success Metrics
- ✅ 150+ GL accounts seeded, chart balances
- ✅ 500+ products with HSN codes assigned
- ✅ 100+ customers with credit limits
- ✅ RLS policies block cross-tenant reads in test
- ✅ Auth flow completes in <2 seconds

---

## 2.2 Phase 2: Inventory Module (Weeks 2-4)

### Objectives
- Stock master and warehouse management
- Real-time stock summary (available, reserved, allocated)
- Goods receipt and stock movement tracking
- Reorder point automation

### Deliverables

| Component | Depends On | Owner | Acceptance Criteria |
|-----------|-----------|-------|-------------------|
| **Stock Master** | Foundational | Inventory | 500+ products with warehouse allocation, reorder levels, safety stock |
| **Stock Summary & Ledger** | Stock Master | Inventory | Real-time qty updates, FIFO valuation, no oversale (concurrent reservations tested) |
| **Goods Receipt Note (GRN)** | Stock Master | Inventory | GRN → Stock Movement → Summary updated atomically, 3-way match ready (PO link field) |
| **Stock Reservation** | Stock Summary | Inventory | SO creation triggers immediate reservation, reservation locks prevent oversale in concurrent scenario |

### Critical Data Flow

```
Product Creation
    ↓
├─ product_master (product_id, hsn_code, reorder_level)
├─ stock_summary (product_id, warehouse_id, available_qty, reserved_qty, allocated_qty)
└─ inventory_ledger (historical log of all movements)

Goods Receipt (Purchase integration point)
    ↓
├─ grn table (grn_id, po_id FK, receipt_date, status)
├─ grn_line_items (grn_id, product_id, qty_received, uom)
└─ stock_summary UPDATE: available_qty += qty_received

Stock Reservation (Sales integration point)
    ↓
├─ reservation table (reservation_id, so_id FK, product_id, qty_reserved, warehouse_id, status)
└─ stock_summary UPDATE: available_qty -= qty_reserved, reserved_qty += qty_reserved
```

### Integration Points (Already Defined)

**Reservation on SO Creation:**
```sql
-- Atomic: SO creation + inventory reservation (same transaction)
BEGIN TRANSACTION;
    INSERT INTO sales_orders (...) VALUES (...);
    INSERT INTO reservation (so_id, product_id, qty_reserved, warehouse_id, status)
    SELECT NEW.so_id, product_id, qty_ordered, 1, 'RESERVED'
    FROM sales_order_line_items WHERE so_id = NEW.so_id;
    
    UPDATE stock_summary
    SET available_qty = available_qty - (SELECT SUM(qty_ordered) FROM sales_order_line_items WHERE so_id = NEW.so_id),
        reserved_qty = reserved_qty + (SELECT SUM(qty_ordered) FROM sales_order_line_items WHERE so_id = NEW.so_id)
    WHERE product_id IN (SELECT DISTINCT product_id FROM sales_order_line_items WHERE so_id = NEW.so_id);
COMMIT;
```

### Success Metrics
- ✅ 500+ SKUs in stock master with reorder levels
- ✅ Stock movements logged to inventory_ledger, no missing entries
- ✅ Concurrent reservation test: 5 simultaneous SOs, no oversale
- ✅ FIFO valuation correct on manual spot-check (10 transactions)
- ✅ Reorder alert triggers when qty < reorder_level

---

## 2.3 Phase 3: Financial Module (Weeks 3-5)

### Objectives
- Real-time General Ledger posting
- Accounts Receivable & Payable subledgers
- Multi-currency GL entries (INR + foreign currency in future)
- Daily reconciliation framework
- Tax subledgers (GST, TDS)

### Deliverables

| Component | Depends On | Owner | Acceptance Criteria |
|-----------|-----------|-------|-------------------|
| **GL Entry & Posting** | Foundational | Finance | GL entries balanced, reconciles to trial balance, no partial posts |
| **AR/AP Subledgers** | GL Entry | Finance | AR matches Accounts Receivable GL, AP matches Accounts Payable GL |
| **Tax Subledgers** | GL Entry, Compliance Data | Finance | GST subledgers track Input Tax, Output Tax separately by HSN |
| **Daily Reconciliation** | AR/AP/GL | Finance | Automated daily TB, manual GL-to-bank reconciliation script ready |

### Critical GL Posting Logic

```sql
-- Invoice posting (O2C flow, from Part 2)
-- When tax_invoices status = 'CONFIRMED' → Post GL entries

BEGIN TRANSACTION;
    -- 1. Debit Accounts Receivable
    INSERT INTO gl_entries (company_id, account_code, debit, credit, reference_type, reference_id)
    VALUES (1, '1010', 53100.00, 0, 'INV', 'INV-2026-0001');
    
    -- 2. Credit Revenue
    INSERT INTO gl_entries (company_id, account_code, debit, credit, reference_type, reference_id)
    VALUES (1, '4010', 0, 45000.00, 'INV', 'INV-2026-0001');
    
    -- 3. Credit GST Output (18%)
    INSERT INTO gl_entries (company_id, account_code, debit, credit, reference_type, reference_id)
    VALUES (1, '3310', 0, 8100.00, 'INV', 'INV-2026-0001');
    
    -- 4. Update AR subledger
    INSERT INTO customer_invoices (company_id, customer_id, invoice_id, invoice_amount, status)
    VALUES (1, 101, 'INV-2026-0001', 53100.00, 'POSTED');
    
    -- 5. Update chart_of_accounts balances
    UPDATE chart_of_accounts SET balance = balance + 53100 WHERE account_code = '1010';
    UPDATE chart_of_accounts SET balance = balance + 45000 WHERE account_code = '4010';
    UPDATE chart_of_accounts SET balance = balance + 8100 WHERE account_code = '3310';
    
    -- 6. Mark invoice as POSTED in transactional table
    UPDATE tax_invoices SET status = 'POSTED', posted_date = NOW() WHERE invoice_id = 'INV-2026-0001';
    
COMMIT; -- Entire GL posting atomic or rolls back completely
```

### Success Metrics
- ✅ 50+ test invoices posted to GL, trial balance matches expected totals
- ✅ AR subledger reconciles to GL (customer_invoices.sum = GL account 1010)
- ✅ GST subledger correct (Input Tax + Output Tax correct by HSN)
- ✅ Daily reconciliation script runs, reports zero variances
- ✅ GL posting fails atomically if any step breaks (no partial posts)

---

## 2.4 Phase 4: Sales & Distribution Module (Weeks 5-7)

### Objectives
- Complete O2C flow (SO → Fulfillment → Invoice → Payment)
- Real-time GL integration (invoice creation posts GL)
- AR subledger updates
- e-invoice IRN generation (NSDL API integration)

### Deliverables

| Component | Depends On | Owner | Acceptance Criteria |
|-----------|-----------|-------|-------------------|
| **Sales Order (SO)** | Inventory, Financial | Sales | SO creation reserves inventory, credit check passes, SO → Invoice chain validated |
| **Fulfillment (Pick/Pack/Dispatch)** | SO | Warehouse | SO → Fulfillment → Dispatch status transitions, inventory reserved until dispatch |
| **Tax Invoice Generation** | Fulfillment, GL Entry | Finance | Invoice created, GL posted (DR AR, CR Revenue, CR GST), status = POSTED |
| **e-Invoice (NSDL IRN)** | Tax Invoice | Finance | IRN generated asynchronously, retry on failure, PDF sent to customer |
| **Payment Receipt** | Tax Invoice | Finance | Payment matched to invoice, GL posted (DR Bank, CR AR), invoice status = PAID |

### O2C State Machine (Summary)

```
SO: DRAFT → CONFIRMED (credit + stock check) → PICKED → PACKED → DISPATCHED
                            ↓
                      Dispatch triggers:
                            ↓
INVOICE: DRAFT → POSTED (GL entries created) → SENT (to customer) → PAID (payment received)
                             ↓
                      GL entries:
                      DR AR 53,100 / CR Revenue 45,000 / CR GST Output 8,100
```

### Integration Test Plan

**Test: SO → Invoice → GL Posting → AR Subledger Update (End-to-End)**

```
Input:
  SO-2026-0001: 100 units @ ₹450 = ₹45,000 + ₹8,100 GST = ₹53,100

Expected Output:
  1. SO created, status = CONFIRMED
  2. Inventory reserved: available_qty -100, reserved_qty +100
  3. After dispatch → Invoice generated
  4. GL entries created (DR AR, CR Rev, CR GST)
  5. AR subledger updated: customer_invoices row exists
  6. GL account balances updated
  7. Trial balance reconciles (debits = credits)
  8. e-invoice queued for NSDL

Pass Criteria:
  ✅ SO created
  ✅ Inventory reserved (stock_summary shows available_qty decreased)
  ✅ Invoice status = POSTED (not DRAFT)
  ✅ GL entries exist and balance
  ✅ customer_invoices row exists with correct amount
  ✅ Trial balance matches (sum of all GL debits = sum of credits)
  ✅ e-invoice status = SUBMITTED or VALID (not FAILED)
```

### Success Metrics
- ✅ 10+ SO → Invoice → Payment cycles completed end-to-end
- ✅ Inventory reserved and released correctly in all paths
- ✅ GL posting verified atomically (no partial posts)
- ✅ AR subledger reconciles to GL
- ✅ e-invoice IRN generated for 90%+ invoices (acceptable 10% retry)
- ✅ Payment receipt updates GL correctly

---

## 2.5 Phase 5: Purchase & Procurement Module (Weeks 6-8)

### Objectives
- Complete P2P flow (Requisition → PO → GRN → Bill)
- 3-way matching (PO ↔ GRN ↔ Bill)
- Real-time GL integration (bill posting)
- AP subledger updates
- TDS (Tax Deducted at Source) calculation

### Deliverables

| Component | Depends On | Owner | Acceptance Criteria |
|-----------|-----------|-------|-------------------|
| **Purchase Requisition (PR)** | Inventory | Procurement | PR created from reorder alerts or manual entry, status = DRAFT/APPROVED |
| **Purchase Order (PO)** | PR, Inventory, Financial | Procurement | PO created from PR, linked to supplier, ready for 3-way match |
| **Goods Receipt Note (GRN)** | PO, Inventory | Warehouse | GRN created from PO, qty received, status = POSTED, inventory updated |
| **Purchase Bill** | GRN, Financial | Finance | Bill matched against PO + GRN (3-way), GL posted (DR Inventory, CR AP), TDS calculated |

### 3-Way Matching Workflow

```
Purchase Order (Supplier A, 100 units @ ₹50 = ₹5,000)
    ↓
    └─ PO Status: CONFIRMED
    
GRN (Receipt: 100 units @ ₹50)
    ↓
    └─ Match PO qty (100 = 100) ✓
    └─ Match PO rate (₹50 = ₹50) ✓
    └─ Stock updated: available_qty +100
    
Purchase Bill (Invoice from Supplier: 100 units @ ₹50 + 18% GST = ₹5,900)
    ↓
    └─ Match PO qty: 100 = 100 ✓
    └─ Match GRN qty: 100 = 100 ✓
    └─ Match Bill amount: ₹5,900 vs. PO + taxes ✓
    └─ Calculate TDS (if applicable): ₹500 @ 10% = ₹50
    └─ GL Posted: DR Inventory 5,000 + DR GST Input 900 / CR AP 5,900
    └─ Status: MATCHED (cleared for payment)
    
Payment (Supplier paid ₹5,850, TDS retained ₹50)
    └─ GL Posted: DR AP 5,900 / CR Bank 5,850 + CR TDS Payable 50
```

### 3-Way Match Database Logic

```sql
-- Check if PO, GRN, Bill quantities match
SELECT 
    po_id,
    po_qty,
    grn_qty,
    bill_qty,
    CASE 
        WHEN po_qty = grn_qty AND grn_qty = bill_qty THEN 'MATCHED'
        WHEN po_qty >= grn_qty AND grn_qty >= bill_qty THEN 'PARTIAL_MATCH' -- All received, partial invoiced
        ELSE 'MISMATCH'
    END AS match_status
FROM (
    SELECT 
        po.po_id,
        SUM(po_line.qty_ordered) as po_qty,
        COALESCE(SUM(grn_line.qty_received), 0) as grn_qty,
        COALESCE(SUM(bill_line.qty_invoiced), 0) as bill_qty
    FROM purchase_orders po
    LEFT JOIN po_line_items po_line ON po.po_id = po_line.po_id
    LEFT JOIN grn_line_items grn_line ON po_line.po_line_id = grn_line.po_line_id
    LEFT JOIN bill_line_items bill_line ON grn_line.grn_line_id = bill_line.grn_line_id
    GROUP BY po.po_id
) matching;
```

### Success Metrics
- ✅ 10+ POs created and CONFIRMED
- ✅ 10+ GRNs matched to POs (3-way match logic validated)
- ✅ 10+ Bills matched (PO qty = GRN qty = Bill qty)
- ✅ GL posting for bills verified (no partial posts)
- ✅ AP subledger reconciles to GL
- ✅ TDS calculated correctly (3 different rates tested)

---

## 2.6 Phase 6: Point of Sale Module (Weeks 7-9)

### Objectives
- Retail counter billing
- Real-time inventory sync (POS ↔ Inventory)
- Cash/card payment handling
- Daily POS reconciliation & GL posting

### Deliverables

| Component | Depends On | Owner | Acceptance Criteria |
|-----------|-----------|-------|-------------------|
| **POS Transaction** | Inventory, Financial | POS | Sale created, inventory deducted immediately, GL posted asynchronously |
| **Payment Method** | Financial | POS | Cash, card, wallet supported, daily settlement to GL |
| **Daily POS Reconciliation** | POS Transaction | Finance | End-of-day summary, cash vs. system reconciled, deposits posted to GL |

### POS-to-GL Flow

```
POS Sale (Cashier scans items):
  Item 1: Red Paint 1L × 1 @ ₹450
  Item 2: Brush × 2 @ ₹50 each = ₹100
  Total: ₹550 + ₹99 GST = ₹649 (cash paid)

System Processing:
  ├─ Deduct inventory immediately
  │  ├─ stock_summary: Red Paint (available_qty -1, allocated_qty +1)
  │  ├─ stock_summary: Brush (available_qty -2, allocated_qty +2)
  │  └─ inventory_ledger: movement logged as 'POS_SALE', qty_allocated
  │
  ├─ Create sales transaction
  │  ├─ pos_transactions table: amount = 649, payment_method = 'CASH', status = 'COMPLETED'
  │  └─ pos_transaction_lines: 2 line items (Paint + Brush)
  │
  └─ GL posting (asynchronous, post-commit)
     ├─ DR Bank/Cash 649
     ├─ CR Revenue 550
     └─ CR GST Output 99

Daily Reconciliation (End of Shift):
  ├─ Cash register: ₹5,000 (system balance)
  ├─ Count physical: ₹4,950 (shortage: ₹50)
  ├─ Reconciliation: FLAGGED (shortage > threshold)
  ├─ Option 1: Operator inputs shortage reason
  ├─ Option 2: GL adjustment entry for ₹50 shortage
  └─ Status: RECONCILED_WITH_VARIANCE
```

### Success Metrics
- ✅ 50+ POS sales completed end-to-end
- ✅ Inventory deducted correctly (spot-check 10 sales)
- ✅ GL posting correct (revenue, GST, bank/cash)
- ✅ Daily reconciliation matches to ±₹5 (acceptable variance)
- ✅ Multi-branch POS tested (inventory isolation per warehouse)

---

## 2.7 Phase 7: CRM & Reporting (Weeks 9-11)

### Objectives
- Customer master integration (from S&D, P&P)
- Order history & repeat customer tracking
- Sales pipeline visibility
- Financial reports (P&L, Balance Sheet, Aging)

### Deliverables

| Component | Depends On | Owner | Acceptance Criteria |
|-----------|-----------|-------|-------------------|
| **Customer Profile** | Sales, Purchase | CRM | Customer data enriched with order/payment history, credit status, repeat frequency |
| **Order History** | Sales, Purchase | CRM | All SOs, invoices, POs, bills visible per customer, no missing transactions |
| **Sales Reports** | Sales, GL | Reporting | Sales summary, growth, customer concentration, top products (all manual spot-checks pass) |
| **Financial Reports** | GL, AR/AP | Reporting | P&L, Balance Sheet, aging reports, GST reconciliation (all totals match GL) |

### Report Validation

```
P&L Report (Sept 2026):
  Revenue: ₹4,53,100 (sum of all invoiced amounts)
  COGS: ₹2,25,000 (cost of goods sold from bill data)
  Gross Profit: ₹2,28,100
  Operating Expenses: ₹45,000
  Net Profit: ₹1,83,100

GL Reconciliation:
  Revenue GL account (4010): ₹4,53,100 ✓
  COGS GL account (5010): ₹2,25,000 ✓
  Expense GL accounts (6010, 6020, ...): ₹45,000 ✓
  
Validation: P&L matches GL totals ✓
```

### Success Metrics
- ✅ 100+ customer profiles with complete order history
- ✅ All SOs, invoices, POs, bills linked to customers
- ✅ Sales reports match invoice GL posting (₹ exact match)
- ✅ P&L matches GL (all line items reconcile)
- ✅ Aging report shows correct DSO by customer
- ✅ GST summary matches tax subledger

---

## 2.8 Phase 8: Integration & Performance Testing (Weeks 11-12)

### Objectives
- Full system stress test (1,000 SOs, 500 GRNs, 10 concurrent users)
- End-to-end workflows under load
- Data integrity validation
- Performance baseline (P95 response times)

### Test Scenarios

| Scenario | Setup | Success Criteria |
|----------|-------|-----------------|
| **High Volume SO** | 1,000 SOs created in 2 hours (500 concurrent) | No oversale, all inventory reserved correctly, GL posting complete, P95 < 2s |
| **Inventory Oversale Prevention** | 5 concurrent users creating SOs for same product (100 available) | Only 100 units sold total, no oversale, 4 users get shortage errors |
| **GL Posting Under Load** | 100 invoices posted simultaneously | All GL entries created, trial balance balances, no deadlocks, P95 < 500ms |
| **Multi-Tenant Isolation** | 3 tenants, each creating 100 transactions | No cross-tenant data leakage, each tenant sees only their data |
| **Daily Reconciliation** | Run reconciliation on 1,000 transactions | Reconciliation completes in < 5 minutes, variances < ₹10 |

### Success Metrics
- ✅ P95 response time < 2 seconds (SO creation, Invoice posting)
- ✅ No data corruption or missing transactions
- ✅ GL always balances (debits = credits)
- ✅ Inventory accuracy maintained (no oversale, no negative stock)
- ✅ Multi-tenant isolation verified (no cross-tenant leaks)
- ✅ System recovers gracefully from failures (automatic rollback)

---

# 3. Module Interconnectivity & Data Flow

## 3.1 Critical Interconnection Points

### 3.1.1 Inventory ↔ Sales & Distribution

**Trigger:** SO Creation  
**Flow:** SO line items → Inventory reservation → stock_summary update

```
Event: POST /api/v1/sales_orders (Create SO)
  Input: { customer_id: 101, lines: [{ product_id: 5001, qty_ordered: 100 }] }
  
  Step 1: Validate customer
    ├─ Query customers WHERE customer_id = 101
    ├─ Check: active = true, credit_limit ≥ SO amount
    └─ Abort if validation fails (ROLLBACK)
  
  Step 2: Create SO record
    ├─ INSERT sales_orders (so_id, customer_id, status = 'DRAFT', ...)
    └─ Transaction: OPEN
  
  Step 3: Create SO line items
    ├─ INSERT sales_order_line_items (so_id, product_id, qty_ordered, unit_price)
    └─ Fetch unit_price from product_master (read-only)
  
  Step 4: Reserve inventory (Atomic with Step 2-3)
    ├─ Lock stock_summary WHERE product_id = 5001 (row-level lock)
    ├─ Check: available_qty >= qty_ordered? (100 >= 100?)
    ├─ If YES:
    │  ├─ INSERT reservation (so_id, product_id, qty_reserved, status='RESERVED')
    │  ├─ UPDATE stock_summary SET available_qty -= 100, reserved_qty += 100
    │  └─ Continue
    ├─ If NO:
    │  ├─ Rollback entire transaction
    │  ├─ Return error: "Insufficient inventory"
    │  └─ Abort
  
  Step 5: Confirm SO
    ├─ UPDATE sales_orders SET status = 'CONFIRMED'
    └─ Transaction: COMMIT
  
Output: { so_id: 'SO-2026-0001', status: 'CONFIRMED', inventory_reserved: true }
```

**Critical Constraint:**
- **Foreign Key:** `sales_order_line_items.product_id` → `product_master.product_id` (RESTRICT DELETE)
- **Row Lock:** `stock_summary` must be locked during reservation to prevent race conditions
- **No Partial Reservation:** If even one line item fails, entire SO is rolled back

---

### 3.1.2 Sales & Distribution ↔ Financial (Invoicing & GL Posting)

**Trigger:** SO Dispatch (Fulfillment complete)  
**Flow:** Dispatch → Invoice generation → GL posting (atomic)

```
Event: POST /api/v1/tax_invoices/post (Post invoice to GL)
  Input: { invoice_id: 'INV-2026-0001' }
  
  Pre-validation:
    ├─ Query tax_invoices WHERE invoice_id = 'INV-2026-0001'
    ├─ Check: status = 'DRAFT', not 'POSTED' or 'PAID'
    ├─ Check: invoice_date <= today
    ├─ Check: line items exist and amounts populated
    └─ Abort if any check fails
  
  GL Posting Transaction (Atomic Block):
    BEGIN TRANSACTION;
      
      Step 1: Post Revenue GL entry
        ├─ Calculate: revenue = SUM(line_amount) where tax excluded
        ├─ INSERT gl_entries (account_code='4010', debit=0, credit=revenue)
        ├─ UPDATE chart_of_accounts SET balance = balance + revenue WHERE account_code='4010'
        └─ Error check: balance >= 0? (credit accounts can be negative, ok)
      
      Step 2: Post GST Output GL entry
        ├─ Calculate: gst_output = SUM(tax_amount)
        ├─ INSERT gl_entries (account_code='3310', debit=0, credit=gst_output)
        ├─ UPDATE chart_of_accounts SET balance = balance + gst_output WHERE account_code='3310'
        └─ Error check: no errors
      
      Step 3: Post AR GL entry
        ├─ Calculate: total_invoice = revenue + gst_output
        ├─ INSERT gl_entries (account_code='1010', debit=total_invoice, credit=0)
        ├─ UPDATE chart_of_accounts SET balance = balance - total_invoice WHERE account_code='1010'
        └─ Error check: no errors
      
      Step 4: Post AR Subledger entry
        ├─ INSERT customer_invoices (customer_id, invoice_id, invoice_amount, status='POSTED')
        └─ UPDATE customers SET current_ar_balance = current_ar_balance + invoice_amount
      
      Step 5: Validate GL balance
        ├─ Query: SELECT SUM(balance) FROM chart_of_accounts WHERE account_type = 'DEBIT'
        ├─ Query: SELECT SUM(balance) FROM chart_of_accounts WHERE account_type = 'CREDIT'
        ├─ Check: debit_sum = credit_sum
        └─ If NOT: ROLLBACK entire transaction with error
      
      Step 6: Mark invoice as POSTED
        ├─ UPDATE tax_invoices SET status='POSTED', posted_date=NOW()
        └─ Publish event: INVOICE_POSTED (async for e-invoice queue)
    
    COMMIT; -- All or nothing

Output: { invoice_id: 'INV-2026-0001', status: 'POSTED', gl_entries_created: 3, trial_balance_valid: true }

On Failure:
  ├─ ROLLBACK triggers automatically
  ├─ All GL entries, AR entries, status updates are rolled back
  ├─ Invoice remains status = 'DRAFT'
  └─ Error message returned to user
```

**Critical Constraint:**
- **Atomicity:** Entire GL posting must succeed or fail as a unit. No partial GL posting.
- **Trial Balance:** Must validate at the end before COMMIT.
- **Foreign Key:** `tax_invoices.customer_id` → `customers.customer_id` (RESTRICT DELETE)

---

### 3.1.3 Purchase & Procurement ↔ Inventory (GRN & Stock Update)

**Trigger:** Goods Receipt (Warehouse receives goods)  
**Flow:** GRN creation → Inventory addition → stock_summary update

```
Event: POST /api/v1/grn (Create GRN and receive goods)
  Input: { po_id: 'PO-2026-0001', lines: [{ product_id: 5001, qty_received: 100 }] }
  
  Transaction: OPEN
    
    Step 1: Validate PO
      ├─ Query purchase_orders WHERE po_id = 'PO-2026-0001'
      ├─ Check: status = 'CONFIRMED' (not yet delivered)
      └─ Abort if invalid
    
    Step 2: Create GRN
      ├─ INSERT grn (grn_id, po_id, receipt_date, status='RECEIVED', ...)
      └─ Auto-generate GRN number
    
    Step 3: Create GRN line items
      ├─ INSERT grn_line_items (grn_id, po_line_id, product_id, qty_received, uom)
      └─ Validate: qty_received <= po_qty_remaining (prevent over-receipt)
    
    Step 4: Update Inventory (Atomic with Step 2-3)
      ├─ Lock stock_summary WHERE product_id = 5001
      ├─ UPDATE stock_summary SET available_qty += qty_received
      ├─ INSERT inventory_ledger (movement_type='GRN_RECEIPT', product_id, qty, reference='GRN-2026-0001')
      └─ Unlock
    
    Step 5: Update PO delivery status
      ├─ Calculate: total_delivered = SUM(qty_received from GRN for this PO)
      ├─ If total_delivered == po_qty_ordered:
      │  └─ UPDATE purchase_orders SET status='DELIVERED'
      ├─ Else if total_delivered < po_qty_ordered:
      │  └─ UPDATE purchase_orders SET status='PARTIALLY_DELIVERED'
      └─ Flag for 3-way match: ready for billing
    
    Step 6: Validate stock
      ├─ Query: SELECT available_qty FROM stock_summary WHERE product_id=5001
      ├─ Check: available_qty > 0 (stock is positive)
      └─ Alert if stock < reorder_level

  COMMIT

Output: { grn_id: 'GRN-2026-0001', po_id: 'PO-2026-0001', status: 'RECEIVED', inventory_updated: true }
```

**Critical Constraint:**
- **No Over-Receipt:** qty_received cannot exceed po_qty_remaining
- **Stock Accuracy:** stock_summary must match sum of all GRNs for a product
- **Foreign Key:** `grn.po_id` → `purchase_orders.po_id` (RESTRICT DELETE)

---

### 3.1.4 Purchase & Procurement ↔ Financial (Bill Posting & 3-Way Match)

**Trigger:** Purchase Bill receipt (Vendor invoice)  
**Flow:** Bill creation → 3-way match (PO ↔ GRN ↔ Bill) → GL posting

```
Event: POST /api/v1/purchase_bills/post (Post purchase bill to GL)
  Input: { bill_id: 'BILL-2026-0001', po_id: 'PO-2026-0001', grn_id: 'GRN-2026-0001' }
  
  Pre-validation: 3-Way Match
    ├─ Fetch PO: po_qty_ordered = 100
    ├─ Fetch GRN: qty_received = 100
    ├─ Fetch Bill: qty_invoiced = 100
    ├─ Validation 1: po_qty == grn_qty? (100 == 100?) ✓
    ├─ Validation 2: grn_qty == bill_qty? (100 == 100?) ✓
    ├─ Fetch PO rate: ₹50/unit
    ├─ Fetch Bill rate: ₹50/unit
    ├─ Validation 3: po_rate == bill_rate? (50 == 50?) ✓
    └─ Match Status: MATCHED (proceed to GL posting)
  
  If Match FAILS:
    ├─ Bill status: 'MATCH_FAILED'
    ├─ Create alert for accountant
    ├─ Abort GL posting
    └─ Return error: "PO-GRN-Bill mismatch: PO qty 100, GRN qty 100, Bill qty 50"
  
  GL Posting Transaction (Atomic):
    BEGIN TRANSACTION;
      
      Step 1: Post Inventory/Asset GL entry
        ├─ Calculate: inventory_value = qty × unit_cost
        ├─ INSERT gl_entries (account_code='1050', debit=inventory_value, credit=0)
        ├─ UPDATE chart_of_accounts SET balance -= inventory_value
        └─ (Cost of Goods Sold account)
      
      Step 2: Post GST Input GL entry
        ├─ Calculate: gst_input = inventory_value × gst_rate
        ├─ INSERT gl_entries (account_code='2050', debit=gst_input, credit=0)
        ├─ UPDATE chart_of_accounts SET balance -= gst_input
        └─ (Input Tax Credit account)
      
      Step 3: Post AP GL entry
        ├─ Calculate: total_payable = inventory_value + gst_input - tds_deducted
        ├─ INSERT gl_entries (account_code='2010', debit=0, credit=total_payable)
        ├─ UPDATE chart_of_accounts SET balance += total_payable
        └─ (Accounts Payable account)
      
      Step 4: Post TDS GL entry (if applicable)
        ├─ Calculate: tds_amount = total_payable × tds_rate (e.g., 10%)
        ├─ INSERT gl_entries (account_code='2030', debit=0, credit=tds_amount)
        ├─ UPDATE chart_of_accounts SET balance += tds_amount
        └─ (TDS Payable account)
      
      Step 5: Post AP Subledger entry
        ├─ INSERT supplier_bills (supplier_id, bill_id, bill_amount, status='POSTED')
        └─ UPDATE suppliers SET current_ap_balance += bill_amount
      
      Step 6: Validate GL balance
        ├─ Trial balance check (debits = credits)
        └─ Abort if mismatch
      
      Step 7: Mark bill as POSTED
        ├─ UPDATE purchase_bills SET status='POSTED', match_status='MATCHED'
        └─ Publish event: BILL_POSTED (async for payment due date alerts)
    
    COMMIT

Output: { bill_id: 'BILL-2026-0001', status: 'POSTED', match_status: 'MATCHED', gl_entries_created: 4 }
```

**Critical Constraints:**
- **3-Way Match Mandatory:** Bill cannot be posted without matching all three (PO, GRN, Bill)
- **Atomicity:** Entire GL posting rolls back if any step fails
- **TDS Withholding:** If supplier is TDS-applicable, deduction is mandatory

---

## 3.2 Data Flow Diagram (Complete O2C Example)

```
Customer Places Order (SO-2026-0001: 100 units Red Paint @ ₹450)
  │
  ├─→ [Inventory Module] Reserve 100 units
  │   └─ stock_summary: available_qty: 150 → 50, reserved_qty: 0 → 100
  │   └─ reservation table: 1 row created (status='RESERVED')
  │
  ├─→ [Financial Module] Create AR entry (optional, pre-invoice)
  │   └─ customers: expected_ar updated
  │   └─ (no GL posting yet, SO is not invoiced)
  │
  ├─→ [Warehouse] Warehouse picks & packs
  │   └─ SO status: PICKED → PACKED → DISPATCHED
  │   └─ Inventory still reserved until invoice
  │
  ├─→ [Sales Module] Dispatch SO
  │   └─ SO status: DISPATCHED
  │   └─ Trigger invoice generation
  │
  ├─→ [Financial Module] Create & Post Invoice
  │   ├─ tax_invoices table: 1 row (status='POSTED')
  │   ├─ GL posting (atomic):
  │   │  ├─ DR AR (1010): +₹53,100
  │   │  ├─ CR Revenue (4010): +₹45,000
  │   │  └─ CR GST Output (3310): +₹8,100
  │   ├─ customer_invoices table: 1 row (status='POSTED')
  │   ├─ customers: current_ar_balance += ₹53,100
  │   ├─ Trial balance validated (debits = credits)
  │   └─ Inventory reservation released (picked goods no longer reserved)
  │
  ├─→ [Financial Module] Generate e-Invoice
  │   ├─ Call NSDL API asynchronously
  │   ├─ IRN generated: "ABC123XYZ..."
  │   ├─ e_invoice_status: VALID
  │   └─ PDF with QR sent to customer
  │
  └─→ [Financial Module] Customer pays ₹53,100
      ├─ payment_receipts table: 1 row
      ├─ GL posting (atomic):
      │  ├─ DR Bank (1020): +₹53,100
      │  └─ CR AR (1010): -₹53,100
      ├─ customer_invoices: status = PAID
      ├─ customers: current_ar_balance -= ₹53,100
      └─ Invoice lifecycle complete (DRAFT → POSTED → PAID)
```

---

# 4. Database Binding & Foreign Key Contracts

## 4.1 Foreign Key Relationships (By Module)

### Inventory Module

```sql
-- Product Master
CREATE TABLE product_master (
    product_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    product_code VARCHAR(50) UNIQUE,
    product_name VARCHAR(255),
    hsn_code CHAR(6),
    tax_rate DECIMAL(5,2),
    reorder_level INT,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (hsn_code) REFERENCES compliance_parameters(hsn_code) ON DELETE RESTRICT
);

-- Stock Summary (Real-time inventory by warehouse)
CREATE TABLE stock_summary (
    stock_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    warehouse_id BIGINT,
    available_qty INT,
    reserved_qty INT,
    allocated_qty INT,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id) ON DELETE RESTRICT,
    CONSTRAINT chk_qty_positive CHECK (available_qty >= 0 AND reserved_qty >= 0)
);

-- Inventory Ledger (Audit trail)
CREATE TABLE inventory_ledger (
    ledger_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    movement_type VARCHAR(50), -- 'GRN_RECEIPT', 'RESERVATION', 'SALE', 'RETURN', etc.
    qty_moved INT,
    reference_type VARCHAR(50), -- 'SO', 'GRN', 'POS', etc.
    reference_id VARCHAR(50),
    movement_date TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT
);

-- Reservation (Links SO to inventory)
CREATE TABLE reservations (
    reservation_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    so_id VARCHAR(50),
    product_id BIGINT NOT NULL,
    qty_reserved INT,
    warehouse_id BIGINT,
    status VARCHAR(50), -- 'RESERVED', 'FULFILLED', 'CANCELLED'
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT,
    FOREIGN KEY (warehouse_id) REFERENCES warehouses(warehouse_id) ON DELETE RESTRICT
);
```

### Sales & Distribution Module

```sql
-- Sales Order
CREATE TABLE sales_orders (
    so_id VARCHAR(50) PRIMARY KEY,
    company_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    quotation_id VARCHAR(50),
    order_date DATE,
    delivery_date DATE,
    status VARCHAR(50), -- 'DRAFT', 'CONFIRMED', 'PICKED', 'PACKED', 'DISPATCHED'
    total_amount DECIMAL(15,2),
    total_tax DECIMAL(15,2),
    grand_total DECIMAL(15,2),
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,
    FOREIGN KEY (quotation_id) REFERENCES quotations(quote_id) ON DELETE SET NULL
);

-- Sales Order Line Items
CREATE TABLE sales_order_line_items (
    line_id BIGINT PRIMARY KEY,
    so_id VARCHAR(50) NOT NULL,
    product_id BIGINT NOT NULL,
    qty_ordered INT,
    qty_reserved INT DEFAULT 0,
    qty_fulfilled INT DEFAULT 0,
    qty_invoiced INT DEFAULT 0,
    unit_price DECIMAL(15,2),
    tax_rate DECIMAL(5,2),
    line_amount DECIMAL(15,2),
    FOREIGN KEY (so_id) REFERENCES sales_orders(so_id) ON DELETE RESTRICT,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT
);

-- Tax Invoice
CREATE TABLE tax_invoices (
    invoice_id VARCHAR(50) PRIMARY KEY,
    company_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    so_id VARCHAR(50),
    invoice_date DATE,
    status VARCHAR(50), -- 'DRAFT', 'POSTED', 'SENT', 'PAID'
    total_amount DECIMAL(15,2),
    total_tax DECIMAL(15,2),
    grand_total DECIMAL(15,2),
    irn VARCHAR(64), -- e-invoice IRN
    e_invoice_status VARCHAR(50), -- 'PENDING', 'VALID', 'FAILED'
    posted_date TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,
    FOREIGN KEY (so_id) REFERENCES sales_orders(so_id) ON DELETE SET NULL
);

-- Tax Invoice Line Items
CREATE TABLE tax_invoice_line_items (
    line_id BIGINT PRIMARY KEY,
    invoice_id VARCHAR(50) NOT NULL,
    product_id BIGINT NOT NULL,
    qty INT,
    unit_price DECIMAL(15,2),
    tax_rate DECIMAL(5,2),
    line_amount DECIMAL(15,2),
    tax_amount DECIMAL(15,2),
    FOREIGN KEY (invoice_id) REFERENCES tax_invoices(invoice_id) ON DELETE RESTRICT,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT
);
```

### Financial Module

```sql
-- General Ledger Entries
CREATE TABLE gl_entries (
    entry_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    account_code VARCHAR(10),
    debit DECIMAL(15,2) DEFAULT 0,
    credit DECIMAL(15,2) DEFAULT 0,
    reference_type VARCHAR(50), -- 'INV', 'BILL', 'JV', etc.
    reference_id VARCHAR(50),
    entry_date TIMESTAMP,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (account_code) REFERENCES chart_of_accounts(account_code) ON DELETE RESTRICT,
    CONSTRAINT chk_debit_or_credit CHECK ((debit > 0 AND credit = 0) OR (credit > 0 AND debit = 0))
);

-- Chart of Accounts
CREATE TABLE chart_of_accounts (
    account_code VARCHAR(10) PRIMARY KEY,
    company_id BIGINT,
    account_name VARCHAR(255),
    account_type VARCHAR(50), -- 'ASSET', 'LIABILITY', 'EQUITY', 'REVENUE', 'EXPENSE'
    balance DECIMAL(15,2) DEFAULT 0,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT
);

-- AR Subledger (Customer-level detail)
CREATE TABLE customer_invoices (
    customer_invoice_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    invoice_id VARCHAR(50),
    invoice_amount DECIMAL(15,2),
    status VARCHAR(50), -- 'POSTED', 'PAID', 'OVERDUE'
    invoice_date DATE,
    due_date DATE,
    paid_date DATE,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,
    FOREIGN KEY (invoice_id) REFERENCES tax_invoices(invoice_id) ON DELETE RESTRICT
);

-- AP Subledger (Supplier-level detail)
CREATE TABLE supplier_bills (
    supplier_bill_id BIGINT PRIMARY KEY,
    company_id BIGINT NOT NULL,
    supplier_id BIGINT NOT NULL,
    bill_id VARCHAR(50),
    bill_amount DECIMAL(15,2),
    status VARCHAR(50), -- 'POSTED', 'PAID', 'OVERDUE'
    bill_date DATE,
    due_date DATE,
    paid_date DATE,
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id) ON DELETE RESTRICT,
    FOREIGN KEY (bill_id) REFERENCES purchase_bills(bill_id) ON DELETE RESTRICT
);
```

### Purchase & Procurement Module

```sql
-- Purchase Order
CREATE TABLE purchase_orders (
    po_id VARCHAR(50) PRIMARY KEY,
    company_id BIGINT NOT NULL,
    supplier_id BIGINT NOT NULL,
    po_date DATE,
    delivery_date DATE,
    status VARCHAR(50), -- 'DRAFT', 'CONFIRMED', 'PARTIALLY_DELIVERED', 'DELIVERED'
    total_amount DECIMAL(15,2),
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id) ON DELETE RESTRICT
);

-- PO Line Items
CREATE TABLE po_line_items (
    po_line_id BIGINT PRIMARY KEY,
    po_id VARCHAR(50) NOT NULL,
    product_id BIGINT NOT NULL,
    qty_ordered INT,
    qty_received INT DEFAULT 0,
    unit_cost DECIMAL(15,2),
    tax_rate DECIMAL(5,2),
    FOREIGN KEY (po_id) REFERENCES purchase_orders(po_id) ON DELETE RESTRICT,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT
);

-- Goods Receipt Note
CREATE TABLE grn (
    grn_id VARCHAR(50) PRIMARY KEY,
    company_id BIGINT NOT NULL,
    po_id VARCHAR(50),
    receipt_date DATE,
    status VARCHAR(50), -- 'RECEIVED', 'INSPECTED', 'ACCEPTED'
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (po_id) REFERENCES purchase_orders(po_id) ON DELETE SET NULL
);

-- GRN Line Items
CREATE TABLE grn_line_items (
    grn_line_id BIGINT PRIMARY KEY,
    grn_id VARCHAR(50) NOT NULL,
    po_line_id BIGINT,
    product_id BIGINT NOT NULL,
    qty_received INT,
    FOREIGN KEY (grn_id) REFERENCES grn(grn_id) ON DELETE RESTRICT,
    FOREIGN KEY (po_line_id) REFERENCES po_line_items(po_line_id) ON DELETE SET NULL,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT
);

-- Purchase Bill (3-way match)
CREATE TABLE purchase_bills (
    bill_id VARCHAR(50) PRIMARY KEY,
    company_id BIGINT NOT NULL,
    supplier_id BIGINT NOT NULL,
    po_id VARCHAR(50),
    grn_id VARCHAR(50),
    bill_date DATE,
    status VARCHAR(50), -- 'DRAFT', 'POSTED', 'MATCHED', 'PAID'
    match_status VARCHAR(50), -- 'MATCHED', 'MATCH_FAILED', 'PENDING'
    FOREIGN KEY (company_id) REFERENCES companies(company_id) ON DELETE RESTRICT,
    FOREIGN KEY (supplier_id) REFERENCES suppliers(supplier_id) ON DELETE RESTRICT,
    FOREIGN KEY (po_id) REFERENCES purchase_orders(po_id) ON DELETE SET NULL,
    FOREIGN KEY (grn_id) REFERENCES grn(grn_id) ON DELETE SET NULL
);

-- Bill Line Items
CREATE TABLE bill_line_items (
    bill_line_id BIGINT PRIMARY KEY,
    bill_id VARCHAR(50) NOT NULL,
    grn_line_id BIGINT,
    product_id BIGINT NOT NULL,
    qty_invoiced INT,
    unit_cost DECIMAL(15,2),
    tax_rate DECIMAL(5,2),
    FOREIGN KEY (bill_id) REFERENCES purchase_bills(bill_id) ON DELETE RESTRICT,
    FOREIGN KEY (grn_line_id) REFERENCES grn_line_items(grn_line_id) ON DELETE SET NULL,
    FOREIGN KEY (product_id) REFERENCES product_master(product_id) ON DELETE RESTRICT
);
```

---

# 5. Transaction Atomicity & Consistency Guarantees

## 5.1 ACID Properties Enforced

### Atomicity: All-or-Nothing Guarantee

**Principle:** A transaction either completes fully or rolls back entirely. No partial updates.

**Example: SO Creation with Inventory Reservation**

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

  -- Insert SO
  INSERT INTO sales_orders (so_id, customer_id, status) VALUES ('SO-2026-0001', 101, 'DRAFT');
  
  -- Insert SO line items
  INSERT INTO sales_order_line_items (so_id, product_id, qty_ordered, unit_price) 
  VALUES ('SO-2026-0001', 5001, 100, 450);
  
  -- Lock and reserve inventory
  LOCK TABLE stock_summary IN EXCLUSIVE MODE;
  
  SELECT available_qty FROM stock_summary 
  WHERE product_id = 5001 FOR UPDATE; -- Row-level lock
  
  -- Check stock
  IF available_qty >= 100 THEN
    -- Deduct from available, add to reserved
    UPDATE stock_summary 
    SET available_qty = available_qty - 100,
        reserved_qty = reserved_qty + 100
    WHERE product_id = 5001;
    
    -- Insert reservation record
    INSERT INTO reservations (so_id, product_id, qty_reserved, status) 
    VALUES ('SO-2026-0001', 5001, 100, 'RESERVED');
    
    -- Update SO status
    UPDATE sales_orders SET status = 'CONFIRMED' WHERE so_id = 'SO-2026-0001';
    
    COMMIT; -- All changes persisted
  ELSE
    ROLLBACK; -- Entire transaction undone, SO + line items + attempted reservation deleted
    RAISE EXCEPTION 'Insufficient inventory';
  END IF;
```

**Guarantee:** Either SO is created with inventory reserved, OR nothing happens. No partial state.

---

### Consistency: Database Integrity

**Principle:** All constraints (PK, FK, CHECK) enforced before COMMIT. Trial balance always balances.

**Example: Invoice Posting Must Balance GL**

```sql
BEGIN TRANSACTION;

  -- Fetch invoice totals
  SELECT 
    SUM(line_amount) as total_revenue,
    SUM(tax_amount) as total_tax
  INTO revenue_amt, tax_amt
  FROM tax_invoice_line_items 
  WHERE invoice_id = 'INV-2026-0001';
  
  total_invoice = revenue_amt + tax_amt;
  
  -- Post GL entries
  INSERT INTO gl_entries VALUES (entry1, 'INV-2026-0001', '1010', total_invoice, 0);   -- DR AR
  INSERT INTO gl_entries VALUES (entry2, 'INV-2026-0001', '4010', 0, revenue_amt);     -- CR Revenue
  INSERT INTO gl_entries VALUES (entry3, 'INV-2026-0001', '3310', 0, tax_amt);         -- CR GST Output
  
  -- Validate trial balance BEFORE committing
  SELECT 
    SUM(CASE WHEN debit > 0 THEN debit ELSE 0 END) as total_debits,
    SUM(CASE WHEN credit > 0 THEN credit ELSE 0 END) as total_credits
  INTO total_dr, total_cr
  FROM gl_entries 
  WHERE company_id = 1;
  
  IF total_dr != total_cr THEN
    ROLLBACK;
    RAISE EXCEPTION 'GL does not balance: DR=%s, CR=%s', total_dr, total_cr;
  END IF;
  
  -- GL balances, proceed
  UPDATE tax_invoices SET status = 'POSTED' WHERE invoice_id = 'INV-2026-0001';
  COMMIT;
```

**Guarantee:** Invoice is only posted if GL balances. Unbalanced entry is rejected, transaction rolled back.

---

### Isolation: No Dirty Reads, No Lost Updates

**Principle:** Concurrent transactions don't interfere with each other. Using SERIALIZABLE isolation level for critical transactions.

**Example: Concurrent SO Creation (No Oversale)**

```
Scenario:
  Available inventory: 100 units
  User A attempts to create SO for 80 units
  User B attempts to create SO for 50 units
  
Transaction A:
  BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT available_qty FROM stock_summary WHERE product_id = 5001 FOR UPDATE;
    -- Locked, A sees available_qty = 100
    
    CHECK: 100 >= 80? YES
    
    UPDATE stock_summary SET available_qty = 20, reserved_qty = 80;
    INSERT INTO sales_orders (so_id='SO-A', qty=80);
  COMMIT; -- A finishes
  
Transaction B (starts while A is running):
  BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT available_qty FROM stock_summary WHERE product_id = 5001 FOR UPDATE;
    -- BLOCKS (waiting for A's lock)
    
    -- A commits, B acquires lock
    -- B now sees available_qty = 20 (updated by A)
    
    CHECK: 20 >= 50? NO
    ROLLBACK; -- B's SO rejected, no data changes
```

**Guarantee:** Only A's SO is created (80 units). B's SO is rejected due to insufficient stock. No oversale, no race condition.

---

### Durability: Committed Changes Persist

**Principle:** Once COMMIT is issued, data is durable even if system crashes.

**Example: Invoice Posted, then System Crash**

```
Step 1: Invoice posting begins
  ├─ GL entries inserted (DR AR, CR Rev, CR GST)
  ├─ AR subledger updated
  ├─ Trial balance validated ✓
  └─ COMMIT issued

Step 2: Commit lands in database transaction log (persistent storage)

Step 3: System crashes (power loss, network error, etc.)
  └─ Invoice status already changed to POSTED (in database commit log)

Step 4: System recovers
  ├─ Database reads commit log
  ├─ Sees: Invoice INV-2026-0001 was POSTED before crash
  ├─ Verifies: GL entries exist, AR subledger correct
  └─ No re-posting needed (data persisted durably)
```

**Guarantee:** Once COMMIT succeeds, changes persist. No data loss even on system failure.

---

## 5.2 Critical Atomic Transactions (By Use Case)

| Use Case | Transaction Scope | Rollback Trigger |
|----------|-------------------|-----------------|
| **SO Creation** | Create SO + line items + reserve inventory | Insufficient stock, customer inactive, credit limit exceeded |
| **Invoice Posting** | Insert GL entries + AR subledger + update CoA + mark invoice POSTED | Trial balance doesn't balance, FK constraint violated, account type mismatch |
| **GRN Receipt** | Create GRN + update stock_summary + update PO status | qty_received > po_qty_remaining, warehouse lock timeout |
| **Bill Posting** | 3-way match validation + GL entries (Inventory, GST Input, AP) + AP subledger + TDS calculation | Qty mismatch (PO 100, GRN 100, Bill 50), GL posting fails, AP balance calculation error |
| **Payment Receipt** | Create payment_receipts + GL entries (DR Bank, CR AR) + mark invoice PAID + update AR balance | Payment amount ≠ invoice amount, invoice not in POSTED status, bank account GL invalid |
| **POS Sale** | Deduct inventory + create pos_transactions + GL posting (async) | Inventory insufficient, customer credit limit exceeded, daily limit exceeded |

---

# 6. State Machines & Workflow Lifecycles

## 6.1 Sales Order State Machine

```
┌────────┐
│ DRAFT  │ ← Created by sales rep
└───┬────┘
    │ (Confirm: credit check + inventory check)
    │ Pre-checks:
    │ ├─ Customer active?
    │ ├─ Credit available?
    │ └─ Inventory sufficient?
    ↓
┌───────────────────┐
│ CONFIRMED         │ ← Inventory reserved, ready for fulfillment
└───┬───────────────┘
    │ (Warehouse picks items)
    ↓
┌───────────────────┐
│ PICKED            │ ← Items physically picked from warehouse
└───┬───────────────┘
    │ (Items packed in box)
    ↓
┌───────────────────┐
│ PACKED            │ ← Ready for dispatch
└───┬───────────────┘
    │ (Items shipped to customer)
    ↓
┌───────────────────┐
│ DISPATCHED        │ ← In transit to customer
└───┬───────────────┘
    │ (Trigger invoice)
    ↓
┌───────────────────┐
│ INVOICED          │ ← Invoice created, GL posted
└───┬───────────────┘
    │ (Customer pays)
    ↓
┌───────────────────┐
│ PAID              │ ← [TERMINAL] Order complete
└───────────────────┘

Cancellation Path (from DRAFT or CONFIRMED):
  ├─ SO status = 'CANCELLED'
  ├─ If CONFIRMED: Release inventory reservation
  └─ [TERMINAL]
```

**State Transition Rules:**
- DRAFT → CONFIRMED: Mandatory credit + stock checks
- CONFIRMED → PICKED: Warehouse confirmation required
- PICKED → PACKED: No validation (internal operation)
- PACKED → DISPATCHED: No validation (shipping confirmation)
- DISPATCHED → INVOICED: Automatic (invoice generation triggered)
- INVOICED → PAID: Payment matched to invoice

---

## 6.2 Tax Invoice State Machine

```
┌────────┐
│ DRAFT  │ ← Created from SO dispatch, not yet posted to GL
└───┬────┘
    │ (Validate: trial balance check)
    ↓
┌──────────┐
│ POSTED   │ ← GL entries created, trial balance validated
└───┬──────┘
    │ (Queue for e-invoice)
    │ (Invoice sent to customer)
    ↓
┌──────────┐
│ SENT     │ ← Awaiting payment
└───┬──────┘
    │ (Payment received)
    ↓
┌──────────┐
│ PAID     │ ← [TERMINAL] Revenue recognized, AR cleared
└──────────┘

Error Paths:
  DRAFT →→→ ERROR: Validation failed (trial balance, FK constraint)
  POSTED → PARTIAL_PAID: Payment < invoice amount (dunning cycle)
  PAID → REVERSED: Credit note issued (return handling)
```

**State Transition Rules:**
- DRAFT → POSTED: Validation + GL balance check mandatory
- POSTED → SENT: Invoice marked as sent to customer
- SENT → PAID: Payment matched (exact amount or partial)
- PAID → REVERSED: Manual reversal via credit note (audit trail maintained)

---

## 6.3 Purchase Order State Machine

```
┌────────┐
│ DRAFT  │ ← Created by procurement, awaiting approval
└───┬────┘
    │ (Approval check)
    ↓
┌───────────────────┐
│ CONFIRMED         │ ← Supplier acknowledged, GRN ready to be created
└───┬───────────────┘
    │ (Goods delivered, GRN created)
    ↓
┌────────────────────────────┐
│ PARTIALLY_DELIVERED        │ ← Part of order received (qty < po_qty)
└───┬────────────────────────┘
    │ (If remaining qty < reorder level or no reorder: finalize)
    ↓
┌───────────────────┐
│ DELIVERED         │ ← All goods received (qty = po_qty)
└───┬───────────────┘
    │ (Bill matched to PO + GRN)
    ↓
┌───────────────────┐
│ BILLED            │ ← Bill posted, AP recorded, ready for payment
└───┬───────────────┘
    │ (Payment made)
    ↓
┌───────────────────┐
│ PAID              │ ← [TERMINAL] Procurement complete
└───────────────────┘

Cancellation Path (from DRAFT or CONFIRMED):
  ├─ PO status = 'CANCELLED'
  ├─ If CONFIRMED: Alert supplier (manual email/phone)
  └─ [TERMINAL]
```

**State Transition Rules:**
- DRAFT → CONFIRMED: Procurement approval required
- CONFIRMED → PARTIALLY_DELIVERED: GRN creation (qty > 0, < po_qty)
- PARTIALLY_DELIVERED → DELIVERED: Remaining qty received
- DELIVERED → BILLED: Bill matched (PO ↔ GRN ↔ Bill)
- BILLED → PAID: Payment processed

---

## 6.4 Goods Receipt Note (GRN) State Machine

```
┌──────────┐
│ RECEIVED │ ← Created from PO, goods physically received
└────┬─────┘
     │ (Warehouse inspects goods)
     ↓
┌───────────┐
│ INSPECTED │ ← Quality check passed, goods accepted
└────┬──────┘
     │ (Goods accepted, inventory updated)
     ↓
┌────────────┐
│ ACCEPTED   │ ← [TERMINAL] Goods added to stock, ready for billing
└────────────┘

Rejection Path (from RECEIVED):
  ├─ Goods quality check failed
  ├─ GRN status = 'REJECTED'
  ├─ Inventory NOT updated (stock remains unchanged)
  ├─ Alert supplier: Return required
  └─ [TERMINAL]
```

**State Transition Rules:**
- RECEIVED → INSPECTED: Warehouse QC validation
- INSPECTED → ACCEPTED: Goods accepted, stock updated
- RECEIVED → REJECTED: Defects found, no stock update, return initiated

---

# 7. Soft-Launch Readiness & Validation Gates

## 7.1 Pre-Launch Validation Checklist (12 Critical Gates)

| # | Gate | Acceptance Criteria | Owner | Risk if Failed |
|---|------|-------------------|-------|----------------|
| **1** | Inventory Accuracy | Stock-on-hand matches ledger within ±1 unit; FIFO valuation spot-checked on 10 transactions | Inventory Lead | Oversale, stock discrepancy, revenue recognition error |
| **2** | GL Posting Atomicity | 50+ test invoices posted; trial balance validates (0 variance); partial post never occurs | Finance Lead | Unbalanced GL, revenue recognition error, tax misstatement |
| **3** | AR/AP Reconciliation | AR subledger = GL account 1010 (exact ₹ match); AP subledger = GL account 2010 (exact ₹ match) | Finance Lead | AR/AP aging misstatement, DSO calculation error |
| **4** | GST Subledger Accuracy | GST Output by HSN matches invoices; GST Input matches bills; GSTR-1 auto-pop validated on 5 invoices | Finance Lead | GST filing error, tax compliance risk |
| **5** | 3-Way Match | 10 PO-GRN-Bill cycles: PO qty = GRN qty = Bill qty; bill posting blocked if mismatch | Procurement Lead | Duplicate payment, under-billing, bill discrepancy |
| **6** | Inventory Reservation (Concurrent) | 5 simultaneous SOs for same product (100 units available); no oversale, correct allocation | Inventory Lead | Oversale, customer order breach, refund liability |
| **7** | e-Invoice IRN Generation | 10+ invoices sent to NSDL; 90%+ IRNs generated successfully; retry logic tested (API timeout scenario) | Finance Lead | e-invoice failure, late filing penalty |
| **8** | Daily GL Reconciliation | Automated daily TB generated; variances logged; 0 unmatched entries for 7 consecutive days | Finance Lead | Manual reconciliation burden, GL integrity risk |
| **9** | Multi-Tenant Isolation | 2 tenants create identical transactions; data verified isolated per company_id RLS; no cross-tenant reads | Backend Lead | Data breach, customer confidentiality violation |
| **10** | POS Inventory Sync | 50 POS sales; stock deducted correctly; GL posting matches POS revenue; daily reconciliation within ±₹5 | POS Lead | Inventory discrepancy, revenue misstatement |
| **11** | Payment Receipt Matching | 10 invoices paid: payment amount = invoice amount; GL entries (DR Bank, CR AR) posted atomically; invoice marked PAID | Finance Lead | Over-/under-payment, AR balance error |
| **12** | Rollback & Error Handling | 3 failure scenarios tested: GL posting fails → entire transaction rolls back; no orphaned records or partial posts | Backend Lead | Data corruption, inconsistent state, unrecoverable error |

---

## 7.2 Go-Live Validation Matrix

### Performance Benchmarks (Must Meet)

| Metric | Target | Acceptable | Critical |
|--------|--------|-----------|----------|
| SO Creation (incl. inventory reservation) | <1s | <2s | >2s ❌ BLOCK |
| Invoice Posting (GL + AR update) | <500ms | <1s | >1s ❌ BLOCK |
| Payment Receipt (GL posting) | <500ms | <1s | >1s ❌ BLOCK |
| GRN Creation (stock update) | <500ms | <1s | >1s ❌ BLOCK |
| API Response (99th percentile) | <2s | <3s | >3s ❌ BLOCK |
| Daily Reconciliation (1,000 txns) | <5min | <10min | >10min ⚠️ WARN |

### Data Integrity Checks (Must Pass 100%)

| Check | Pass Criteria | Failure Action |
|-------|---------------|----------------|
| **GL Trial Balance** | Sum(Debits) = Sum(Credits) for all entries | Block invoice posting, manual audit |
| **AR Subledger Reconciliation** | customer_invoices.sum = GL account 1010 | Block payment receipt, manual audit |
| **AP Subledger Reconciliation** | supplier_bills.sum = GL account 2010 | Block bill posting, manual audit |
| **Inventory Accuracy** | stock_summary matches inventory_ledger | Block SO creation, physical recount |
| **FK Integrity** | No orphaned records (all FKs resolve to parent) | Database repair, manual cleanup |
| **GST Subledger** | GST Output by HSN = invoices; GST Input = bills | Block GSTR-1 filing, manual verification |

---

## 7.3 Hard Go-Live Gates

### Must-Pass Criteria (Non-Negotiable)

**BLOCK Launch if any of these fail:**

1. ❌ **GL Trial Balance unbalanced** (any variance > ₹1)
   - Action: Audit all GL entries, identify discrepancy, rollback offending transaction
   
2. ❌ **AR/AP subledger mismatches GL** (variance > ₹1)
   - Action: Manual reconciliation, reverse and re-post affected invoices
   
3. ❌ **Inventory Oversale occurs** (any single incident of selling > stock)
   - Action: Manual inventory recount, identify root cause, prevent future via code review
   
4. ❌ **e-Invoice IRN fails on 10%+ of invoices** (success rate < 90%)
   - Action: Verify NSDL API credentials, test connectivity, resolve before launch
   
5. ❌ **Multi-Tenant Isolation breach** (any cross-tenant data read)
   - Action: Security audit, RLS policy review, penetration test, fix before launch
   
6. ❌ **API Response time > 3 seconds** (99th percentile)
   - Action: Performance profiling, database optimization, caching strategy review
   
7. ❌ **Concurrent Transaction Deadlock** (any deadlock detected during load test)
   - Action: Query analysis, transaction isolation review, lock ordering fix

---

## 7.4 Soft-Launch Success Metrics (Post Go-Live)

### First 4 Weeks of Pilot (3-5 SMB Customers)

| Metric | Target | Acceptable | Action if Miss |
|--------|--------|-----------|----------------|
| **Uptime** | 99.9% | 99% | Alert on-call engineer, RCA |
| **Critical Error Rate** | 0 | <0.1% | Immediate fix, customer notification |
| **Manual Workarounds** | 0 | <5% | Feature refinement for Phase 2 |
| **Data Reconciliation Variance** | ±₹0 | ±₹5/day | Manual audit, process improvement |
| **Customer Activation Rate** | 100% | 80% | Onboarding process review |
| **Module Usage** | All 5 modules used | 4/5 modules | Feature education, in-app tips |

---

# 8. Go-Live Checklist & First Customer Onboarding

## 8.1 Pre-Launch (T-1 Week)

- [ ] All 12 validation gates passed
- [ ] Performance benchmarks met (SO <2s, Invoice <1s, Payment <1s)
- [ ] GL Trial Balance reconciled to zero variance
- [ ] AR/AP subledgers match GL (₹ exact match)
- [ ] GST subledger validated for 10 invoices
- [ ] 3-Way Match tested on 10 PO-GRN-Bill cycles
- [ ] Inventory accuracy verified within ±1 unit
- [ ] e-Invoice IRN success rate 90%+
- [ ] Multi-Tenant isolation verified (penetration test)
- [ ] Daily Reconciliation running successfully (7 days)
- [ ] Concurrent transaction testing passed (no deadlocks)
- [ ] Backup & disaster recovery tested
- [ ] On-call support schedule finalized
- [ ] Customer training completed (first 3 pilot SMBs)
- [ ] API documentation reviewed & finalized
- [ ] Error message clarity verified (customer-facing text)

## 8.2 Go-Live Day (T+0)

### Hour 1: System Handoff
- [ ] Production environment verified (no errors in logs)
- [ ] Database backups confirmed
- [ ] Monitoring alerts active (GL imbalance, API errors, inventory discrepancy)
- [ ] On-call engineer standing by

### Hour 2-4: First Customer Onboarding

**Customer Profile:** Kerala Trading Co (Wholesale trader)

**Onboarding Steps:**
1. **Sign-up & Tenant Provisioning** (Part 4)
   - Email: owner@keraltrading.com
   - Password set, SSO verified
   - Tenant created in <2 seconds
   - Setup checklist displayed

2. **Master Data Seeding**
   - GSTIN: Added manually (optional for first invoice)
   - State: Kerala (pre-filled from sign-up)
   - Products: 50 test products uploaded via CSV
   - Customers: 10 test customers imported

3. **First Transaction Walkthrough**
   - Sales Rep creates test SO (100 units @ ₹450)
   - Inventory reserved (visible in checklist)
   - Warehouse picks & packs
   - Invoice generated, GL posted (verify in Finance dashboard)
   - Payment received (GL bank account updated)
   - Full cycle time logged

4. **Verification Checklist**
   - [ ] SO created, status = CONFIRMED
   - [ ] Inventory reserved (stock_summary shows correct qty)
   - [ ] Invoice posted (GL entries created, trial balance 0 variance)
   - [ ] AR subledger updated (customer_invoices row exists)
   - [ ] e-Invoice IRN generated (customer receives PDF with QR)
   - [ ] Payment receipt posted (GL bank account updated)
   - [ ] All GL accounts balance (trial balance reconciles)

5. **Customer Training**
   - [ ] Dashboard walk-through (Sales, Inventory, Finance dashboards)
   - [ ] Core workflows explained (O2C, P2P)
   - [ ] Support contact info provided (email, WhatsApp)
   - [ ] On-call engineer introduced

### Hour 5+: Continuous Monitoring

- [ ] Monitor all GL entries hourly (check for imbalances)
- [ ] Monitor AR/AP reconciliation (hourly comparison to GL)
- [ ] Monitor API error rates (target: < 0.1%)
- [ ] Monitor concurrent user load (target: < 10 concurrent users in pilot)
- [ ] Check daily reconciliation runs (no unmatched entries)
- [ ] Check e-invoice processing (NSDL API status, IRN generation)

---

## 8.3 Post-Launch (T+7 Days)

### Week 1 Validation

- [ ] All 3-5 pilot customers successfully onboarded
- [ ] 50+ transactions processed (SOs, Invoices, Payments, GRNs)
- [ ] GL Trial Balance maintained (0 variance)
- [ ] AR/AP subledgers match GL (exact ₹ reconciliation)
- [ ] No critical errors (0 unhandled exceptions)
- [ ] No data corruption (spot-check 10 invoices, 10 POs)
- [ ] Performance stable (response times consistent with benchmarks)
- [ ] Uptime 99.9%+ (no unplanned downtime)

### Week 1-2 Reporting

- [ ] Weekly financial summary report (revenue, AP, AR, GL balances)
- [ ] Inventory accuracy report (stock discrepancies)
- [ ] System health report (error rates, API latency, database performance)
- [ ] Customer feedback (usability, feature gaps, support quality)

### Week 2 Decision: Expand to Phase 2

**Go/No-Go Criteria:**
- ✅ **GO:** All validation gates holding, customers satisfied, no critical bugs
- ⚠️ **CAUTION:** Minor issues found, hotfix deployed, monitoring for recurrence
- ❌ **STOP:** Critical issue (GL imbalance, data corruption, security breach) → rollback, root cause analysis, fix required

---

## Conclusion

This comprehensive implementation sequence ensures Niyanthra's soft launch succeeds with:

✅ **Strict dependency ordering** — modules build in correct sequence, no blocked dependencies  
✅ **Atomic transactions** — no partial posts, GL always balances, inventory never oversells  
✅ **Hard validation gates** — 12 non-negotiable criteria verified before pilot launch  
✅ **Detailed workflows** — every state transition, error path, and recovery scenario documented  
✅ **Performance benchmarks** — response times tracked, tail latencies managed  
✅ **First-customer readiness** — onboarding checklist, verification steps, 24/7 support  

**Target: Launch with 3-5 pilot SMBs by Week 12, scale to 20+ customers by Month 6.**

---

**Document Control**

| | |
|---|---|
| **Version** | 1.0 (Production Release) |
| **Status** | Ready for Engineering Intake |
| **Audience** | Engineering Lead, Database Architects, QA Lead, Implementation Manager |
| **Last Updated** | September 2026 |
| **Related Docs** | Part 1 (Workflow & Interconnectivity), Part 3 (Signup & Onboarding UX), Part 2 (Functional Module & Data Architecture) |
