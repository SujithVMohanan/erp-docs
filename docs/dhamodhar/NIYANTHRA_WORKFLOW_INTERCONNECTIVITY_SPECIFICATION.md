# 🔄 Niyanthra ERP — Workflow & Interconnectivity Specification

**Version:** 1.0 (Production Edition)  
**Date:** September 2026  
**Classification:** INTERNAL — Engineering & Implementation Use  
**Audience:** Backend Engineering Team, Database Architects, Implementation Leads, Product Managers

---

## Table of Contents

1. [Executive Overview](#executive-overview)
2. [Core ERP Workflows](#core-erp-workflows)
3. [Module Interconnectivity & Database Binding](#module-interconnectivity--database-binding)
4. [State Machines & Workflow Transitions](#state-machines--workflow-transitions)
5. [Atomic Transactions & Consistency Guarantees](#atomic-transactions--consistency-guarantees)
6. [Soft-Launch Critical Path & Dependencies](#soft-launch-critical-path--dependencies)
7. [API Contract Specifications](#api-contract-specifications)
8. [Error Handling & Rollback Scenarios](#error-handling--rollback-scenarios)

---

## Executive Overview

Niyanthra's soft launch targets **3-5 regional SMBs** (trading/wholesale/distribution) across Kerala with **MVP scope:**
- **Sales & Distribution:** Quotations → Orders → Invoices → Payments
- **Purchase & Inventory:** Requisitions → POs → GRNs → Stock → Reorder alerts
- **Point of Sale:** Retail counter billing, real-time inventory sync
- **Financial Accounting:** Real-time GL posting, GSTR-1 auto-population
- **CRM:** Customer profiles, order history, repeat tracking

**This specification ensures:**
1. ✅ **Workflow choreography** (5 core lifecycles, step-by-step)
2. ✅ **Database atomicity** (transactions span multiple modules without data loss)
3. ✅ **State consistency** (invoices can't be paid twice, orders can't be over-fulfilled)
4. ✅ **Real-time synchronization** (inventory reserved instantly when SO created)
5. ✅ **Tax compliance** (GST postings automatic, TDS deducted, e-invoices generated)

---

# PART 1: CORE ERP WORKFLOWS

---

## Workflow 1: Order-to-Cash (O2C)

### 1.1 Complete Lifecycle Overview

```
Quotation Phase
    ↓
Sales Order Phase (Inventory Reserved)
    ↓
Fulfillment Phase (Pick → Pack → Dispatch)
    ↓
Invoicing Phase (Tax Invoice + e-Invoice + GL Posted)
    ↓
Payment Phase (Collection → Reconciliation)
    ↓
Closure Phase (Revenue recognized, AR aged)
```

---

### 1.2 Step-by-Step Workflow with State Transitions

#### **Phase 1: Quotation (T+0 days)**

```
ACTORS: Sales Rep, Customer
EVENT: Customer inquiry for "Red Paint 1L × 100 units"

STEP 1: Sales Rep creates Quotation
├─ Input:
│   ├─ Customer selection (from master)
│   ├─ Product selection (Red Paint 1L from product DB)
│   ├─ Quantity (100 units)
│   ├─ Pricing (from product master: ₹450/unit)
│   ├─ Tax rate (from HSN code: 18% GST)
│   └─ Terms (Net 30 days credit)
│
├─ System Processing:
│   ├─ Fetch product: product_id = 5001, hsn_code = '3208'
│   ├─ Calculate line value: 100 × ₹450 = ₹45,000
│   ├─ Calculate tax: ₹45,000 × 18% = ₹8,100
│   ├─ Total quote: ₹53,100
│   └─ Check inventory (informational only, no reservation)
│
├─ Database Insert:
│   ├─ quotations table:
│   │   ├─ quote_id = QT-2026-0001 (auto-generated)
│   │   ├─ company_id = 1
│   │   ├─ customer_id = 101
│   │   ├─ quote_date = 2026-09-01
│   │   ├─ validity_date = 2026-09-15
│   │   ├─ status = 'DRAFT'
│   │   ├─ total_amount = 53100
│   │   └─ created_at = 2026-09-01 09:30 IST
│   │
│   └─ quotation_line_items table:
│       ├─ line_id = 1
│       ├─ quotation_id = QT-2026-0001 (FK)
│       ├─ product_id = 5001 (FK)
│       ├─ quantity = 100
│       ├─ unit_price = 450
│       ├─ discount_pct = 0
│       ├─ tax_rate = 18
│       ├─ line_amount = 45000
│       └─ tax_amount = 8100
│
└─ State: status = 'DRAFT' (quotation exists, not committed)

STEP 2: Sales Rep sends Quotation to Customer
├─ System generates PDF
├─ Sends email to customer@customerdomain.com
└─ Status remains 'DRAFT' (awaiting response)

STEP 3: Customer reviews & accepts (or rejects)
├─ If accepted: Sales Rep marks quotation as "SUBMITTED"
│   └─ quotations.status = 'SUBMITTED'
│
└─ If rejected/expired: Status = 'REJECTED' or 'EXPIRED'
```

**Quotation State Diagram:**
```
┌────────┐
│ DRAFT  │ ← Created, edited by sales rep
└───┬────┘
    │ (Send to Customer)
    ↓
┌──────────────┐
│ SUBMITTED    │ ← Waiting for customer response
└───┬──────────┘
    │
    ├─→ ACCEPTED ──→ (Convert to SO)
    │
    ├─→ REJECTED ──→ [TERMINAL]
    │
    └─→ EXPIRED (if > validity_date) ──→ [TERMINAL]
```

---

#### **Phase 2: Sales Order Creation (T+4 days)**

```
ACTORS: Sales Rep, Finance (Credit check)
EVENT: Customer accepts quotation QT-2026-0001

STEP 1: Convert Quotation to Sales Order
├─ Input:
│   ├─ Quotation ID: QT-2026-0001
│   └─ Conversion triggered by sales rep
│
├─ Pre-checks (Atomic Block 1):
│   ├─ Quotation status = 'ACCEPTED' ? ✓
│   ├─ Quotation validity_date >= today ? ✓
│   ├─ Customer active status = 'ACTIVE' ? ✓
│   └─ No duplicate SO from this quote ? ✓
│
├─ Credit Validation (Atomic Block 2):
│   ├─ Fetch customer credit limit: ₹1,00,000
│   ├─ Fetch unpaid AR: ₹30,000 (sum of unpaid invoices)
│   ├─ Available credit: ₹1,00,000 - ₹30,000 = ₹70,000
│   ├─ SO amount: ₹53,100
│   ├─ Validation: ₹53,100 ≤ ₹70,000 ? ✓ PASS
│   └─ If fails: Reject SO, send alert to finance
│
├─ Inventory Availability Check (Atomic Block 3):
│   ├─ Check stock_summary for product_id = 5001
│   ├─ Available quantity: 150 units
│   ├─ SO quantity: 100 units
│   ├─ Validation: 100 ≤ 150 ? ✓ PASS
│   ├─ If partial: Create back-order, confirm available qty only
│   └─ If out of stock: Reject SO, alert procurement
│
├─ Database Inserts (Atomic Block 4 — Transaction):
│   ├─ sales_orders table:
│   │   ├─ so_id = SO-2026-0001 (auto-generated)
│   │   ├─ company_id = 1
│   │   ├─ customer_id = 101 (FK)
│   │   ├─ quotation_id = QT-2026-0001 (FK)
│   │   ├─ order_date = 2026-09-05
│   │   ├─ delivery_date = 2026-09-10 (default: today + 5 days)
│   │   ├─ status = 'CONFIRMED'
│   │   ├─ total_amount = 45000
│   │   ├─ total_tax = 8100
│   │   ├─ grand_total = 53100
│   │   └─ created_at = 2026-09-05 10:15 IST
│   │
│   ├─ sales_order_line_items table:
│   │   ├─ (Copy from quotation_line_items)
│   │   ├─ so_id = SO-2026-0001 (FK)
│   │   ├─ qty_ordered = 100
│   │   ├─ qty_reserved = 0 (initially)
│   │   ├─ qty_fulfilled = 0
│   │   └─ qty_invoiced = 0
│   │
│   └─ quotations table (update):
│       └─ status = 'CONVERTED_TO_SO'
│
├─ Inventory Reservation (Atomic Block 5 — Transaction):
│   ├─ Query stock_summary WHERE product_id = 5001
│   ├─ Create reservation record:
│   │   ├─ reservation_id = auto-generated
│   │   ├─ so_id = SO-2026-0001 (FK)
│   │   ├─ product_id = 5001 (FK)
│   │   ├─ qty_reserved = 100
│   │   ├─ warehouse_id = 1 (default warehouse)
│   │   └─ status = 'RESERVED'
│   │
│   ├─ Update stock_summary:
│   │   ├─ available_qty: 150 → 50 (reserved qty deducted)
│   │   ├─ reserved_qty: 0 → 100
│   │   └─ timestamp = now
│   │
│   └─ inventory_ledger entry (informational):
│       ├─ movement_type = 'RESERVATION'
│       ├─ product_id = 5001
│       ├─ qty_reserved = 100
│       ├─ reference = SO-2026-0001
│       └─ timestamp = now
│
├─ AR Subledger Update:
│   ├─ customers table (update expected AR):
│   │   ├─ expected_ar = 53100
│   │   └─ last_order_date = 2026-09-05
│   │
│   └─ No GL posting yet (SO is not invoiced)
│
└─ State: status = 'CONFIRMED' (ready for fulfillment)

STEP 2: Notify Warehouse
├─ Trigger event: SO_CONFIRMED
├─ Send to warehouse via message queue:
│   ├─ SO ID: SO-2026-0001
│   ├─ Delivery address: ABC Trading, XYZ Street, Bangalore
│   ├─ Items: Red Paint 1L, 100 units
│   ├─ Priority: Normal
│   └─ Required by: 2026-09-10
│
└─ Warehouse status: Awaiting picking
```

**Sales Order State Diagram:**
```
┌───────────┐
│ DRAFT     │ ← Created from SO form
└─────┬─────┘
      │ (Confirm after credit/stock check)
      ↓
┌───────────────────┐
│ CONFIRMED         │ ← Reserved inventory, awaiting fulfillment
└─────┬─────────────┘
      │ (Warehouse picks & packs)
      ↓
┌───────────────────┐
│ PICKED            │ ← Items removed from shelf
└─────┬─────────────┘
      │ (QC & packing done)
      ↓
┌───────────────────┐
│ PACKED            │ ← Ready for dispatch
└─────┬─────────────┘
      │ (Goods handed to courier/driver)
      ↓
┌───────────────────┐
│ DISPATCHED        │ ← In transit to customer
└─────┬─────────────┘
      │ (Customer receives)
      ↓
┌───────────────────┐
│ DELIVERED         │ ← Can now invoice
└─────┬─────────────┘
      │ (Invoice generated & posted)
      ↓
┌───────────────────┐
│ INVOICED          │ ← Tax invoice issued, GL posted
└─────┬─────────────┘
      │ (Payment received)
      ↓
┌───────────────────┐
│ PAID              │ ← Order complete, AR cleared
└───────────────────┘

[Alternative: CANCELLED, REJECTED, BACK-ORDER]
```

---

#### **Phase 3: Fulfillment (Pick, Pack, Dispatch) — T+8 days**

```
ACTORS: Warehouse Manager, Picking staff, QC, Driver
EVENT: SO-2026-0001 confirmed, reserved inventory available

STEP 1: Generate Pick List
├─ Input: SO-2026-0001
├─ System Query:
│   ├─ Fetch SO line items
│   ├─ Fetch product locations (warehouse bins)
│   ├─ Generate pick sequence (FIFO by bin location)
│   └─ Print pick list
│
├─ Database Insert:
│   ├─ pick_lists table:
│   │   ├─ pick_list_id = PL-2026-0001
│   │   ├─ so_id = SO-2026-0001 (FK)
│   │   ├─ warehouse_id = 1
│   │   ├─ status = 'INITIATED'
│   │   └─ created_at = 2026-09-08 08:00 IST
│   │
│   └─ pick_list_items table:
│       ├─ line_id = auto
│       ├─ pick_list_id = PL-2026-0001
│       ├─ product_id = 5001
│       ├─ qty_to_pick = 100
│       ├─ bin_location = 'A1-02' (from warehouse layout)
│       └─ status = 'PENDING'
│
└─ State: SO status remains 'CONFIRMED', PL status = 'INITIATED'

STEP 2: Picking (Physical Movement)
├─ Warehouse staff picks items from bin A1-02
├─ Count verified: 100 units Red Paint
├─ Update pick_list_items:
│   ├─ qty_picked = 100
│   ├─ status = 'PICKED'
│   └─ picked_at = 2026-09-08 09:30 IST
│
└─ Inventory still reserved (not yet deducted from GL)

STEP 3: Quality Check
├─ QC inspects 100 units
├─ Results:
│   ├─ Acceptable: 100 units
│   ├─ Damaged: 0 units
│   └─ Qty accepted: 100 units
│
├─ Update pick_list_items:
│   ├─ qty_accepted = 100
│   ├─ status = 'QC_PASSED'
│   └─ qc_passed_at = 2026-09-08 10:00 IST
│
└─ If failed: Flag, hold for rework, update SO status = 'QUALITY_HOLD'

STEP 4: Packing
├─ Pack 100 units into 10 boxes (10 units per box)
├─ Apply labels, stickers, pack date
├─ Create packing slip (shows contents, weight, dimensions)
│
├─ Database Insert:
│   └─ packing_slips table:
│       ├─ packing_slip_id = PS-2026-0001
│       ├─ pick_list_id = PL-2026-0001
│       ├─ so_id = SO-2026-0001
│       ├─ qty_packed = 100
│       ├─ weight = 250 kg (100 units × 2.5 kg)
│       ├─ boxes = 10
│       └─ packed_at = 2026-09-08 11:00 IST
│
├─ Update SO status: 'PACKED'
└─ Ready for dispatch

STEP 5: Dispatch
├─ Handover to driver/courier
├─ Driver details: Raj Kumar, Vehicle: TATA XUV, Plate: KA-02-AB-0001
├─ Customer address verified
├─ Goods handed over, signed
│
├─ Database Inserts:
│   └─ delivery_challans table:
│       ├─ challan_id = DC-2026-0001
│       ├─ so_id = SO-2026-0001 (FK)
│       ├─ challan_date = 2026-09-08
│       ├─ dispatch_date = 2026-09-08 14:00
│       ├─ driver_id = (Raj Kumar)
│       ├─ vehicle_number = 'KA-02-AB-0001'
│       ├─ origin_location = 'Main Warehouse, Bangalore'
│       ├─ destination_address = 'ABC Trading, XYZ Street, Bangalore'
│       ├─ qty_dispatched = 100
│       ├─ status = 'DISPATCHED'
│       └─ expected_delivery = 2026-09-09 18:00 IST
│
├─ Update SO status: 'DISPATCHED'
├─ Create e-Way Bill (if inter-state, value > ₹50,000):
│   └─ e_way_bills table:
│       ├─ ewb_id = auto
│       ├─ so_id = SO-2026-0001
│       ├─ challan_id = DC-2026-0001
│       ├─ generation_date = 2026-09-08
│       ├─ validity_date = 2026-10-08 (30 days)
│       ├─ ewb_number = (generated from NSDL API)
│       └─ status = 'GENERATED'
│
└─ Event: SO_DISPATCHED → Alert customer (SMS/Email)
```

---

#### **Phase 4: Invoicing (T+11 days)**

```
ACTORS: Finance/Billing team
EVENT: Goods delivered & invoicing triggered

STEP 1: Pre-Invoice Validation
├─ Query SO-2026-0001:
│   ├─ Status = 'DELIVERED' ? ✓
│   ├─ Qty delivered = 100 ? ✓
│   ├─ No prior invoice ? ✓
│   └─ Customer active ? ✓
│
├─ Fetch GRN equivalent (delivery confirmation):
│   ├─ challan_id = DC-2026-0001
│   ├─ qty_dispatched = 100 ✓
│   └─ Status = 'DELIVERED' ✓
│
└─ All checks pass → Proceed to invoice

STEP 2: Generate Tax Invoice
├─ Input:
│   ├─ SO ID: SO-2026-0001
│   ├─ Delivery Challan: DC-2026-0001
│   └─ Invoice date: 2026-09-11
│
├─ System Processing:
│   ├─ Fetch SO line items (Red Paint 100 × ₹450)
│   ├─ Lookup HSN code: 3208 (Paint)
│   ├─ Determine tax jurisdiction:
│   │   ├─ Seller state (company): Karnataka
│   │   ├─ Buyer state (customer): Karnataka
│   │   └─ Same state → Intra-state GST
│   │
│   ├─ Calculate tax (Intra-state):
│   │   ├─ Taxable amount: ₹45,000
│   │   ├─ GST rate: 18%
│   │   ├─ SGST (State 9%): ₹4,050
│   │   ├─ CGST (Central 9%): ₹4,050
│   │   ├─ Total tax: ₹8,100
│   │   └─ Invoice total: ₹53,100
│   │
│   └─ Verify rounding (GST compliance):
│       └─ Total = Amount + SGST + CGST = ₹45,000 + ₹4,050 + ₹4,050 = ₹53,100 ✓
│
├─ Database Inserts (Atomic Block):
│   ├─ tax_invoices table:
│   │   ├─ invoice_id = INV-2026-0001 (auto-generated)
│   │   ├─ invoice_number = INV-2026-0001 (external reference)
│   │   ├─ company_id = 1
│   │   ├─ customer_id = 101 (FK)
│   │   ├─ so_id = SO-2026-0001 (FK)
│   │   ├─ challan_id = DC-2026-0001 (FK)
│   │   ├─ invoice_date = 2026-09-11
│   │   ├─ due_date = 2026-10-11 (invoice_date + 30 days)
│   │   ├─ status = 'DRAFTED'
│   │   ├─ subtotal = 45000
│   │   ├─ sgst_amount = 4050
│   │   ├─ cgst_amount = 4050
│   │   ├─ total_tax = 8100
│   │   ├─ grand_total = 53100
│   │   ├─ irn = NULL (e-invoice not generated yet)
│   │   ├─ e_invoice_status = 'NOT_GENERATED'
│   │   └─ created_at = 2026-09-11 10:00 IST
│   │
│   └─ tax_invoice_line_items table:
│       ├─ line_id = auto
│       ├─ invoice_id = INV-2026-0001 (FK)
│       ├─ product_id = 5001 (FK)
│       ├─ hsn_code = '3208'
│       ├─ quantity = 100
│       ├─ unit_price = 450
│       ├─ line_amount = 45000
│       ├─ tax_rate = 18
│       └─ tax_amount = 8100
│
├─ Update SO:
│   ├─ status = 'INVOICED'
│   └─ invoiced_date = 2026-09-11
│
└─ Invoice Status: 'DRAFTED' (ready for approval)

STEP 3: E-Invoice Generation (GST Compliance)
├─ Trigger: Auto-generate when invoice approved
├─ System calls NSDL API:
│   ├─ Seller GSTIN: (Company GSTIN)
│   ├─ Buyer GSTIN: (Customer GSTIN or PAN if unregistered)
│   ├─ Invoice number, date, amount
│   ├─ Line items with HSN, qty, tax
│   └─ Document data in JSON format
│
├─ NSDL Response:
│   ├─ IRN (Invoice Reference Number): 12345-IRP-2026 (unique identifier)
│   ├─ Acknowledgement number: (IRP acknowledgement)
│   ├─ QR code data: (embedded in PDF)
│   ├─ Timestamp: (NSDL timestamp)
│   └─ Status: 'VALID' (invoice legitimized)
│
├─ Database Update (Atomic Block):
│   └─ tax_invoices table:
│       ├─ irn = '12345-IRP-2026'
│       ├─ ack_number = (from IRP)
│       ├─ e_invoice_status = 'VALID'
│       ├─ qr_code_path = '/invoices/QR/INV-2026-0001.png'
│       └─ e_invoiced_at = 2026-09-11 10:15 IST
│
├─ Generate PDF with QR code:
│   └─ PDF includes:
│       ├─ Invoice details
│       ├─ QR code (scannable)
│       ├─ IRN
│       ├─ Acknowledgement number
│       └─ "E-Invoice Generated - Valid" watermark
│
└─ Invoice Status: 'ISSUED' (e-invoiced, ready to send to customer)

STEP 4: GL Posting (CRITICAL — Real-time Posting)
├─ Trigger: On invoice approval/e-invoice generation
├─ GL Entry Details (Journal):
│   ├─ Entry Date: 2026-09-11
│   ├─ Reference: INV-2026-0001
│   ├─ Debit Side:
│   │   ├─ Account 1010 (Trade Receivable):
│   │   │   └─ Debit: ₹53,100 (customer now owes full amount including tax)
│   │   │
│   ├─ Credit Side:
│   │   ├─ Account 4010 (Sales Revenue @ 18% GST):
│   │   │   └─ Credit: ₹45,000 (actual revenue)
│   │   │
│   │   ├─ Account 2020 (SGST Payable):
│   │   │   └─ Credit: ₹4,050 (tax collected, liability to govt)
│   │   │
│   │   └─ Account 2030 (CGST Payable):
│   │       └─ Credit: ₹4,050 (tax collected, liability to govt)
│   │
│   └─ Balancing Check:
│       └─ Debits (₹53,100) = Credits (₹45,000 + ₹4,050 + ₹4,050) ✓
│
├─ Database Inserts (Atomic Transaction):
│   ├─ gl_entries table:
│   │   ├─ entry_id = auto
│   │   ├─ entry_date = 2026-09-11
│   │   ├─ posted_by_user_id = (Finance user)
│   │   ├─ reference_document = 'INV-2026-0001'
│   │   ├─ description = 'Sales invoice to ABC Trading'
│   │   ├─ status = 'POSTED'
│   │   ├─ total_debits = 53100
│   │   ├─ total_credits = 53100
│   │   └─ posted_at = 2026-09-11 10:30 IST
│   │
│   └─ gl_entry_details table:
│       ├─ Line 1:
│       │   ├─ entry_id = (FK)
│       │   ├─ account_id = 1010 (Trade Receivable)
│       │   ├─ debit = 53100
│       │   ├─ credit = 0
│       │   └─ cost_center = NULL
│       │
│       ├─ Line 2:
│       │   ├─ account_id = 4010 (Sales Revenue)
│       │   ├─ debit = 0
│       │   ├─ credit = 45000
│       │   └─ cost_center = 'SALES'
│       │
│       ├─ Line 3:
│       │   ├─ account_id = 2020 (SGST Payable)
│       │   ├─ debit = 0
│       │   ├─ credit = 4050
│       │   └─ cost_center = 'TAX'
│       │
│       └─ Line 4:
│           ├─ account_id = 2030 (CGST Payable)
│           ├─ debit = 0
│           ├─ credit = 4050
│           └─ cost_center = 'TAX'
│
├─ COGS Posting (Inventory → GL):
│   ├─ Fetch inventory cost (FIFO method):
│   │   ├─ Red Paint purchased @ ₹300 per unit (oldest batch)
│   │   ├─ Qty sold: 100 units
│   │   └─ COGS: 100 × ₹300 = ₹30,000
│   │
│   ├─ GL Entry:
│   │   ├─ Account 5010 (Cost of Goods Sold):
│   │   │   └─ Debit: ₹30,000 (expense)
│   │   │
│   │   └─ Account 1040 (Inventory):
│   │       └─ Credit: ₹30,000 (asset reduced)
│   │
│   └─ GL Posting Details (new entry, separate from above)
│       └─ Debits (₹30,000) = Credits (₹30,000) ✓
│
├─ AR Subledger Update:
│   ├─ customers table:
│   │   ├─ customer_id = 101
│   │   ├─ current_ar_balance = 30000 (prior) + 53100 (this invoice) = 83100
│   │   ├─ ar_aging_bucket = 'Current' (0-30 days from invoice date)
│   │   └─ last_invoice_date = 2026-09-11
│   │
│   └─ customer_invoices table (subledger):
│       ├─ customer_id = 101 (FK)
│       ├─ invoice_id = INV-2026-0001 (FK)
│       ├─ invoice_amount = 53100
│       ├─ due_date = 2026-10-11
│       ├─ status = 'UNPAID'
│       ├─ aging_days = 0 (just issued)
│       └─ last_reminder_date = NULL
│
├─ Inventory Update (CRITICAL):
│   ├─ Reduce reserved qty to fulfilled:
│   │   ├─ stock_summary (product_id = 5001):
│   │   │   ├─ available_qty: 50 (before deduction) → 50 (no change, already reserved)
│   │   │   ├─ reserved_qty: 100 → 0 (reservation released)
│   │   │   ├─ valuation_method: FIFO
│   │   │   ├─ cost_per_unit: ₹300 (FIFO cost)
│   │   │   ├─ inventory_value: (50+100) × ₹300 = ₹45,000 (before) → 50 × ₹300 = ₹15,000 (after)
│   │   │   └─ updated_at = 2026-09-11 10:30 IST
│   │
│   ├─ inventory_ledger entry (COGS deduction):
│   │   ├─ movement_type = 'COGS_DEDUCTION'
│   │   ├─ product_id = 5001 (FK)
│   │   ├─ qty_out = 100 (units sold)
│   │   ├─ unit_cost = 300 (FIFO)
│   │   ├─ total_cost = 30000
│   │   ├─ reference = 'INV-2026-0001'
│   │   ├─ running_balance_qty = 150 (before) → 50 (after)
│   │   └─ timestamp = 2026-09-11 10:30 IST
│   │
│   └─ Reconciliation Check:
│       └─ GL Inventory account (₹15,000) = stock_summary valuation (50 units × ₹300) ✓
│
└─ Invoice Status: 'POSTED' (GL entries recorded, immutable)
```

---

#### **Phase 5: Payment Collection (T+40 days)**

```
ACTORS: Accountant, Finance, Bank
EVENT: Invoice due date approaches (Oct 11)

STEP 1: Payment Receipt
├─ Customer pays ₹53,100 via NEFT (bank transfer)
├─ Transaction:
│   ├─ Date: 2026-10-20 (9 days late)
│   ├─ Amount: ₹53,100
│   ├─ Mode: NEFT (electronic transfer)
│   ├─ Reference: INV-2026-0001 / SO-2026-0001
│   └─ Status: Credited to company bank account
│
├─ Bank Statement Entry:
│   ├─ Bank: ICICI, Account: 1234567890
│   ├─ Date: 2026-10-20
│   ├─ Credit: ₹53,100
│   ├─ Description: "NEFT INV-2026-0001"
│   └─ Running balance: ₹5,53,100
│
├─ Database Insert:
│   ├─ payment_receipts table:
│   │   ├─ receipt_id = auto
│   │   ├─ invoice_id = INV-2026-0001 (FK)
│   │   ├─ receipt_date = 2026-10-20
│   │   ├─ amount = 53100
│   │   ├─ mode = 'NEFT'
│   │   ├─ bank_ref = 'UTR-12345' (NEFT UTR number)
│   │   ├─ status = 'RECEIVED'
│   │   └─ created_at = 2026-10-20 09:45 IST
│   │
│   └─ bank_receipts table (bank reconciliation):
│       ├─ bank_id = 1 (ICICI)
│       ├─ receipt_date = 2026-10-20
│       ├─ reference = 'NEFT INV-2026-0001'
│       ├─ amount = 53100
│       └─ matched_with_payment = NULL (pending match)
│
└─ Status: Payment physically received

STEP 2: GL Posting (Bank + AR Clearing)
├─ GL Entry:
│   ├─ Debit Side:
│   │   └─ Account 1020 (Cash at Bank):
│   │       └─ Debit: ₹53,100 (cash received)
│   │
│   └─ Credit Side:
│       └─ Account 1010 (Trade Receivable):
│           └─ Credit: ₹53,100 (customer debt cleared)
│
├─ Database Insert:
│   └─ gl_entries table:
│       ├─ entry_id = auto
│       ├─ entry_date = 2026-10-20
│       ├─ reference_document = 'NEFT INV-2026-0001'
│       ├─ total_debits = 53100
│       ├─ total_credits = 53100
│       └─ status = 'POSTED'
│
├─ AR Subledger Update:
│   ├─ customer_invoices table:
│   │   ├─ invoice_id = INV-2026-0001
│   │   ├─ status = 'PAID'
│   │   ├─ paid_date = 2026-10-20
│   │   ├─ payment_amount = 53100
│   │   └─ dso = 39 (Days Sales Outstanding: Sept 11 → Oct 20)
│   │
│   └─ customers table:
│       ├─ customer_id = 101
│       ├─ current_ar_balance = 83100 (prior) - 53100 (this payment) = 30000 (other invoices)
│       └─ last_payment_date = 2026-10-20
│
├─ Bank Reconciliation (Setup for monthly close):
│   └─ bank_reconciliations table:
│       ├─ reconciliation_id = auto
│       ├─ bank_id = 1
│       ├─ reconciliation_date = 2026-10-20
│       ├─ gl_bank_balance = ₹5,53,100
│       ├─ bank_statement_balance = ₹5,53,100
│       ├─ status = 'RECONCILED'
│       └─ timestamp = 2026-10-20 14:00 IST
│
└─ Invoice Status: 'PAID' (order complete)

STEP 3: AR Aging Update (for reporting)
├─ Daily batch job recalculates aging:
│   ├─ Current (0-30 days): ₹X (invoices within 30 days of invoice date)
│   ├─ 30-60 days: ₹Y
│   ├─ 60-90 days: ₹Z
│   └─ 90+ days: ₹W (overdue)
│
└─ AR aging report generated for finance dashboard
```

**Invoice State Diagram:**
```
┌─────────────────┐
│ DRAFT           │ ← Created from SO
└────────┬────────┘
         │ (Approve & e-invoice)
         ↓
┌─────────────────┐
│ ISSUED          │ ← E-invoiced (IRN assigned)
└────────┬────────┘
         │ (Posted to GL)
         ↓
┌─────────────────┐
│ POSTED          │ ← GL entries immutable
└────────┬────────┘
         │ (Payment received)
         ↓
┌─────────────────┐
│ PAID            │ ← AR cleared, order closed
└─────────────────┘

[Alternative paths: CANCELLED, REJECTED, OVERDUE]
```

---

## Workflow 2: Procure-to-Pay (P2P)

### 2.1 Step-by-Step Workflow (Condensed)

```
STEP 1: Reorder Alert Triggered
├─ Inventory check: Red Paint stock = 150 units
├─ Reorder point: 200 units
├─ Status: BELOW REORDER POINT → Alert
└─ → Create Purchase Requisition (PR-2026-0001)

STEP 2: Create Purchase Requisition (PR)
├─ Item: Red Paint 1L
├─ Qty needed: 500 units (1-month supply)
├─ Suggested vendor: ABC Paint Co. (from supplier master)
├─ Est. cost: ₹225,000
├─ Approver: Procurement Manager
└─ PR Status: APPROVED

STEP 3: Create Purchase Order (PO)
├─ Convert PR → PO-2026-0001
├─ Vendor: ABC Paint Co. (Vendor ID 201)
├─ Items: Red Paint 1L, 500 units @ ₹450
├─ PO Amount: ₹225,000 (before tax)
├─ Payment Terms: Net 30
├─ TDS: Applicable (1% = ₹2,250 on goods)
├─ PO sent to vendor via email
├─ Status: SENT
└─ Database: purchase_orders table (+ line items)

STEP 4: Goods Receipt (GRN)
├─ Goods arrive (Sept 6, 3-day lead time)
├─ Physical count: 500 units received ✓
├─ Quality check: All OK
├─ Create GRN-2026-0001 (linked to PO)
├─ GL Posting:
│   ├─ DR Inventory: ₹225,000 (stock added)
│   └─ CR Accounts Payable: ₹225,000
├─ Inventory updated: Available qty 150 + 500 = 650 units
└─ Status: RECEIVED

STEP 5: Vendor Invoice Matching
├─ Invoice received (Sept 8): VP-2026-001
├─ Amount: ₹265,500 (incl. 18% GST)
├─ 3-Way Match:
│   ├─ PO line: 500 units @ ₹450 = ₹225,000 ✓
│   ├─ GRN: 500 units received ✓
│   └─ Invoice: 500 units @ ₹450 = ₹225,000 ✓
├─ Status: MATCHED & APPROVED
└─ Database: bills_payable table

STEP 6: GL Posting (Vendor Bill)
├─ GL Entry:
│   ├─ DR Expense (Purchase): ₹225,000
│   ├─ DR Input GST (tax claimable): ₹40,500
│   └─ CR Accounts Payable: ₹265,500
├─ AP Subledger: Vendor balance increased to ₹265,500
└─ Status: POSTED

STEP 7: Payment (Oct 7, due date)
├─ Payment approved: ₹265,500
├─ TDS deduction: 1% on goods = ₹2,250
├─ Net payment: ₹263,250 (₹265,500 - ₹2,250)
├─ Transfer via NEFT
├─ GL Posting:
│   ├─ DR AP: ₹265,500 (liability cleared)
│   ├─ CR Bank: ₹263,250
│   └─ CR TDS Payable: ₹2,250
├─ Vendor AR cleared: Status = PAID
└─ TDS tracked for Form 26Q filing
```

**PO & Bill State Diagrams:**
```
Purchase Requisition:
DRAFT → SUBMITTED → APPROVED → CONVERTED_TO_PO → [TERMINAL]

Purchase Order:
DRAFT → SENT → ACKNOWLEDGED → PARTIAL_RECEIPT / FULLY_RECEIVED → CLOSED / CANCELLED

Vendor Bill:
RECEIVED → MATCHED → APPROVED → PAID → [TERMINAL]
```

---

## Workflow 3: Inventory & Warehouse Transfers

### 3.1 Inter-Warehouse Stock Movement

```
SCENARIO: Transfer 200 units Red Paint from HQ to Branch

STEP 1: Create Transfer Request
├─ From: HQ Warehouse (WH-01)
├─ To: Branch Warehouse (WH-02)
├─ Product: Red Paint 1L
├─ Qty: 200 units
├─ Reason: Branch sales increasing, low stock
└─ Status: SUBMITTED

STEP 2: Approval
├─ HQ Warehouse Manager reviews
├─ Stock check: HQ has 650 units available ✓
├─ Approval: GRANTED
└─ Status: APPROVED

STEP 3: Dispatch from HQ
├─ Pick 200 units from HQ
├─ Update stock_summary (HQ):
│   ├─ available_qty: 650 → 450 (transferred out)
│   └─ timestamp = now
│
├─ Inventory Ledger (HQ):
│   ├─ movement_type = 'TRANSFER_OUT'
│   ├─ qty_out = 200
│   └─ reference = 'TR-2026-0001'
│
└─ No GL posting (transfer is internal, doesn't affect profit)

STEP 4: In-Transit (Holding)
├─ Goods en route from HQ to Branch
├─ Stock movement logged but not yet in Branch inventory
└─ Duration: Depends on location distance

STEP 5: Receipt at Branch
├─ Branch receives 200 units
├─ Update stock_summary (Branch):
│   ├─ available_qty: 50 → 250 (transferred in)
│   └─ timestamp = now
│
├─ Inventory Ledger (Branch):
│   ├─ movement_type = 'TRANSFER_IN'
│   ├─ qty_in = 200
│   └─ reference = 'TR-2026-0001'
│
└─ Status: COMPLETED

FINAL POSITION:
├─ HQ: 450 units (was 650)
├─ Branch: 250 units (was 50)
├─ Total: 700 units (unchanged, redistributed)
└─ GL Inventory: ₹210,000 (unchanged, just reclassified by location)
```

---

## Workflow 4: Point of Sale (POS) Retail Checkout

### 4.1 Counter Billing & Inventory Sync

```
SCENARIO: Customer buys Red Paint at retail counter

STEP 1: Scan & Add Items
├─ Cashier scans barcode: 5001-RP-1L
├─ System looks up:
│   ├─ Product: Red Paint 1L
│   ├─ Price: ₹500 (retail, including tax)
│   ├─ Tax: 18% (₹76.27 included in ₹500)
│   └─ Stock: 250 units available
│
├─ Add to cart:
│   ├─ Qty: 1 unit
│   ├─ Price: ₹500
│   └─ Running cart total: ₹500
│
├─ Repeat for more items (assume same item again)
│   ├─ Qty: 2 units
│   ├─ Price: ₹1,000
│   └─ Running cart total: ₹1,500
│
└─ Database: pos_cart (session table, temp storage)

STEP 2: Bill Preview
├─ Total items: 2 units
├─ Subtotal (excl. tax): ₹847.46
├─ Tax (18%): ₹152.54
├─ Total amount: ₹1,500.00
└─ System rounds to nearest rupee

STEP 3: Payment
├─ Mode: Cash (₹1,500 notes)
├─ Tendered: ₹1,500
├─ Change: ₹0
└─ Payment Status: RECEIVED

STEP 4: POS Transaction Processing (Atomic Block)
├─ Create POS Transaction:
│   ├─ pos_transactions table:
│   │   ├─ transaction_id = auto
│   │   ├─ terminal_id = POS-01 (counter 1)
│   │   ├─ transaction_date = 2026-09-11 15:30
│   │   ├─ cashier_id = (cashier username)
│   │   ├─ items_count = 1 (Red Paint 1L: qty 2 units)
│   │   ├─ subtotal = 847.46
│   │   ├─ tax_amount = 152.54
│   │   ├─ total = 1500.00
│   │   ├─ payment_mode = 'CASH'
│   │   ├─ payment_amount = 1500.00
│   │   ├─ change = 0.00
│   │   ├─ status = 'COMPLETED'
│   │   └─ created_at = 2026-09-11 15:30:45 IST
│   │
│   └─ pos_transaction_items table:
│       ├─ transaction_id = (FK)
│       ├─ product_id = 5001 (Red Paint)
│       ├─ qty = 2
│       ├─ unit_price = ₹423.73 (excl tax)
│       ├─ line_total = ₹847.46
│       └─ tax = ₹152.54
│
├─ Inventory Deduction (REAL-TIME):
│   ├─ Query stock_summary (product_id = 5001, warehouse_id = retail_wh):
│   │   ├─ available_qty: 250 → 248 (2 units sold)
│   │   └─ timestamp = now
│   │
│   ├─ inventory_ledger entry:
│   │   ├─ movement_type = 'POS_SALE'
│   │   ├─ product_id = 5001
│   │   ├─ qty_out = 2
│   │   ├─ unit_cost = ₹300 (FIFO cost)
│   │   ├─ total_cost = ₹600
│   │   ├─ reference = transaction_id
│   │   └─ timestamp = now
│   │
│   └─ Reconciliation:
│       └─ GL Inventory = ₹74,400 (248 units × ₹300) ✓
│
├─ GL Posting (IMMEDIATE):
│   ├─ GL Entry:
│   │   ├─ Debit Side:
│   │   │   ├─ Account 1020 (Cash at Bank):
│   │   │   │   └─ Debit: ₹1,500 (cash received)
│   │   │   │
│   │   ├─ Credit Side:
│   │   │   ├─ Account 4010 (Retail Sales Revenue):
│   │   │   │   └─ Credit: ₹847.46 (sales)
│   │   │   │
│   │   │   ├─ Account 2020 (SGST Payable):
│   │   │   │   └─ Credit: ₹76.27 (9% of taxable)
│   │   │   │
│   │   │   ├─ Account 2030 (CGST Payable):
│   │   │   │   └─ Credit: ₹76.27 (9% of taxable)
│   │   │   │
│   │   └─ Balancing: Debits (₹1,500) = Credits (₹847.46 + ₹76.27 + ₹76.27) ✓
│   │
│   ├─ COGS Posting:
│   │   ├─ Debit Side:
│   │   │   ├─ Account 5010 (COGS):
│   │   │   │   └─ Debit: ₹600 (2 units @ ₹300 cost)
│   │   │   │
│   │   ├─ Credit Side:
│   │   │   ├─ Account 1040 (Inventory):
│   │   │   │   └─ Credit: ₹600 (stock reduced)
│   │   │   │
│   │   └─ Balancing: Debits (₹600) = Credits (₹600) ✓
│   │
│   └─ GL entries posted immediately (not batched)
│
└─ Transaction Status: COMPLETED

STEP 5: Receipt Generation & Delivery
├─ Thermal receipt printed:
│   ├─ Store name, address
│   ├─ Terminal ID, Date, Time
│   ├─ Items: Red Paint 1L × 2 @ ₹423.73 = ₹847.46
│   ├─ Tax: ₹152.54
│   ├─ Total: ₹1,500.00
│   ├─ Payment: Cash ₹1,500.00
│   ├─ Change: ₹0.00
│   ├─ Barcode: (transaction barcode for return tracking)
│   └─ Thank you message
│
└─ Handed to customer

STEP 6: End-of-Shift Reconciliation (5:00 PM)
├─ Cashier closes shift
├─ Cash drawer balance calculation:
│   ├─ Opening float: ₹5,000
│   ├─ Cash sales during shift: ₹15,200 (total of all POS transactions)
│   ├─ Expected cash in drawer: ₹5,000 + ₹15,200 = ₹20,200
│   │
│   ├─ Physical count: ₹20,200 ✓ (matches)
│   │
│   └─ Variance: ₹0 (perfect)
│
├─ System reconciliation:
│   ├─ Query all POS transactions for shift:
│   │   ├─ Sum of transaction totals = ₹15,200
│   │   ├─ Sum of GL cash debits = ₹15,200
│   │   ├─ GL Sales Revenue = ₹12,847.46 (all POS sales)
│   │   ├─ GL SGST/CGST = ₹2,352.54 (all POS taxes)
│   │   └─ Reconciliation: ✓ PASS
│   │
│   └─ Shift Status: CLOSED
│
└─ End-of-shift report generated (for manager review)
```

**POS Transaction State:**
```
┌───────────────────┐
│ IN_PROGRESS       │ ← Items being scanned
└─────────┬─────────┘
          │ (Customer pays)
          ↓
┌───────────────────┐
│ COMPLETED         │ ← Transaction done, receipt printed
└───────────────────┘

[Alternative: CANCELLED (if customer changes mind before payment)]
```

---

## Workflow 5: Record-to-Report (Financial Closing & Tax)

### 5.1 Real-Time GL Posting & Compliance

```
SCENARIO: Month-end close (September 30, 2026)

STEP 1: Pre-Close Validation
├─ Check GL entries posted for all operational events:
│   ├─ Sales invoices (all SO→INV→GL) ✓
│   ├─ Purchase bills (all PO→GRN→BILL→GL) ✓
│   ├─ POS transactions (all POS→GL) ✓
│   └─ Bank receipts/payments (all reconciled) ✓
│
├─ Verify subledger balances:
│   ├─ GL AR account (1010) = Sum of customer AR subledger ✓
│   ├─ GL AP account (2010) = Sum of vendor AP subledger ✓
│   ├─ GL Inventory (1040) = Stock valuation (qty × cost) ✓
│   └─ GL Cash (1020) = Bank statement balance ✓
│
└─ All reconciliations pass → Proceed to close

STEP 2: Accruals & Adjustments (Manual GL Entries)
├─ Accrued Expenses (e.g., Sept utilities not yet billed):
│   ├─ DR Utilities Expense: ₹5,000
│   └─ CR Accrued Utilities Payable: ₹5,000
│
├─ Bad Debt Provision:
│   ├─ Calculate: 2% of overdue AR (>90 days)
│   ├─ Overdue AR: ₹50,000
│   ├─ Provision: ₹1,000 (2%)
│   ├─ DR Bad Debt Expense: ₹1,000
│   └─ CR Allowance for Bad Debt: ₹1,000
│
├─ Depreciation (monthly):
│   ├─ PPE cost: ₹50,00,000
│   ├─ Useful life: 10 years
│   ├─ Annual depreciation: ₹5,00,000
│   ├─ Monthly: ₹41,667
│   ├─ DR Depreciation Expense: ₹41,667
│   └─ CR Accumulated Depreciation: ₹41,667
│
└─ All manual entries approved & posted

STEP 3: GST Compliance (GSTR-1 & GSTR-3B)

GSTR-1 Generation (Outward Supplies):
├─ Query all sales invoices for Sept:
│   ├─ Filter: invoice_date >= 2026-09-01 AND < 2026-10-01
│   ├─ Group by tax rate & customer type (B2B vs B2C)
│   │
│   └─ Sample data:
│       ├─ B2B @ 5% GST: ₹1,00,000 (tax: ₹5,000)
│       ├─ B2B @ 12% GST: ₹5,00,000 (tax: ₹60,000)
│       ├─ B2B @ 18% GST: ₹10,00,000 (tax: ₹1,80,000)
│       ├─ B2C (MRP-based, no GSTIN): ₹2,00,000
│       └─ Total Sales: ₹18,00,000
│
├─ GL Reconciliation (GSTR-1):
│   ├─ Sum of all sales revenue accounts (4010, 4020, etc.): ₹18,00,000 ✓
│   ├─ Sum of output tax accounts (2020, 2030, 2040): ₹2,45,000 ✓
│   └─ Match confirmed
│
├─ GSTR-1 Report Generated:
│   ├─ File on Govt portal (by Sept 20)
│   ├─ Includes QR codes for e-invoices
│   └─ Status: Filed & Acknowledged

GSTR-3B Generation (Self-Assessment):
├─ Calculate net tax payable:
│   ├─ Output Tax (GSTR-1): ₹2,45,000 (tax collected from customers)
│   │   ├─ SGST: ₹75,000
│   │   ├─ CGST: ₹75,000
│   │   └─ IGST: ₹95,000
│   │
│   ├─ Input Tax (from purchases):
│   │   ├─ Query all vendor bills for Sept
│   │   ├─ Sum of input tax accounts (1070, 1071, 1072): ₹1,50,000
│   │   ├─ Breakdown:
│   │   │   ├─ SGST received: ₹45,000
│   │   │   ├─ CGST received: ₹45,000
│   │   │   └─ IGST received: ₹60,000
│   │   │
│   │   └─ Note: Some purchases might be exempt or reverse-charge
│   │
│   ├─ Net GST Payable:
│   │   ├─ Output Tax: ₹2,45,000
│   │   ├─ Less: Input Tax: -₹1,50,000
│   │   └─ Net Payable: ₹95,000
│   │
│   └─ Govt Account Balance (prior months): ₹20,000 (balance from Aug)
│
├─ GSTR-3B Self-Assessment:
│   ├─ File on Govt portal (by Sept 20)
│   ├─ Pay ₹95,000 GST (due by Sept 20)
│   ├─ GL Posting (when paid):
│   │   ├─ DR GST Payable: ₹95,000
│   │   └─ CR Bank: ₹95,000
│   │
│   └─ Status: Paid & Acknowledged

TDS Form 26Q (Quarterly, if applicable):
├─ Compile all TDS deducted during Q2 (Jul-Sep):
│   ├─ TDS on goods purchases: ₹5,000 (1% of ₹5,00,000)
│   ├─ TDS on service purchases: ₹3,000 (2% of ₹1,50,000)
│   └─ Total TDS: ₹8,000
│
├─ GL Posting (when deposited):
│   ├─ DR TDS Payable: ₹8,000
│   └─ CR Bank: ₹8,000
│
└─ File Form 26Q (by Oct 15)

E-Invoice Compliance Check:
├─ Validate all invoices have IRN:
│   ├─ Query: tax_invoices WHERE e_invoice_status != 'VALID'
│   ├─ If any missing: Alert, regenerate
│   └─ Sept invoices: 100% e-invoiced ✓
│
└─ NSDL records match internal records ✓

STEP 4: Bank Reconciliation

Bank Reconciliation Process:
├─ Bank Statement (Sept 30, 2026):
│   ├─ Opening balance (Sept 1): ₹5,00,000
│   ├─ Credits (cash received): ₹15,00,000
│   ├─ Debits (payments): ₹10,00,000
│   ├─ Bank interest (quarterly): ₹5,000
│   ├─ Bank charges (monthly): ₹500
│   └─ Closing balance (Sept 30): ₹9,04,500
│
├─ GL Bank Account (1020):
│   ├─ Opening balance: ₹5,00,000
│   ├─ All receipts posted: +₹15,00,000
│   ├─ All payments posted: -₹10,00,000
│   └─ GL balance: ₹10,00,000 (before bank interest/charges)
│
├─ Reconciliation:
│   ├─ GL balance: ₹10,00,000
│   ├─ Add: Bank interest (not yet recorded): +₹5,000
│   ├─ Add: Bank charges (not yet recorded): -₹500
│   ├─ Adjusted GL: ₹10,04,500
│   │
│   ├─ Bank statement: ₹9,04,500
│   ├─ Outstanding cheques (issued but not cleared): ₹1,00,000
│   ├─ Adjusted bank: ₹10,04,500
│   │
│   └─ Reconciliation: Adjusted GL (₹10,04,500) = Adjusted Bank (₹10,04,500) ✓
│
├─ Manual GL Entries (for bank items):
│   ├─ DR Bank: ₹5,000 (interest income)
│   ├─ CR Interest Income: ₹5,000
│   │
│   ├─ DR Bank Charges: ₹500
│   └─ CR Bank: ₹500
│
└─ Bank Reconciliation Status: COMPLETE

STEP 5: Generate Financial Statements (Real-Time Reports)

P&L Statement (Sept 1-30, 2026):
├─ Revenue:
│   ├─ Sales Revenue: ₹18,00,000
│   └─ Interest Income: ₹5,000
│   └─ Total Revenue: ₹18,05,000
│
├─ Cost of Goods Sold:
│   ├─ Opening Inventory: ₹45,00,000
│   ├─ Add: Purchases: ₹8,00,000
│   ├─ Less: Closing Inventory: -₹44,00,000
│   └─ COGS: ₹9,00,000
│
├─ Gross Profit: ₹18,05,000 - ₹9,00,000 = ₹9,05,000
│
├─ Operating Expenses:
│   ├─ Salaries: ₹1,50,000
│   ├─ Utilities (accrued): ₹5,000
│   ├─ Depreciation: ₹41,667
│   ├─ Bad Debt Provision: ₹1,000
│   └─ Total: ₹1,97,667
│
├─ Operating Profit (EBIT): ₹9,05,000 - ₹1,97,667 = ₹7,07,333
│
├─ Interest & Other:
│   ├─ Interest Expense: ₹10,000
│   └─ Net: -₹10,000
│
├─ Profit Before Tax (PBT): ₹6,97,333
│
├─ Tax Expense:
│   ├─ Income Tax (30% assumed): ₹2,09,200
│   └─ GST (already in revenue, not income tax)
│
└─ Profit After Tax (PAT): ₹4,88,133

Balance Sheet (as of Sept 30, 2026):
├─ Assets:
│   ├─ Cash: ₹10,04,500
│   ├─ Trade Receivable: ₹8,00,000
│   ├─ Inventory: ₹44,00,000
│   ├─ PPE (net): ₹49,58,333 (₹50L - ₹41,667 depreciation)
│   └─ Total Assets: ₹1,11,62,833
│
├─ Liabilities:
│   ├─ Trade Payable: ₹5,00,000
│   ├─ GST Payable: ₹95,000
│   ├─ Accrued Utilities: ₹5,000
│   └─ Total Liabilities: ₹6,00,000
│
├─ Equity:
│   ├─ Capital: ₹1,00,00,000
│   ├─ Retained Earnings (Aug close): ₹5,74,700
│   ├─ Net Income (Sept): ₹4,88,133
│   └─ Total Equity: ₹1,05,62,833
│
└─ Check: Assets (₹1,11,62,833) = Liabilities (₹6,00,000) + Equity (₹1,05,62,833) ✓

Cash Flow Statement (Sept, Simplified):
├─ Operating Activities:
│   ├─ Profit: ₹4,88,133
│   ├─ Add: Depreciation (non-cash): ₹41,667
│   ├─ Changes in working capital:
│   │   ├─ AR increase: -₹1,50,000 (cash out, AR increase)
│   │   ├─ Inventory decrease: +₹1,00,000 (COGS, cash effect)
│   │   └─ AP increase: +₹3,00,000 (cash saved, deferred payment)
│   │
│   └─ Net Operating Cash: ₹6,79,800
│
├─ Investing Activities:
│   ├─ PPE purchase: -₹50,000
│   └─ Net Investing: -₹50,000
│
├─ Financing Activities:
│   ├─ Dividends: ₹0
│   └─ Net Financing: ₹0
│
├─ Net Cash Flow: ₹6,79,800 - ₹50,000 = ₹6,29,800
├─ Opening Cash (Sept 1): ₹3,75,000
└─ Closing Cash (Sept 30): ₹10,04,500 ✓

STEP 6: Lock Period (Month Closed)
├─ Sept GL locked → No more entries for Sept allowed
├─ Oct GL open → New entries for Oct accepted
├─ All GL reconciliations complete
└─ Financial statements finalized & ready for stakeholder review
```

**Month-End Close State:**
```
┌─────────────────────┐
│ OPEN (Sept 1-29)    │ ← Transactions posted daily
└──────────┬──────────┘
           │ (Sept 30 close process)
           ↓
┌─────────────────────┐
│ IN_CLOSE (Sept 30)  │ ← Accruals, adjustments, reconciliations
└──────────┬──────────┘
           │ (Validation & approval)
           ↓
┌─────────────────────┐
│ LOCKED (Sept)       │ ← No edit, GL immutable
└─────────────────────┘
           
Next month: Oct GL OPEN
```

---

# PART 2: MODULE INTERCONNECTIVITY & DATABASE BINDING

---

## 2.1 Transaction Atomicity & Multi-Module Consistency

### Example: Sales Order Creation (Single Atomic Transaction)

```sql
BEGIN TRANSACTION;
SAVEPOINT sp_so_creation;

-- Step 1: Validate & lock customer record
SELECT * FROM customers WHERE id = 101 FOR UPDATE;
-- Pre-check: Credit limit, status, payment terms

-- Step 2: Create SO
INSERT INTO sales_orders (
  so_id, company_id, customer_id, quotation_id, order_date, 
  delivery_date, status, total_amount, total_tax, grand_total
) VALUES (
  SO-2026-0001, 1, 101, QT-2026-0001, 2026-09-05,
  2026-09-10, 'CONFIRMED', 45000, 8100, 53100
);
-- ROW LOCKED: sales_orders table
-- FOREIGN KEY CHECK: customer_id = 101 exists ✓

-- Step 3: Create SO line items
INSERT INTO sales_order_line_items (
  so_id, product_id, quantity, unit_price, tax_rate, line_amount, tax_amount
) VALUES (
  SO-2026-0001, 5001, 100, 450, 18, 45000, 8100
);
-- ROW LOCKED: sales_order_line_items table
-- FOREIGN KEY CHECK: product_id = 5001 exists ✓, so_id = SO-2026-0001 exists ✓

-- Step 4: Lock & update inventory
SELECT * FROM stock_summary WHERE product_id = 5001 AND warehouse_id = 1 FOR UPDATE;
-- Before: available_qty = 150, reserved_qty = 0

UPDATE stock_summary SET
  available_qty = available_qty - 100,
  reserved_qty = reserved_qty + 100,
  updated_at = NOW()
WHERE product_id = 5001 AND warehouse_id = 1;
-- After: available_qty = 50, reserved_qty = 100
-- ROW LOCKED: stock_summary table

-- Step 5: Create inventory reservation log (audit trail)
INSERT INTO inventory_reservations (
  so_id, product_id, qty_reserved, warehouse_id, status, created_at
) VALUES (
  SO-2026-0001, 5001, 100, 1, 'RESERVED', NOW()
);
-- ROW LOCKED: inventory_reservations table

-- Step 6: Log inventory movement
INSERT INTO inventory_ledger (
  movement_type, product_id, warehouse_id, qty_in, qty_out, 
  qty_reserved, reference, timestamp
) VALUES (
  'RESERVATION', 5001, 1, 0, 0, 100, 'SO-2026-0001', NOW()
);
-- ROW LOCKED: inventory_ledger table

-- Step 7: Update customer expected AR (informational)
UPDATE customers SET
  expected_ar = expected_ar + 53100,
  last_order_date = 2026-09-05
WHERE id = 101;
-- ROW LOCKED: customers table

-- Step 8: Commit all changes atomically
COMMIT TRANSACTION;
-- All locks released, all changes permanent
```

**Consistency Guarantees:**
- ✅ If any step fails (e.g., inventory < qty), entire transaction rolls back
- ✅ SO created only if inventory reserved successfully
- ✅ Inventory never oversold (reserved qty ≤ available qty)
- ✅ Customer AR always matches sum of unpaid invoices

---

### Example: Invoice Generation & GL Posting (Multi-Module Atomic)

```sql
BEGIN TRANSACTION;
SAVEPOINT sp_invoice_posting;

-- Step 1: Lock SO & verify status
SELECT * FROM sales_orders WHERE id = SO-2026-0001 FOR UPDATE;
-- Pre-check: status = 'DELIVERED' ✓

-- Step 2: Lock GRN/Challan & verify delivery
SELECT * FROM delivery_challans WHERE so_id = SO-2026-0001 FOR UPDATE;
-- Pre-check: status = 'DELIVERED' ✓

-- Step 3: Create invoice
INSERT INTO tax_invoices (
  invoice_id, invoice_number, company_id, customer_id, so_id, challan_id,
  invoice_date, due_date, status, subtotal, sgst_amount, cgst_amount, 
  total_tax, grand_total
) VALUES (
  INV-2026-0001, INV-2026-0001, 1, 101, SO-2026-0001, DC-2026-0001,
  2026-09-11, 2026-10-11, 'DRAFTED', 45000, 4050, 4050, 8100, 53100
);
-- ROW LOCKED: tax_invoices table

-- Step 4: Create invoice line items
INSERT INTO tax_invoice_line_items (
  invoice_id, product_id, hsn_code, quantity, unit_price, line_amount, 
  tax_rate, tax_amount
) VALUES (
  INV-2026-0001, 5001, '3208', 100, 450, 45000, 18, 8100
);
-- ROW LOCKED: tax_invoice_line_items table

-- Step 5: Lock GL accounts & post AR entry
SELECT * FROM chart_of_accounts WHERE account_id IN (1010, 4010, 2020, 2030) FOR UPDATE;
-- Account 1010 (AR): Balance before = ₹30,000
-- Account 4010 (Sales): Balance before = ₹0
-- Account 2020 (SGST): Balance before = ₹0
-- Account 2030 (CGST): Balance before = ₹0

-- Create GL entry header
INSERT INTO gl_entries (
  entry_date, reference_document, description, status, total_debits, total_credits
) VALUES (
  2026-09-11, 'INV-2026-0001', 'Sales invoice ABC Trading', 'POSTED', 53100, 53100
);
-- ROW LOCKED: gl_entries table
-- Get entry_id = 1001 (auto-generated)

-- Create GL entry line 1 (AR debit)
INSERT INTO gl_entry_details (
  entry_id, account_id, debit, credit, cost_center
) VALUES (
  1001, 1010, 53100, 0, NULL
);
-- Update account balance:
UPDATE chart_of_accounts SET
  current_balance = current_balance + 53100
WHERE account_id = 1010;
-- After: AR balance = ₹30,000 + ₹53,100 = ₹83,100

-- Create GL entry lines 2-4 (Revenue & Tax credits)
INSERT INTO gl_entry_details (entry_id, account_id, debit, credit, cost_center)
VALUES (1001, 4010, 0, 45000, 'SALES');
UPDATE chart_of_accounts SET current_balance = current_balance + 45000 WHERE account_id = 4010;

INSERT INTO gl_entry_details (entry_id, account_id, debit, credit, cost_center)
VALUES (1001, 2020, 0, 4050, 'TAX');
UPDATE chart_of_accounts SET current_balance = current_balance + 4050 WHERE account_id = 2020;

INSERT INTO gl_entry_details (entry_id, account_id, debit, credit, cost_center)
VALUES (1001, 2030, 0, 4050, 'TAX');
UPDATE chart_of_accounts SET current_balance = current_balance + 4050 WHERE account_id = 2030;

-- Step 6: Post COGS & inventory deduction
-- Lock inventory accounts
SELECT * FROM chart_of_accounts WHERE account_id IN (5010, 1040) FOR UPDATE;
-- Account 5010 (COGS): Balance before = ₹0
-- Account 1040 (Inventory): Balance before = ₹45,000

-- Create separate GL entry for COGS
INSERT INTO gl_entries (
  entry_date, reference_document, description, status, total_debits, total_credits
) VALUES (
  2026-09-11, 'INV-2026-0001', 'COGS for invoice', 'POSTED', 30000, 30000
);
-- Get entry_id = 1002

INSERT INTO gl_entry_details (entry_id, account_id, debit, credit)
VALUES (1002, 5010, 30000, 0);
UPDATE chart_of_accounts SET current_balance = current_balance + 30000 WHERE account_id = 5010;

INSERT INTO gl_entry_details (entry_id, account_id, debit, credit)
VALUES (1002, 1040, 0, 30000);
UPDATE chart_of_accounts SET current_balance = current_balance - 30000 WHERE account_id = 1040;

-- Step 7: Update inventory ledger (COGS deduction)
UPDATE stock_summary SET
  available_qty = available_qty, -- No change, already reserved
  reserved_qty = 0, -- Release reservation (fulfilled)
  inventory_value = 50 * 300, -- 50 units × ₹300 = ₹15,000
  updated_at = NOW()
WHERE product_id = 5001;

INSERT INTO inventory_ledger (
  movement_type, product_id, qty_out, unit_cost, total_cost, reference
) VALUES (
  'COGS_DEDUCTION', 5001, 100, 300, 30000, 'INV-2026-0001'
);

-- Step 8: Update AR subledger
INSERT INTO customer_invoices (
  customer_id, invoice_id, invoice_amount, due_date, status
) VALUES (
  101, INV-2026-0001, 53100, 2026-10-11, 'UNPAID'
);

UPDATE customers SET
  current_ar_balance = 83100,
  last_invoice_date = 2026-09-11
WHERE id = 101;

-- Step 9: Update SO status
UPDATE sales_orders SET
  status = 'INVOICED',
  invoiced_date = 2026-09-11
WHERE id = SO-2026-0001;

-- Step 10: Verify reconciliation
-- AR GL account (1010) should = sum of customer_invoices
SELECT SUM(invoice_amount) FROM customer_invoices WHERE status = 'UNPAID';
-- Should match chart_of_accounts.current_balance for account_id = 1010

-- Commit all changes
COMMIT TRANSACTION;
-- All locks released, invoice fully posted
```

**Critical Guarantees:**
1. **Atomicity:** All GL entries created together, or none
2. **Consistency:** AR GL balance = Sum of customer unpaid invoices (verified in step 10)
3. **Isolation:** No other transaction can interfere (all rows locked)
4. **Durability:** Once committed, data permanent even if system fails

---

## 2.2 Foreign Key & Referential Integrity Map

```
┌─────────────────────────────────────────────────────────────────┐
│                      CORE FK RELATIONSHIPS                      │
└─────────────────────────────────────────────────────────────────┘

SALES MODULE RELATIONSHIPS:
├─ sales_orders.company_id → companies.id
├─ sales_orders.customer_id → customers.id
├─ sales_orders.quotation_id → quotations.id (optional, for traceability)
├─ sales_order_line_items.so_id → sales_orders.id
├─ sales_order_line_items.product_id → products.id
├─ delivery_challans.so_id → sales_orders.id
├─ delivery_challans.customer_id → customers.id
├─ delivery_challan_items.challan_id → delivery_challans.id
├─ delivery_challan_items.product_id → products.id
├─ tax_invoices.company_id → companies.id
├─ tax_invoices.customer_id → customers.id
├─ tax_invoices.so_id → sales_orders.id (links invoice to original order)
├─ tax_invoices.challan_id → delivery_challans.id (links to shipment)
├─ tax_invoice_line_items.invoice_id → tax_invoices.id
├─ tax_invoice_line_items.product_id → products.id
├─ payment_receipts.invoice_id → tax_invoices.id
└─ e_way_bills.invoice_id → tax_invoices.id

PURCHASE MODULE RELATIONSHIPS:
├─ purchase_requisitions.company_id → companies.id
├─ purchase_requisitions.requester_id → users.id
├─ purchase_orders.company_id → companies.id
├─ purchase_orders.vendor_id → suppliers.id
├─ purchase_orders.pr_id → purchase_requisitions.id
├─ purchase_order_line_items.po_id → purchase_orders.id
├─ purchase_order_line_items.product_id → products.id
├─ goods_receipts.company_id → companies.id
├─ goods_receipts.po_id → purchase_orders.id
├─ goods_receipt_items.grn_id → goods_receipts.id
├─ goods_receipt_items.po_line_id → purchase_order_line_items.id
├─ bills_payable.company_id → companies.id
├─ bills_payable.vendor_id → suppliers.id
├─ bills_payable.po_id → purchase_orders.id
├─ bill_line_items.bill_id → bills_payable.id
├─ bill_line_items.product_id → products.id
└─ payment_vouchers.bill_id → bills_payable.id

INVENTORY MODULE RELATIONSHIPS:
├─ stock_summary.company_id → companies.id
├─ stock_summary.product_id → products.id
├─ stock_summary.warehouse_id → warehouses.id
├─ inventory_ledger.company_id → companies.id
├─ inventory_ledger.product_id → products.id
├─ inventory_ledger.warehouse_id → warehouses.id
├─ inventory_reservations.so_id → sales_orders.id
├─ inventory_reservations.product_id → products.id
├─ inventory_batches.product_id → products.id
├─ inventory_batches.warehouse_id → warehouses.id
└─ physical_counts.warehouse_id → warehouses.id

GL MODULE RELATIONSHIPS:
├─ gl_entries.company_id → companies.id
├─ gl_entry_details.entry_id → gl_entries.id
├─ gl_entry_details.account_id → chart_of_accounts.id
├─ chart_of_accounts.company_id → companies.id
├─ bank_accounts.company_id → companies.id
├─ bank_accounts.gl_account_id → chart_of_accounts.id (Link GL to bank)
├─ bank_reconciliations.company_id → companies.id
├─ bank_reconciliations.bank_account_id → bank_accounts.id
├─ customer_invoices.customer_id → customers.id
├─ customer_invoices.invoice_id → tax_invoices.id
├─ vendor_invoices.vendor_id → suppliers.id
├─ vendor_invoices.bill_id → bills_payable.id
└─ tax_ledger.company_id → companies.id

POS MODULE RELATIONSHIPS:
├─ pos_terminals.warehouse_id → warehouses.id
├─ pos_transactions.terminal_id → pos_terminals.id
├─ pos_transaction_items.transaction_id → pos_transactions.id
├─ pos_transaction_items.product_id → products.id
└─ cash_drawer_logs.terminal_id → pos_terminals.id

MULTI-TENANT CONSTRAINT (CRITICAL):
├─ Every table includes company_id
├─ FK validation: All related tables must have same company_id
├─ Example: SO.company_id = customers.company_id = products.company_id
└─ Runtime check: WHERE company_id = :current_user_company_id
```

---

## 2.3 Event Triggers & Message Queue Workflows

```
SO Created Event:
├─ Trigger: After INSERT on sales_orders WHERE status = 'CONFIRMED'
├─ Action: Publish SO_CONFIRMED event to message queue
├─ Consumers:
│   ├─ Warehouse service: Generate pick list
│   ├─ Customer service: Send "Order Received" email
│   ├─ Notification service: Alert sales rep
│   └─ Analytics service: Record new order event
└─ Implementation: Event-driven architecture with Kafka/RabbitMQ

SO Invoiced Event:
├─ Trigger: After UPDATE on sales_orders WHERE status = 'INVOICED'
├─ Action: Publish SO_INVOICED event
├─ Consumers:
│   ├─ GL service: Verify GL entries posted
│   ├─ Customer service: Send invoice email with PDF
│   ├─ AR aging service: Add to customer AR subledger
│   ├─ GSTR-1 service: Update sales register
│   └─ Analytics service: Record invoice event
└─ Implementation: Real-time, async processing

PO Created Event:
├─ Trigger: After INSERT on purchase_orders WHERE status = 'SENT'
├─ Action: Publish PO_CREATED event
├─ Consumers:
│   ├─ Vendor service: Send PO email
│   ├─ Procurement service: Track PO status
│   └─ Budget service: Update commitment against budget
└─ Implementation: Event streaming

GRN Posted Event:
├─ Trigger: After INSERT on goods_receipts WHERE status = 'RECEIVED'
├─ Action: Publish GRN_RECEIVED event
├─ Consumers:
│   ├─ Inventory service: Update stock, post GL
│   ├─ Warehouse service: Update location bins
│   ├─ AP service: Flag for bill matching
│   └─ Reorder service: Alert if stock > reorder point
└─ Implementation: Real-time GL posting triggered by GRN event

Invoice Paid Event:
├─ Trigger: After INSERT on payment_receipts (with matching to invoice)
├─ Action: Publish INVOICE_PAID event
├─ Consumers:
│   ├─ AR service: Clear invoice, update DSO
│   ├─ Finance service: Post GL payment entry
│   ├─ Customer service: Send receipt email
│   └─ Analytics service: Track collection metrics
└─ Implementation: Real-time, affects AR aging immediately
```

---

# PART 3: STATE MACHINES & WORKFLOW TRANSITIONS

---

## 3.1 Key Entity State Diagrams

### Sales Order States (Complete)

```
DRAFT
  ├─ Actions: Edit, Delete
  ├─ Trigger: Create from SO form
  └─→ SUBMITTED (Finance approves credit)
       ├─ Actions: Edit, Cancel
       ├─ Validation: Credit OK, Stock available
       └─→ CONFIRMED (Inventory reserved)
            ├─ Actions: View, Track, Adjust qty (before picking)
            ├─ Event: SO_CONFIRMED (publish to queue)
            ├─ Inventory: 100 units reserved for this SO
            └─→ PICKING (Warehouse picks items)
                 ├─ Actions: View, Track
                 ├─ Physical: Items removed from shelf
                 └─→ PICKED (QC verifies)
                      ├─ Actions: View, Track, Hold if QC fail
                      └─→ PACKED (Goods packed in boxes)
                           ├─ Actions: View, Track
                           └─→ DISPATCHED (Handed to courier)
                                ├─ Actions: View, Track, Generate e-way bill
                                ├─ Event: SO_DISPATCHED (customer notified)
                                └─→ DELIVERED (Customer receives)
                                     ├─ Actions: View, Invoice, Return (if damaged)
                                     └─→ INVOICED (Tax invoice generated & GL posted)
                                          ├─ Actions: View, Track payment
                                          ├─ Event: SO_INVOICED (GL entries immutable)
                                          └─→ PAID (Payment received & matched)
                                               ├─ Actions: View, Close
                                               └─→ CLOSED (Order complete, AR cleared)

[Alternative paths]
CONFIRMED → QUALITY_HOLD (QC rejected items)
CONFIRMED → BACK_ORDER (Stock insufficient, partial fulfillment)
ANY STATE → CANCELLED (Customer or internal cancellation)
ANY STATE → REJECTED (Credit or validation failure)
```

### Invoice States (Complete)

```
DRAFT
  ├─ Actions: Edit, Delete (before posting)
  ├─ Trigger: Create from SO delivery
  ├─ GL: NOT posted yet
  └─→ ISSUED (Approved, e-invoice generated)
       ├─ Actions: Send to customer, View
       ├─ E-Invoice: IRN assigned, QR embedded
       ├─ Event: INVOICE_ISSUED
       └─→ POSTED (GL entries created, immutable)
            ├─ Actions: View, Receive payment, Generate CN
            ├─ GL: Locked (no edit allowed)
            ├─ AR Subledger: Invoice added to customer balance
            ├─ Tax: GSTR-1 updated
            └─→ PAID (Payment received, matched, AR cleared)
                 ├─ Actions: View, Close
                 ├─ Event: INVOICE_PAID
                 └─→ CLOSED (Order complete)

[Alternative paths]
DRAFT → REJECTED (Before posting)
POSTED → OVERDUE (If due date passed without payment)
POSTED → DISPUTED (Customer contests amount)
POSTED → REVERSED (Via credit note, not deletion)
```

### PO States (Complete)

```
DRAFT
  ├─ Actions: Edit, Delete
  ├─ Trigger: Create from PR or manual
  ├─ GL: NOT posted
  └─→ SENT (Approved, email to vendor)
       ├─ Actions: View, Track, Cancel
       ├─ Event: PO_SENT
       ├─ Vendor: Receives email with PO details
       └─→ ACKNOWLEDGED (Vendor confirms receipt)
            ├─ Actions: View, Track
            ├─ Lead time timer: Starts counting to delivery_date
            └─→ PARTIAL_RECEIPT (GRN created for partial qty)
                 ├─ Actions: View, Track, Receive remaining
                 ├─ Inventory: Partial stock updated
                 ├─ AP: Waiting for final invoice
                 └─→ FULLY_RECEIVED (GRN for remaining qty)
                      ├─ Actions: View, Match with bill
                      ├─ Inventory: Full qty received
                      ├─ GL: Posted on final GRN
                      └─→ CLOSED (Bill matched, paid)
                           ├─ Actions: View, Archive
                           └─→ [TERMINAL]

[Alternative paths]
SENT → REJECTED (Vendor declined)
ACKNOWLEDGED → OVERDUE (Delivery delayed past due date)
PARTIAL_RECEIPT → CANCELLED (Remaining qty cancelled)
ANY STATE → CANCELLED (Internal cancellation)
```

### Bill States (Complete)

```
RECEIVED
  ├─ Actions: Review, Reject
  ├─ Trigger: Vendor invoice received
  ├─ GL: NOT posted yet
  └─→ MATCHED (3-way match passed: PO ↔ GRN ↔ Bill)
       ├─ Actions: Approve, Hold, Dispute
       ├─ System: 3-way match verification
       └─→ APPROVED (Finance approved)
            ├─ Actions: Schedule payment
            ├─ Event: BILL_APPROVED
            ├─ GL: Posted (expense + input tax + AP)
            └─→ PAID (Payment made, TDS deducted)
                 ├─ Actions: Reconcile, Close
                 ├─ Event: BILL_PAID
                 ├─ AP Subledger: Vendor balance cleared
                 └─→ CLOSED (Order complete)

[Alternative paths]
RECEIVED → DISPUTED (Amount mismatch)
MATCHED → ON_HOLD (Waiting for clarification)
APPROVED → OVERDUE (If due date passed before payment)
ANY STATE → RETURNED (Return/adjust for quality issues)
```

### Inventory States (Stock Levels)

```
Stock at Location (e.g., HQ Warehouse):
├─ Available: 50 units (can be sold/transferred)
├─ Reserved: 100 units (committed to pending SOs)
├─ In-Transit: 200 units (ordered but not received)
├─ Damaged: 10 units (pending inspection/return)
└─ Total Stock: 360 units

State Transitions:
├─ Available → Reserved (when SO confirmed)
├─ Reserved → Available (when SO cancelled)
├─ Reserved → Fulfilled (when invoice issued)
├─ In-Transit → Available (when GRN posted)
├─ Damaged → Available (when rework approved)
├─ Damaged → Scrap (when written off)
└─ Available → Damaged (when inspection fails)
```

---

# PART 4: ATOMIC TRANSACTIONS & CONSISTENCY GUARANTEES

---

## 4.1 ACID Properties in Action

### Atomicity: All-or-Nothing

```
Scenario: SO creation fails at inventory reservation

SO Creation Transaction:
├─ Step 1: Validate customer ✓
├─ Step 2: Create SO ✓
├─ Step 3: Create SO line items ✓
├─ Step 4: Reserve inventory ✗ (FAIL: Stock insufficient)
└─ Result: ROLLBACK

Outcome:
├─ SO deleted (not created)
├─ SO line items deleted
├─ Inventory unchanged (reservation never happened)
└─ Customer AR unchanged (no obligation created)

Guarantee: SO doesn't exist partially. Either fully created or not at all.
```

### Consistency: Referential Integrity Always Maintained

```
Scenario: Delete customer (attempt)

System Checks:
├─ Query: SELECT COUNT(*) FROM sales_orders WHERE customer_id = 101
├─ Result: 5 open SOs exist
├─ FK constraint on sales_orders.customer_id → customers.id
├─ Outcome: DELETE customers WHERE id = 101
│   → ERROR: Foreign key violation
│   → Cannot delete customer with linked sales orders
└─ Result: Customer not deleted, data consistent

Guarantee: Cannot delete customers with active SOs. Referential integrity maintained.
```

### Isolation: Concurrent Transactions Don't Interfere

```
Scenario: Two finance staff process payment simultaneously

Transaction 1 (by Raj):
├─ BEGIN
├─ SELECT payment_receipts WHERE invoice_id = INV-2026-0001
├─ Record: ₹53,100 received
├─ UPDATE customer_invoices SET status = 'PAID'
└─ COMMIT

Transaction 2 (by Priya):
├─ BEGIN
├─ SELECT payment_receipts WHERE invoice_id = INV-2026-0001
├─ Result: Waiting (row locked by Raj's transaction)
├─ BLOCKED until Raj commits
└─ After Raj commits:
    ├─ Priya sees updated data (payment already recorded)
    ├─ Detects duplicate, aborts
    └─ Result: No double-posting

Guarantee: Lock prevents duplicate payment recording.
```

### Durability: Committed Data Survives System Failure

```
Scenario: System crash during invoice posting

State Before Crash:
├─ Invoice entry in progress
├─ GL entries partially posted

Outcome:
├─ If crash before COMMIT: Everything rolled back
│   └─ Invoice not created, GL not posted
├─ If crash after COMMIT: Everything durable
│   └─ Invoice exists, GL posted, even after recovery
└─ Result: No partial/orphaned data

Guarantee: COMMIT ensures durability to disk.
```

---

# PART 5: SOFT-LAUNCH CRITICAL PATH & DEPENDENCIES

---

## 5.1 Critical Path to Go-Live

### Phase 0: Core Infrastructure (Week 1-2, Must Complete First)

```
Prerequisite:
├─ Database schema deployed (all tables, FK constraints, indices)
├─ Company master created (Demo company for testing)
├─ Chart of Accounts configured (Sales, Expense, AR, AP, Inventory, Tax)
├─ Product master seeded (500 test products with HSN codes)
├─ Customer master seeded (50 test customers)
├─ Vendor master seeded (20 test suppliers)
├─ Warehouse setup (2-3 test locations)
├─ GL account reconciliation queries tested
└─ → Ready for module-level testing
```

### Phase 1: Sales Module (MVP — Weeks 3-4)

```
Dependency Chain:
├─ Customers table (required for SO creation)
├─ Products table with HSN codes (required for tax calc)
├─ Inventory/Stock table (required for availability check)
├─ Chart of Accounts (required for GL posting)
│
└─ Workflows to implement:
   ├─ Quotation creation & conversion (DRAFT → SUBMITTED → ACCEPTED)
   ├─ Sales Order creation with credit check (CONFIRMED state)
   ├─ Inventory reservation (atomic with SO creation)
   ├─ Delivery tracking (DC generation, status updates)
   ├─ Tax Invoice generation with e-Invoice (GSTR-1 ready)
   ├─ GL posting (AR, Revenue, Tax accounts)
   ├─ COGS deduction (inventory to GL)
   └─ Payment collection & AR subledger

Soft-Launch Gate (SLG):
├─ ✓ Create SO for all product categories (5%, 12%, 18%, 28% GST)
├─ ✓ Inventory reserved correctly (available qty reduced)
├─ ✓ Intra-state invoice generated (SGST + CGST split)
├─ ✓ Inter-state invoice generated (IGST instead)
├─ ✓ E-invoice IRN assigned (QR code in PDF)
├─ ✓ GL posting verified (AR/Rev/Tax balance)
├─ ✓ COGS posted (inventory value correct)
├─ ✓ Payment received, AR cleared (subledger reconciled)
├─ ✓ GSTR-1 data matches invoices (for compliance filing)
└─ → Sales module ready for live customer orders
```

### Phase 2: Purchase & Inventory (MVP — Weeks 3-4, Parallel with Sales)

```
Dependency Chain:
├─ Suppliers table (required for PO)
├─ Products table (required for GRN)
├─ Chart of Accounts (required for GL posting)
│
└─ Workflows to implement:
   ├─ Purchase Requisition (stock alert trigger)
   ├─ Purchase Order creation (to suppliers)
   ├─ GRN receipt & 3-way matching
   ├─ Inventory update (stock replenished)
   ├─ GL posting (Inventory + AP)
   ├─ Input tax tracking (ITC eligible)
   ├─ Vendor bill matching (PO ↔ GRN ↔ Bill)
   ├─ Payment & TDS deduction
   └─ AP subledger aging

Soft-Launch Gate (SLG):
├─ ✓ PO created, sent to vendor
├─ ✓ GRN received, qty matched, inventory updated
├─ ✓ Vendor invoice matched (3-way OK)
├─ ✓ GL posted (Inventory + Input Tax + AP)
├─ ✓ Payment made with TDS deduction
├─ ✓ AP subledger reconciled
├─ ✓ Stock replenishment working (low stock alerts)
└─ → Purchase module ready for vendor operations
```

### Phase 3: GL & Accounting (MVP — Weeks 4-5)

```
Dependency Chain:
├─ Sales module complete (invoices generating GL entries)
├─ Purchase module complete (bills generating GL entries)
├─ Chart of Accounts configured (all accounts created)
├─ Bank account setup (GL linked to bank)
│
└─ Workflows to implement:
   ├─ Real-time GL posting (from SO/PO/Invoice events)
   ├─ AR subledger reconciliation (GL ↔ customer invoices)
   ├─ AP subledger reconciliation (GL ↔ vendor bills)
   ├─ Bank reconciliation (GL ↔ bank statement)
   ├─ GST reconciliation (GSTR-1/3B auto-compilation)
   ├─ Month-end close (accruals, adjustments)
   ├─ Financial statements generation (P&L, Balance Sheet)
   └─ Audit trail (immutable GL entries)

Soft-Launch Gate (SLG):
├─ ✓ SO → Invoice → GL posting (end-to-end)
├─ ✓ PO → Bill → GL posting (end-to-end)
├─ ✓ AR GL (1010) = Sum of customer invoices ✓
├─ ✓ AP GL (2010) = Sum of vendor bills ✓
├─ ✓ Inventory GL (1040) = Stock valuation ✓
├─ ✓ Bank GL (1020) = Bank statement balance ✓
├─ ✓ GST output = GSTR-1 invoices (match)
├─ ✓ GST input = Purchase bills (tracked)
├─ ✓ P&L generated (revenue, COGS, profit)
├─ ✓ Balance Sheet balanced (assets = liabilities + equity)
└─ → GL module ready for financial close
```

### Phase 4: POS (MVP — Weeks 5-6, Parallel Option)

```
Dependency Chain:
├─ Products table (barcode mapping)
├─ Inventory (stock management)
├─ Chart of Accounts (GL posting)
├─ POS terminal setup (terminal master)
│
└─ Workflows to implement:
   ├─ Barcode scanning & cart building
   ├─ Real-time tax calculation (by HSN)
   ├─ Instant inventory deduction
   ├─ Real-time GL posting (cash + revenue + tax)
   ├─ Cash drawer reconciliation
   ├─ End-of-shift reporting
   └─ Offline mode (sync when online restored)

Soft-Launch Gate (SLG):
├─ ✓ Scan barcode, item added to cart with correct price
├─ ✓ Tax calculated correctly (5%, 12%, 18%, 28%)
├─ ✓ Inventory reduced immediately (real-time)
├─ ✓ GL posting at POS time (not batched)
├─ ✓ Cash drawer reconciliation (physical ↔ system)
├─ ✓ End-of-shift report correct (∑ sales = GL cash)
└─ → POS ready for retail operations
```

### Phase 5: CRM (MVP — Week 6, Optional for Launch)

```
Dependency Chain:
├─ Customers table (master)
├─ Sales orders (order history)
├─ Invoices (transaction history)
│
└─ Workflows to implement:
   ├─ Customer profile (contact, credit limit, history)
   ├─ Order history (list of all SOs/invoices)
   ├─ AR aging (payment behavior)
   ├─ Repeat purchase tracking (DSO, frequency)
   └─ Sales pipeline (opportunity tracking, optional)

Soft-Launch Gate (SLG):
├─ ✓ Customer master complete (name, contact, GSTIN)
├─ ✓ Order history visible (list of orders & invoices)
├─ ✓ AR aging calculated (current, 30, 60, 90+ days)
└─ → CRM ready (basic version, enough for launch)
```

---

## 5.2 Go-Live Criteria (Hard Requirements)

### Before Soft Launch, All Cells Must Be ✓:

```
┌──────────────────┬──────────────────┬──────────────────┬──────────────┐
│   Module         │  Workflow        │  Test Status     │  Gate        │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ Sales            │ Quotation → SO   │ ✓ Tested         │ GO           │
│                  │ SO → Invoice     │ ✓ Tested         │ GO           │
│                  │ Invoice → Paid   │ ✓ Tested         │ GO           │
│                  │ E-invoice (IRN)  │ ✓ NSDL API ready │ GO           │
│                  │ GST calc (all)   │ ✓ 5/12/18/28%    │ GO           │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ Purchase         │ PR → PO          │ ✓ Tested         │ GO           │
│                  │ PO → GRN         │ ✓ Tested         │ GO           │
│                  │ 3-way match      │ ✓ Tested         │ GO           │
│                  │ TDS deduction    │ ✓ Tested         │ GO           │
│                  │ Input GST track  │ ✓ Tested         │ GO           │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ Inventory        │ Reservation      │ ✓ Atomic OK      │ GO           │
│                  │ FIFO COGS        │ ✓ Cost calc OK   │ GO           │
│                  │ Replenish alert  │ ✓ Threshold OK   │ GO           │
│                  │ Multi-warehouse  │ ✓ Transfer OK    │ GO           │
│                  │ Batch/Expiry     │ ✓ Perishables OK │ GO           │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ GL               │ Real-time post   │ ✓ No delay       │ GO           │
│                  │ Reconciliation   │ ✓ 100% daily     │ GO           │
│                  │ Bank recon       │ ✓ Automated      │ GO           │
│                  │ Financial close  │ ✓ Month-end OK   │ GO           │
│                  │ P&L & BS         │ ✓ Balanced       │ GO           │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ POS              │ Barcode scan     │ ✓ Fast (<1s)     │ GO           │
│                  │ Inventory deduct │ ✓ Real-time      │ GO           │
│                  │ GL posting       │ ✓ Immediate      │ GO           │
│                  │ Cash recon       │ ✓ Accuracy OK    │ GO           │
│                  │ Offline mode     │ ✓ Sync OK        │ GO           │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ Tax Compliance   │ GSTR-1 filing    │ ✓ Data ready     │ GO           │
│                  │ GSTR-3B filing   │ ✓ Payment calc   │ GO           │
│                  │ TDS Form 26Q     │ ✓ Deduction track│ GO           │
│                  │ E-Way Bill       │ ✓ NSDL API       │ GO           │
│                  │ Audit trail      │ ✓ Immutable      │ GO           │
├──────────────────┼──────────────────┼──────────────────┼──────────────┤
│ Data Quality     │ Referential      │ ✓ FK checks      │ GO           │
│                  │ Atomicity        │ ✓ ACID OK        │ GO           │
│                  │ Reconciliation   │ ✓ Daily pass     │ GO           │
│                  │ Master data      │ ✓ Complete       │ GO           │
└──────────────────┴──────────────────┴──────────────────┴──────────────┘

Hard Stops (If any NOT ✓, Delay Launch):
├─ E-invoice IRN generation fails
├─ GST subledger (GSTR-1) doesn't match invoices
├─ AR GL ≠ sum of customer invoices (reconciliation broken)
├─ POS inventory deduction delayed (>5 second lag)
├─ Bank reconciliation can't be balanced
└─ Atomicity fail: Partial SO created (FK violated)
```

---

## 5.3 First Customer Onboarding Checklist

### For Each Regional Customer Going Live:

```
Pre-Launch Setup:
├─ Customers master created (customer_id, GSTIN, credit limit)
├─ Billing address configured
├─ Shipping address configured
├─ Payment terms set (Net 30, Net 45, 2/10 Net 30)
├─ Products configured (those customer will order)
├─ HSN codes verified (tax rates match customer expectations)
├─ Pricing configured (wholesale or retail rate)
├─ Credit limit set (based on credit evaluation)
└─ → Ready for order entry

Day 1: First Order Creation
├─ Sales rep creates SO (pilot order, small qty)
├─ Inventory reserved successfully
├─ Order confirmed, warehouse notified
└─ → Success: Workflow operational

Day 3: Order Fulfillment
├─ Goods picked, packed, dispatched
├─ Delivery challan generated
├─ Tracking shared with customer
└─ → Success: Fulfillment working

Day 5: Invoice Generation
├─ Tax invoice created
├─ E-invoice IRN assigned
├─ GL posting verified (AR/Revenue/Tax)
├─ PDF with QR sent to customer
└─ → Success: Invoicing working

Day 20: Payment Collection
├─ Customer pays (via bank transfer, check, or card)
├─ Payment recorded & matched to invoice
├─ AR subledger cleared
├─ GL bank entry posted
└─ → Success: Payment & AR cleared

End of Month: Close & Reporting
├─ Invoices aged correctly (current, 30, 60+ days)
├─ GSTR-1 includes customer invoices
├─ Month-end close completed
├─ Financial statements generated
└─ → Success: Reporting & compliance working

Go-Live Sign-Off:
├─ ✓ All workflows completed successfully
├─ ✓ No data integrity issues
├─ ✓ No unresolved GL reconciliations
├─ ✓ Customer satisfied with speed & accuracy
└─ → Customer confirmed go-live, production operations begin
```

---

# PART 6: API CONTRACT SPECIFICATIONS

---

## 6.1 Key REST APIs (Examples)

### Sales Order Creation API

```
POST /api/v1/sales-orders

Request Body:
{
  "company_id": 1,
  "customer_id": 101,
  "quotation_id": "QT-2026-0001",  // optional, for traceability
  "order_date": "2026-09-05",
  "delivery_date": "2026-09-10",
  "line_items": [
    {
      "product_id": 5001,
      "quantity": 100,
      "unit_price": 450
    }
  ],
  "notes": "Urgent delivery required"
}

Validation (API Layer):
├─ company_id exists (FK check)
├─ customer_id exists & status = 'ACTIVE' (FK check)
├─ quotation_id (if provided) matches customer & company (FK check)
├─ All product_ids exist (FK check)
├─ delivery_date >= order_date (logical check)
├─ quantity > 0 (business rule)
└─ → If all checks pass: Proceed to database transaction

Database Transaction (Atomic):
├─ Lock customer (credit check)
├─ Lock products (for each line item)
├─ Lock stock_summary (for inventory check)
├─ Create SO, SO line items, reservations, ledger entries
├─ Update customer expected AR
├─ Publish SO_CONFIRMED event
└─ → If any step fails: ROLLBACK entire transaction

Response (201 Created):
{
  "so_id": "SO-2026-0001",
  "status": "CONFIRMED",
  "total_amount": 45000,
  "total_tax": 8100,
  "grand_total": 53100,
  "inventory_reserved": 100,
  "created_at": "2026-09-05T10:15:00Z",
  "message": "SO created successfully, inventory reserved"
}

Error Response (400 Bad Request):
{
  "error_code": "CREDIT_LIMIT_EXCEEDED",
  "message": "Customer credit limit exceeded: ₹70,000 available, SO amount ₹80,000",
  "customer_id": 101,
  "available_credit": 70000,
  "so_amount": 80000,
  "timestamp": "2026-09-05T10:15:00Z"
}

Error Response (409 Conflict):
{
  "error_code": "INVENTORY_INSUFFICIENT",
  "message": "Insufficient inventory for product Red Paint: 50 units available, 100 requested",
  "product_id": 5001,
  "available_qty": 50,
  "requested_qty": 100,
  "timestamp": "2026-09-05T10:15:00Z"
}
```

### Tax Invoice Generation API

```
POST /api/v1/tax-invoices

Request Body:
{
  "so_id": "SO-2026-0001",
  "challan_id": "DC-2026-0001",
  "invoice_date": "2026-09-11",
  "notes": "As per PO #XYZ"
}

Pre-checks (API Layer):
├─ SO exists & status = 'DELIVERED' (workflow state check)
├─ Challan exists & status = 'DELIVERED' (shipment verified)
├─ No prior invoice for this SO (duplicate prevention)
└─ → Proceed if all checks pass

GL Posting (Database Transaction, Atomic):
├─ Create tax_invoices & line items
├─ Calculate tax (by HSN, jurisdiction)
├─ Create 4 GL entries atomically:
│   ├─ DR AR (1010): Full invoice amount
│   ├─ CR Sales Revenue (4010): Taxable amount
│   ├─ CR SGST (2020): State tax
│   └─ CR CGST (2030): Central tax
├─ Create COGS entry:
│   ├─ DR COGS (5010): FIFO cost
│   └─ CR Inventory (1040): Reduce stock value
├─ Update AR subledger (customer_invoices table)
├─ Update SO status: 'INVOICED'
├─ Publish INVOICE_CREATED event
└─ → If any step fails: ROLLBACK entire transaction

E-Invoice Generation (Async, Post-Commit):
├─ Call NSDL API with invoice JSON
├─ Receive IRN & acknowledgement
├─ Update tax_invoices table (irn, ack_number)
├─ Generate PDF with QR code
├─ Publish INVOICE_E_INVOICED event
└─ → If NSDL fails: Retry logic (exponential backoff, max 3 attempts)

Response (201 Created):
{
  "invoice_id": "INV-2026-0001",
  "invoice_number": "INV-2026-0001",
  "status": "POSTED",
  "irn": "12345-IRP-2026",
  "ack_number": "123456789",
  "qr_code_url": "/invoices/qr/INV-2026-0001.png",
  "subtotal": 45000,
  "sgst": 4050,
  "cgst": 4050,
  "total_tax": 8100,
  "grand_total": 53100,
  "due_date": "2026-10-11",
  "gl_entries_created": 4,
  "cogs_posted": true,
  "ar_updated": true,
  "created_at": "2026-09-11T10:30:00Z"
}
```

### Payment Receipt API

```
POST /api/v1/payment-receipts

Request Body:
{
  "invoice_id": "INV-2026-0001",
  "amount": 53100,
  "payment_date": "2026-10-20",
  "payment_mode": "NEFT",
  "bank_reference": "UTR-2026NEFT001",
  "notes": "Full payment for INV-2026-0001"
}

Validation:
├─ Invoice exists & status = 'POSTED' (not draft, not already paid)
├─ amount = grand_total (exact match required)
├─ payment_date >= invoice_date (logical check)
└─ → Proceed if valid

GL Posting (Atomic Transaction):
├─ Create payment_receipts entry
├─ Create GL entry:
│   ├─ DR Bank (1020): Cash received
│   └─ CR AR (1010): Customer debt cleared
├─ Match payment to invoice (link in payment_receipts)
├─ Update tax_invoices status: 'PAID'
├─ Update AR subledger (customer_invoices):
│   ├─ status = 'PAID'
│   ├─ paid_date = 2026-10-20
│   └─ dso = days since invoice date
├─ Update customers table:
│   ├─ current_ar_balance -= invoice_amount
│   └─ last_payment_date = 2026-10-20
├─ Flag for bank reconciliation (next close)
├─ Publish INVOICE_PAID event
└─ → If any step fails: ROLLBACK

Response (201 Created):
{
  "receipt_id": "RCP-2026-0001",
  "invoice_id": "INV-2026-0001",
  "amount_received": 53100,
  "payment_date": "2026-10-20",
  "payment_mode": "NEFT",
  "status": "RECEIVED",
  "invoice_status_updated": "PAID",
  "ar_updated": true,
  "gl_posted": true,
  "dso": 39,
  "created_at": "2026-10-20T09:45:00Z"
}
```

---

# PART 7: ERROR HANDLING & ROLLBACK SCENARIOS

---

## 7.1 Common Failure Scenarios

### Scenario 1: Inventory Insufficient

```
User Action: Create SO for 100 units
Available Stock: 50 units

Workflow Execution:
├─ Step 1: Validate customer ✓
├─ Step 2: Create SO ✓
├─ Step 3: Create SO line items ✓
├─ Step 4: Lock stock_summary ✓
├─ Step 5: Check availability
│   ├─ Query: SELECT available_qty FROM stock_summary WHERE product_id = 5001
│   ├─ Result: 50 < 100 (required)
│   └─ Validation FAILED
│
├─ Step 6: ROLLBACK
│   ├─ Delete SO (created in step 2)
│   ├─ Delete SO line items (created in step 3)
│   └─ Release all locks
│
└─ Result:
    ├─ No SO created
    ├─ Inventory unchanged
    └─ Customer notified: "Insufficient inventory"

Option A: Wait for Stock:
├─ Create BACK-ORDER (qty 100, only 50 available)
├─ Reserve 50 units
├─ Wait for restock
└─ Once inventory replenished: Auto-fulfill back-order

Option B: Partial Fulfillment:
├─ Create SO for 50 units (available qty only)
├─ Customer receives 50, aware of shortage
└─ Re-order remaining 50 units later
```

### Scenario 2: GL Posting Fails

```
Trigger: Invoice generation attempted, GL unavailable

Workflow:
├─ Step 1: Create tax_invoices ✓
├─ Step 2: Create tax_invoice_line_items ✓
├─ Step 3: Lock GL accounts ✓
├─ Step 4: Post GL entry (debit AR)
│   ├─ DB call: INSERT gl_entries
│   ├─ DB call: INSERT gl_entry_details (line 1)
│   ├─ Update chart_of_accounts (line 1) ✓
│   ├─ DB call: INSERT gl_entry_details (line 2) ✗ NETWORK ERROR
│   └─ Connection lost to database
│
├─ Step 5: Exception handling
│   ├─ DB returns error (connection timeout)
│   ├─ Transaction context: ROLLBACK (automatic)
│   ├─ Delete gl_entries (step 4)
│   ├─ Delete gl_entry_details (line 1)
│   ├─ Restore chart_of_accounts balance
│   └─ Release all locks
│
├─ Step 6: Cleanup
│   ├─ Update tax_invoices.status = 'ERROR'
│   ├─ Log error message (for audit trail)
│   └─ Alert operations team
│
└─ Recovery:
    ├─ IT team checks GL availability
    ├─ Re-trigger invoice posting (manual or automatic retry)
    └─ GL posting succeeds on retry

Guarantee: Invoice not partially posted. Either fully posted or not at all.
```

### Scenario 3: E-Invoice IRN Generation Fails

```
Trigger: Tax invoice created, e-invoice API unreachable

Workflow:
├─ Step 1: Create tax_invoices ✓
├─ Step 2: GL posting ✓
├─ Step 3: Call NSDL API (async, post-commit)
│   ├─ API endpoint: https://nsdl-irnapi.incometax.gov.in/...
│   ├─ Request: Invoice JSON
│   ├─ Timeout: 30 seconds
│   └─ Error: 503 Service Unavailable (NSDL API down)
│
├─ Step 4: Exception handling
│   ├─ Catch HTTP error (503)
│   ├─ Invoice status: 'POSTED' (already in GL)
│   ├─ e_invoice_status: 'FAILED' (NSDL error)
│   ├─ Log error & retry flag
│   └─ Alert operations team
│
├─ Step 5: Retry mechanism (exponential backoff)
│   ├─ Wait 60 seconds, retry
│   ├─ Wait 120 seconds, retry
│   ├─ Wait 300 seconds, retry (max 3 attempts)
│   └─ If all fail: Escalate to manual intervention
│
└─ Recovery:
    ├─ Once NSDL API recovers
    ├─ Manual trigger: Re-send invoice to NSDL
    ├─ Receive IRN
    ├─ Update tax_invoices (irn, e_invoice_status = 'VALID')
    ├─ Generate & send PDF with QR to customer
    └─ e-invoice complete

Guarantee: GL posting not blocked by NSDL unavailability. Invoice is GL-posted and usable even if e-invoice delayed.
```

### Scenario 4: Stock Oversale (Concurrent Orders)

```
Scenario: Two salespeople create SOs simultaneously
Product: Red Paint, Available: 100 units
SO1 (Salesperson A): Order for 80 units
SO2 (Salesperson B): Order for 50 units

Execution (Database Isolation):

Transaction 1 (SO1, Salesperson A):
├─ BEGIN
├─ Lock stock_summary (product 5001)
├─ SELECT available_qty: 100 ✓
├─ Check: 100 >= 80? YES
├─ UPDATE available_qty = 100 - 80 = 20, reserved = 80
├─ Create SO1 (80 units reserved)
└─ COMMIT (Salesperson A finishes)

Transaction 2 (SO2, Salesperson B):
├─ BEGIN
├─ Tries to lock stock_summary (product 5001)
├─ BLOCKED (locked by Transaction 1 until commit)
├─ Waits...
├─ (Transaction 1 commits)
├─ Lock acquired
├─ SELECT available_qty: 20 (updated by T1)
├─ Check: 20 >= 50? NO ✗ FAILS
├─ ROLLBACK (SO2 not created)
└─ Error: "Insufficient inventory: 20 available, 50 requested"

Result:
├─ SO1: Created successfully (80 units reserved)
├─ SO2: Rejected (not created)
├─ Total stock: 20 units + 80 reserved = 100 (matches)
└─ No oversale occurred (DB locks prevented it)

Guarantee: Lock ensures no race condition. Second transaction waits, sees updated stock, correctly identifies shortage.
```

---

## 7.2 Rollback Triggers & Recovery

```
Automatic Rollback Triggers:
├─ FK constraint violation (e.g., customer_id not found)
├─ Unique constraint violation (e.g., SO_number duplicated)
├─ Check constraint violation (e.g., quantity ≤ 0)
├─ Out-of-range error (e.g., tax rate > 100%)
├─ Deadlock (two transactions waiting for each other)
├─ Connection timeout (network unavailable)
├─ Disk full (storage space exhausted)
└─ Out of memory (transaction too large)

Manual Rollback (For Corrections):
├─ Operational error (e.g., wrong SO amount entered)
├─ Data quality issue (e.g., duplicate SO created)
├─ Refund/Return (e.g., customer returns goods)
│
└─ Process:
    ├─ Create credit note (don't delete invoice)
    ├─ Post GL reversal entry (DR Revenue, CR AR)
    ├─ Update AR subledger (invoice status: 'REVERSED')
    └─ Maintain audit trail (original + reversal both visible)
```

---

## CONCLUSION

This specification provides the **production-grade workflow orchestration blueprint** for Niyanthra's soft launch:

1. ✅ **Core workflows detailed** (O2C, P2P, Inventory, POS, Record-to-Report)
2. ✅ **State machines defined** (entity transitions for SO, Invoice, PO, Bill)
3. ✅ **Database atomicity guaranteed** (ACID principles, no partial posts)
4. ✅ **Module interconnectivity mapped** (FK relationships, event triggers)
5. ✅ **Soft-launch critical path outlined** (phased rollout, go-live gates)
6. ✅ **API contracts specified** (request/response, validation, error handling)
7. ✅ **Error scenarios planned** (rollback triggers, recovery procedures)

**For Engineering Team:**
- API endpoints can be built from section 6.1 (REST specifications)
- DB migrations from section 2 (FK relationships, constraints)
- Business logic from section 1 (step-by-step workflows)
- Error handling from section 7 (rollback procedures)

**For Implementation Lead:**
- Critical path in section 5.2 (phase ordering, dependencies)
- Go-live checklist in section 5.3 (what must pass before launch)
- First customer setup in section 5.3 (onboarding steps)

**For Product:**
- State diagrams in section 3 (what customers see, what internal processes happen)
- Workflow details in sections 1 & 4 (ACID guarantees, no data loss)

