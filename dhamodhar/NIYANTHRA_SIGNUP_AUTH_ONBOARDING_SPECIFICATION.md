# 🔐 Niyanthra ERP — Sign-Up, Authentication & Onboarding Specification

**Version:** 1.0 (Production Edition)  
**Date:** September 2026  
**Classification:** INTERNAL — Product, Engineering & Security Use  
**Audience:** Product Managers, Frontend Engineers, Backend Engineers, Security Teams

---

## Table of Contents

1. [Executive Overview](#executive-overview)
2. [Architectural Sign-Up & Access Models](#architectural-sign-up--access-models)
3. [Step-by-Step UI/UX Wireframe Flow](#step-by-step-uiux-wireframe-flow)
4. [Security, Data Isolation & Multi-Tenancy Rules](#security-data-isolation--multi-tenancy-rules)
5. [Implementation Guidelines for Developers](#implementation-guidelines-for-developers)
6. [Database State Machine](#database-state-machine)
7. [Error Handling & Edge Cases](#error-handling--edge-cases)

---

## Executive Overview

Niyanthra's onboarding is **organization-first, not user-first**. This is fundamentally different from consumer SaaS (where individuals sign up) and critical for ERP systems:

- **One organization = One legal entity** (company, GSTIN, bank account)
- **One company = Multiple users** (accountant, warehouse manager, sales rep, admin)
- **Multi-tenant isolation** (data never crosses company boundaries)
- **Role-based access** (initialized on Day 1, not added later)

**Target user for sign-up:** Business owner, CFO, or IT manager (primary admin) setting up the company.

**Outcome after 6 steps:** Fully initialized company workspace with 1 admin, ready for team invites and first transactions.

---

# PART 1: ARCHITECTURAL SIGN-UP & ACCESS MODELS

---

## 1.1 Model Comparison: Organization-Centric vs. Open Self-Service

### Model A: Open Self-Service Registration (❌ NOT FOR ERP)

**How it works:**
```
User signs up individually
↓
User's email = unique identity
↓
User can create multiple "teams" or "workspaces"
↓
User can invite colleagues anytime
↓
Example: Zoho Mail, Dropbox, Slack (consumer SaaS)
```

**Why this fails for ERP:**

```
Problem 1: Unclear company boundaries
├─ If user "Raj" from ABC Trading signs up with raj@abctrading.com
├─ And "Priya" from XYZ Retail signs up with priya@xyzvretail.com
├─ System doesn't know:
│   ├─ Are they in the same company? (Name-based, not official)
│   ├─ What's the company GSTIN? (Not captured upfront)
│   ├─ What if both claim to be "ABC Trading"? (Duplication)
│   └─ GL data could be split across two workspaces (data fragmentation)
│
└─ Result: Regulatory & financial reporting breaks

Problem 2: Role assignment too late
├─ User signed up as generic "user"
├─ Roles assigned later (ad-hoc, prone to error)
├─ Permissions not enforced from day 1
├─ Example: Accountant gets sales permission (data exposure)
│
└─ Result: Weak access control, compliance violation

Problem 3: Tax ID not verified upfront
├─ GSTIN entered later (or not at all)
├─ No company validation (GSTIN could be fake)
├─ Can't file GSTR-1 without confirmed GSTIN
│
└─ Result: Tax compliance failures

Problem 4: Company master data fragmented
├─ Company info spread across user profiles
├─ Bank account registered by one user, updated by another
├─ Chart of Accounts might differ per user's view
│
└─ Result: Single source of truth lost
```

---

### Model B: Organization-Centric Admin Setup (✅ REQUIRED FOR ERP)

**How it works:**
```
Primary Admin registers with company info (GSTIN, legal name, location)
↓
System creates isolated tenant for that company
↓
Tenant configured with company's chart of accounts, tax settings, users
↓
Admin invites staff (accountant, warehouse, sales) with pre-defined roles
↓
All staff login to same company workspace (single source of truth)
↓
Example: Zoho Books, Tally Prime, SAP (enterprise ERP)
```

**Why this works for ERP:**

```
Benefit 1: Clear company boundaries
├─ One organization = One legal entity
├─ One GSTIN = One GL, one tax return
├─ No confusion about "which company's data?"
│
└─ Result: Data integrity, regulatory compliance

Benefit 2: Role-based access from day 1
├─ Admin assigns roles at provisioning time
├─ Accountant gets GL, AP, AR permissions (not sales)
├─ Warehouse manager gets inventory permissions (not GL)
├─ Compliance built-in
│
└─ Result: Least-privilege access, auditable from inception

Benefit 3: Tax ID validated upfront
├─ GSTIN captured at registration (Step 3)
├─ Verified against TDS database (optional, for future)
├─ GL posting rules configured to company's GST jurisdiction
│
└─ Result: Tax compliance ready before first invoice

Benefit 4: Company master is single source
├─ Registered at Step 3 (never fragmented)
├─ Chart of Accounts, bank account, tax settings derived from one source
├─ Audit trail on company setup changes
│
└─ Result: Historical integrity, audit-ready
```

---

## 1.2 Role-Based Access Control (RBAC) Initialization

### RBAC Hierarchy

```
Niyanthra
├─ Organization (Company)
│   ├─ User 1: "Raj" (Primary Admin)
│   │   └─ Roles:
│   │       ├─ company_admin (all permissions)
│   │       ├─ financial_close (month-end close)
│   │       └─ user_management (invite staff, assign roles)
│   │
│   ├─ User 2: "Priya" (Accountant)
│   │   └─ Roles:
│   │       ├─ general_ledger (GL posting, reconciliation)
│   │       ├─ ar_management (invoices, collections)
│   │       ├─ ap_management (vendor bills, payment)
│   │       └─ tax_compliance (GSTR filing, TDS tracking)
│   │
│   ├─ User 3: "Kumar" (Warehouse Manager)
│   │   └─ Roles:
│   │       ├─ inventory_management (stock, GRN)
│   │       └─ warehouse_operations (picking, packing)
│   │
│   └─ User 4: "Anjali" (Sales)
│       └─ Roles:
│           ├─ sales_order_creation (SO, invoices)
│           └─ customer_management (CRM)
```

### Permission Matrix (RBAC at API Level)

```
Permission             | Admin | Accountant | Warehouse | Sales | View-Only
─────────────────────────────────────────────────────────────────────────────
View Dashboard         | ✓     | ✓          | ✓         | ✓     | ✓
Create Sales Order     | ✓     | ✗          | ✗         | ✓     | ✗
Create Purchase Order  | ✓     | ✓          | ✗         | ✗     | ✗
Post GL Entry          | ✓     | ✓          | ✗         | ✗     | ✗
View AR Aging          | ✓     | ✓          | ✗         | ✗     | ✗
Edit Chart of Accounts | ✓     | ✗          | ✗         | ✗     | ✗
Invite Users           | ✓     | ✗          | ✗         | ✗     | ✗
Close Month            | ✓     | ✓          | ✗         | ✗     | ✗
File GSTR-1            | ✓     | ✓          | ✗         | ✗     | ✗
Manage Inventory       | ✓     | ✗          | ✓         | ✗     | ✗
View Financial Reports | ✓     | ✓          | ✓         | ✓     | ✓
Edit Company Settings  | ✓     | ✗          | ✗         | ✗     | ✗
```

### RBAC Initialization During Onboarding

```
Step 1-2 (Authentication): User identified via email + OTP
├─ No roles yet (user unverified)
├─ Temporary session created for onboarding flow

Step 3 (Company Profile): Company registered
├─ Database: companies table created (company_id = 1)
├─ No roles yet (company not activated)

Step 4 (Module Selection): Feature set chosen (Sales + Inventory + GL)
├─ No role changes (feature preference only)

Step 5 (User Provisioning): First user registered as Admin
├─ Database: users table created (user_id = 1, role_id = ADMIN)
├─ Database: user_roles table created:
│   ├─ user_id = 1
│   ├─ company_id = 1
│   ├─ role = 'COMPANY_ADMIN'
│   ├─ permissions = ['all']
│   └─ created_at = NOW()
│
├─ Database: role_permissions table (lookup):
│   ├─ role = 'COMPANY_ADMIN'
│   ├─ permission = 'users:invite' ✓
│   ├─ permission = 'gl:post' ✓
│   ├─ permission = 'company:edit' ✓
│   └─ ... (50+ permissions)
│
└─ Session updated: Add claims to JWT
    ├─ sub: user_id
    ├─ org: company_id
    ├─ roles: ['COMPANY_ADMIN']
    └─ permissions: ['all']

Step 6 (Workspace Init): Dashboard opens
├─ Homepage checks permissions
│   ├─ Can create SO? Yes (COMPANY_ADMIN)
│   ├─ Can create GL entry? Yes
│   └─ Show all modules
│
└─ Admin ready to invite staff (future, not in onboarding)

Later (Staff Invited): Secondary users provisioned
├─ Admin invites accountant (email invite link)
├─ Accountant signs up → joins existing company
├─ Database: users table (user_id = 2)
├─ Database: user_roles table:
│   ├─ user_id = 2
│   ├─ company_id = 1 (same as admin)
│   ├─ role = 'ACCOUNTANT'
│   └─ permissions = ['gl:post', 'ar:view', 'ap:view', 'tax:file']
│
├─ Accountant's JWT gets limited permissions
└─ Accountant can't delete company, can't invite users (proper RBAC)
```

---

# PART 2: STEP-BY-STEP UI/UX WIREFRAME FLOW

---

## Step 1: Landing & Fast Identity Capture

### User Journey
```
User lands on https://app.niyanthra.io/signup
         ↓
    "Ready to digitize your business?"
    (Video, 2 minute overview)
         ↓
    Click "Start Free Trial"
         ↓
    Presented with Step 1 form
```

### UI Layout - Step 1 (Desktop, 1200px width)

```
┌─────────────────────────────────────────────────────┐
│  Niyanthra Logo          [EN ▼] [Help] [Login →]    │  ← Header
├─────────────────────────────────────────────────────┤
│                                                       │
│   Step 1 of 6: Your Identity                         │
│   ─────────────────────────────────────────────      │
│   Set up your Niyanthra account in 5 minutes         │
│                                                       │
│   ┌──────────────────────────────────┐               │
│   │ Email Address *                   │               │ ← Form Section
│   │ ┌────────────────────────────────┤               │
│   │ │ you@businessname.com            │ ← Red border │
│   │ │ (auto-filled if from email link) │ if invalid  │
│   │ └────────────────────────────────┤               │
│   │ This is your unique login ID      │               │
│   │                                   │               │
│   │ Phone Number (Optional)           │               │
│   │ ┌────────────────────────────────┤               │
│   │ │ +91 [   ] [         ]           │ ← Format     │
│   │ │         (10 digits)              │   validation │
│   │ └────────────────────────────────┤               │
│   │ For OTP verification               │               │
│   │                                   │               │
│   │ Password *                        │               │
│   │ ┌────────────────────────────────┤               │
│   │ │ ••••••••                         │               │
│   │ └────────────────────────────────┤               │
│   │ Min 8 chars, 1 uppercase, 1 number │              │
│   │                                   │               │
│   │ Confirm Password *                │               │
│   │ ┌────────────────────────────────┤               │
│   │ │ ••••••••                         │               │
│   │ └────────────────────────────────┤               │
│   │ Match password above               │               │
│   │                                   │               │
│   │ ☐ I agree to Terms & Privacy      │ ← Checkbox   │
│   │   (Links to legal pages)          │               │
│   │                                   │               │
│   │           [Continue →]            │ ← CTA Button │
│   │                                   │               │
│   │ Already have an account? Log in   │ ← Alt Action │
│   │                                   │               │
│   └──────────────────────────────────┘               │
│                                                       │
│   Progress Indicator:                                │
│   ●───○───○───○───○───○                             │
│    1   2   3   4   5   6                             │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Form Inputs & Validation

```
Field: Email Address
├─ Type: email
├─ Required: Yes
├─ Validation:
│   ├─ Format: RFC 5322 regex
│   ├─ Domain: Not free email (no @gmail.com, @yahoo.com)
│   ├─ DB check: Unique (not already registered)
│   └─ Error message: "Email already registered. Try logging in."
│
├─ On blur: 
│   ├─ API call: POST /auth/check-email
│   ├─ Response: {"exists": false, "available": true}
│   └─ Show spinner (UX: "Checking...")
│
└─ Prefill: If user came from email invite link (admin inviting accountant)
    ├─ Link: signup.html?email=accountant@xyz.com
    └─ Auto-fill: accountant@xyz.com (pre-verified)

Field: Phone Number
├─ Type: tel
├─ Required: No (but strongly encouraged for OTP)
├─ Validation:
│   ├─ Format: +91 [10 digits]
│   ├─ Country code: +91 only (India focus, Phase 1)
│   └─ Logical: Not test numbers (e.g., 9999999999)
│
└─ Prefill: Detect from browser (geolocation, if permitted)

Field: Password
├─ Type: password
├─ Required: Yes
├─ Validation (Real-time feedback):
│   ├─ Length: >= 8 characters (show: "✓ 8+ characters" or "✗ Too short")
│   ├─ Uppercase: >= 1 (show: "✓ Has uppercase" or "✗ Add uppercase")
│   ├─ Number: >= 1 (show: "✓ Has number" or "✗ Add number")
│   ├─ Special char: Optional but encouraged (show: "✓ Strong" or "○ Add special char")
│   ├─ Entropy check: Not in common passwords list
│   └─ Breached check: Cross-check with haveibeenpwned.com API (optional)
│
├─ Show/hide toggle: Eye icon (click to reveal/mask)
└─ Strength meter: Visual bar (Weak → Fair → Good → Strong)

Field: Confirm Password
├─ Type: password
├─ Required: Yes
├─ Validation (Real-time):
│   ├─ Match: password === confirm_password
│   ├─ Feedback: "✓ Passwords match" or "✗ Passwords don't match"
│   └─ Disable continue button if mismatch
│
└─ Keyboard shortcut: Tab to Confirm → Auto-validate

Field: Terms & Privacy
├─ Type: Checkbox (required)
├─ Text: "I agree to Niyanthra's [Terms of Service] and [Privacy Policy]"
├─ Links: Open in new tab (don't lose form data)
└─ Validation: Unchecked → "Continue" button disabled (grayed out)
```

### Backend Trigger on Submit (Step 1)

```
User clicks "Continue →"
    ↓
Frontend Validation:
├─ Email valid & unique? ✓
├─ Passwords match? ✓
├─ Terms accepted? ✓
    ↓
API Call:
POST /auth/register-step-1
{
  "email": "raj@abctrading.com",
  "phone": "+919876543210",
  "password_hash": "bcrypt(password)",
  "source": "organic" / "invite_link" (if admin invited)
}

Backend Processing:
├─ Rate limiting: Max 5 signup attempts per IP/hour
├─ Password hashing: bcrypt (salt rounds = 12)
├─ Database insert:
│   ├─ users table (unverified):
│   │   ├─ user_id = auto-generated UUID
│   │   ├─ email = "raj@abctrading.com"
│   │   ├─ phone = "+919876543210"
│   │   ├─ password_hash = bcrypt_result
│   │   ├─ status = 'UNVERIFIED' (not yet OTP'd)
│   │   ├─ created_at = NOW()
│   │   └─ onboarding_stage = 1
│   │
│   └─ onboarding_sessions table (temp):
│       ├─ session_id = UUID
│       ├─ user_id = (FK)
│       ├─ stage = 1
│       ├─ data = {email, phone, password_hash}
│       ├─ expires_at = NOW() + 1 hour (session TTL)
│       └─ created_at = NOW()
│
├─ Create OTP (6 digits, valid 5 minutes):
│   └─ otp_store table:
│       ├─ user_id = (FK)
│       ├─ otp_code = "123456" (hashed)
│       ├─ method = "email" / "phone" (preferred: email)
│       ├─ expires_at = NOW() + 5 minutes
│       ├─ attempts = 0
│       └─ created_at = NOW()
│
└─ Send OTP via email:
    ├─ Subject: "Niyanthra Verification Code: 123456"
    ├─ Body: 
    │   "Enter this code to complete sign-up: 123456
    │    Code expires in 5 minutes.
    │    If this wasn't you, ignore this email."
    │
    └─ Email service: AWS SES / SendGrid (with retry logic)

Response (201 Created):
{
  "status": "success",
  "message": "OTP sent to raj@abctrading.com",
  "session_id": "sess_abc123",
  "otp_delivery_method": "email",
  "expires_in_seconds": 300,
  "next_step": 2
}

Frontend Actions:
├─ Store session_id in memory (not localStorage, for security)
├─ Show success toast: "OTP sent to your email"
├─ Redirect to Step 2 (OTP verification)
└─ Start 5-minute countdown timer (auto-expiry warning)
```

---

## Step 2: OTP Verification

### UI Layout - Step 2

```
┌─────────────────────────────────────────────────────┐
│  Niyanthra Logo          [EN ▼] [Help] [Login →]    │
├─────────────────────────────────────────────────────┤
│                                                       │
│   Step 2 of 6: Verify Your Email                    │
│   ─────────────────────────────────────────────────  │
│   We sent a 6-digit code to raj@abctrading.com       │
│   (Verify within 5:00 minutes)                      │
│                                                       │
│   ┌──────────────────────────────────┐               │
│   │ Enter Verification Code *        │               │
│   │ ┌────────────────────────────────┤               │
│   │ │ [1][2][3][4][5][6]             │ ← OTP boxes  │
│   │ │  (Focus auto-moves to next box) │  (Monospace)│
│   │ └────────────────────────────────┤               │
│   │                                   │               │
│   │ Didn't receive code?              │ ← Support    │
│   │ [Resend Code] (Enabled after 30s) │              │
│   │                                   │               │
│   │ ────────────────────────────────  │               │
│   │ Or verify via SMS instead         │ ← Alt method │
│   │ [Send Code via SMS]               │              │
│   │                                   │               │
│   │           [Continue →]            │ ← CTA        │
│   │                                   │               │
│   └──────────────────────────────────┘               │
│                                                       │
│   Progress Indicator:                                │
│   ●───●───○───○───○───○                             │
│    1   2   3   4   5   6                             │
│                                                       │
│   [← Back to Step 1]                 │ ← Back link   │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### OTP Input & Verification

```
Input: OTP (6 digits)
├─ UI: 6 separate input boxes (HTML5 <input type="number" />)
├─ Behavior:
│   ├─ Type digit in box 1 → auto-focus box 2
│   ├─ Backspace in box 2 → focus box 1, clear box 2
│   ├─ Paste: "123456" → auto-fill all 6 boxes
│   ├─ Copy protection: Disable copy/paste via JS (optional, UX tradeoff)
│   └─ Numeric only: Validation (no letters)
│
├─ Real-time validation:
│   ├─ All 6 boxes filled? → Enable "Continue" button
│   ├─ Auto-submit after 6th digit (optional UX, skip manual click)
│   └─ Invalid format? → Red border, shake animation
│
└─ Timeout: If 5 minutes elapsed
    ├─ Disable input
    ├─ Show: "Code expired. Resend a new code."
    └─ Force user to resend or go back

Resend Code Button:
├─ Initially disabled (grayed out)
├─ Re-enabled after 30 seconds (countdown timer: "Resend in 0:30")
├─ Rate limit: Max 3 resend attempts
└─ On click:
    ├─ API: POST /auth/resend-otp (session_id)
    ├─ New OTP generated, sent via email
    ├─ Timer resets (5 minutes, resend cooldown 30s)
    └─ Toast: "New code sent to your email"

SMS Alternative:
├─ Available if phone was provided in Step 1
├─ Button: "Send Code via SMS" (below email OTP entry)
├─ On click:
│   ├─ API: POST /auth/resend-otp-sms (session_id)
│   ├─ OTP sent to phone number
│   ├─ Same OTP code (reusable across email & SMS)
│   └─ Toast: "Code sent via SMS to +91 9876543210"
│
└─ Security: Phone number partially masked (show last 4 digits only)
```

### Backend Trigger on Submit (Step 2)

```
User enters 6-digit OTP, clicks "Continue"
    ↓
Frontend Validation:
├─ All 6 boxes filled? ✓
├─ Only digits? ✓
├─ Format: 6 digits ✓
    ↓
API Call:
POST /auth/verify-otp
{
  "session_id": "sess_abc123",
  "otp_code": "123456",
  "method": "email"  // or "sms"
}

Backend Processing:
├─ Rate limiting: Max 5 OTP attempts per session
├─ Session lookup:
│   ├─ Fetch session_id from onboarding_sessions
│   ├─ Check: Not expired (expires_at > NOW()) ✓
│   └─ Check: Still at stage 1 or 2 (not completed earlier steps)
│
├─ OTP verification:
│   ├─ Fetch otp_code from otp_store
│   ├─ Check: Not expired (expires_at > NOW()) ✓
│   ├─ Check: Attempts < 5 (not too many failed attempts)
│   ├─ Compare: bcrypt_compare(otp_input, otp_hash) ✓
│   └─ If fails: Increment attempts, return error
│
├─ Mark user as verified:
│   ├─ Update users table:
│   │   ├─ status = 'VERIFIED'
│   │   ├─ email_verified_at = NOW()
│   │   └─ onboarding_stage = 2
│   │
│   └─ Delete otp_store (cleanup)
│
├─ Extend session:
│   ├─ Generate new session token (JWT or signed cookie)
│   ├─ Add claims: {user_id, email, verified: true}
│   └─ TTL: 24 hours (allows Step 1-6 completion within a day)
│
└─ Log audit:
    ├─ audit_logs table:
    │   ├─ user_id = (FK)
    │   ├─ action = "otp_verified"
    │   ├─ ip_address = 203.0.113.45
    │   ├─ user_agent = "Mozilla/5.0..."
    │   └─ timestamp = NOW()

Response (200 OK):
{
  "status": "success",
  "message": "Email verified successfully",
  "session_id": "sess_def456",
  "user_id": "user_xyz789",
  "verified": true,
  "next_step": 3,
  "authorization": "Bearer eyJhbGciOi..."  ← JWT (store in memory)
}

Frontend Actions:
├─ Store JWT in memory (not localStorage for XSS protection)
├─ Show success toast: "Email verified ✓"
├─ Redirect to Step 3 (Company Profile)
└─ Clear OTP form
```

---

## Step 3: Company Profile & Regional Compliance Setup

### UI Layout - Step 3

```
┌─────────────────────────────────────────────────────┐
│  Niyanthra Logo          [EN ▼] [Help] [Login →]    │
├─────────────────────────────────────────────────────┤
│                                                       │
│   Step 3 of 6: Your Company Profile                 │
│   ─────────────────────────────────────────────────  │
│   Set up your business details for compliance        │
│   & financial reporting (India-specific)            │
│                                                       │
│   ┌──────────────────────────────────────────┐       │
│   │ Company Legal Name *                      │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ ABC Trading Pvt Ltd                  │ │ ← Max │
│   │ └──────────────────────────────────────┤ │   150 │
│   │ As registered with Ministry of Incorporation  │ chars
│   │                                         │       │
│   │ Business Type *                         │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ [Proprietorship ▼]                   │ │ ← Select
│   │ └──────────────────────────────────────┤ │       │
│   │ Options: Proprietorship, Partnership,  │       │
│   │ Pvt Ltd, Public Ltd, LLP, Sole Trader │       │
│   │                                         │       │
│   │ Business Category *                    │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ ☐ Trading/Wholesale ☑              │ │ ← Multi
│   │ │ ☐ Retail                           │ │   select
│   │ │ ☐ Distribution                     │ │   (for Phase 6)
│   │ │ ☐ Manufacturing                    │ │       │
│   │ │ ☐ Services                         │ │       │
│   │ └──────────────────────────────────────┤ │       │
│   │                                         │       │
│   │ GSTIN (Goods & Services Tax ID) *     │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ 27AABCT1234H1Z5                     │ │ ← Format
│   │ └──────────────────────────────────────┤ │   SSGG
│   │ Format: 2 digits state, 10 digit PAN,  │       │ AAAXN
│   │ 1 digit entity, 1 check digit         │       │ CNNNN
│   │ Status: ✓ Valid GSTIN                 │       │ C
│   │                                         │       │
│   │ Company Pan Card Number *              │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ AABCT1234H                          │ │ ← Format
│   │ └──────────────────────────────────────┤ │   validation
│   │ This is derived from GSTIN but can    │       │
│   │ be verified separately                 │       │
│   │                                         │       │
│   │ Registration State *                   │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ [Kerala ▼]                          │ │ ← From
│   │ └──────────────────────────────────────┤ │   GSTIN
│   │ Extracted from GSTIN (27 = Karnataka) │       │ auto-fill
│   │ But shows user's actual state         │       │
│   │                                         │       │
│   │ Primary Business Address *             │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ XYZ Road, Kochi, 682001            │ │ ← Max
│   │ └──────────────────────────────────────┤ │   300
│   │                                         │       │ chars
│   │ City/Town *                            │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ [Kochi ▼]                           │ │ ← Select
│   │ └──────────────────────────────────────┤ │   from
│   │ Populated based on address/PIN        │       │ list
│   │                                         │       │
│   │ Pincode *                              │       │
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ 682001                              │ │ ← Numeric
│   │ └──────────────────────────────────────┤ │   only,
│   │                                         │       │ validates
│   │ Financial Year End (Month) *          │       │ ZIP codes
│   │ ┌──────────────────────────────────────┤ │       │
│   │ │ [March (Default) ▼]                 │ │ ← For GL
│   │ └──────────────────────────────────────┤ │   close
│   │ Default: March (Indian FY = Apr-Mar)  │       │
│   │ Customize if different FY             │       │
│   │                                         │       │
│   │ ☐ This company is GST registered     │ ← Checkbox
│   │   (Enables GST tracking, GSTR filing) │       │
│   │                                         │       │
│   │ ☐ This company is TDS filer          │ ← Checkbox
│   │   (Enables TDS tracking, Form 26Q)   │       │
│   │                                         │       │
│   │           [Continue →]                 │ ← CTA  │
│   │                                         │       │
│   │ [← Back to Step 2]                    │ ← Back │
│   │                                         │       │
│   └──────────────────────────────────────────┘       │
│                                                       │
│   Progress Indicator:                                │
│   ●───●───●───○───○───○                             │
│    1   2   3   4   5   6                             │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Form Inputs & Validation

```
Field: Company Legal Name
├─ Type: text
├─ Required: Yes
├─ Max length: 150 characters
├─ Validation:
│   ├─ Not empty ✓
│   ├─ No special characters (/, \, @, etc.)
│   ├─ Alphanumeric, spaces, hyphens only
│   └─ DB check: Unique per state (to avoid duplicates)
│
├─ Error: "Company name must contain only letters, numbers, spaces, hyphens"
└─ Auto-capitalize: "abc trading" → "Abc Trading"

Field: Business Type
├─ Type: select (dropdown)
├─ Required: Yes
├─ Options:
│   ├─ Proprietorship
│   ├─ Partnership
│   ├─ Private Limited (Pvt Ltd)
│   ├─ Public Limited (Ltd)
│   ├─ Limited Liability Partnership (LLP)
│   ├─ Sole Trader
│   └─ Other
│
├─ Impact on later steps:
│   ├─ Proprietorship: Single owner, simplified GL
│   ├─ Partnership: Multiple partners, shared GL
│   ├─ Pvt Ltd: Formal cap table, Directors
│   └─ Each requires different Chart of Accounts init
│
└─ Default: None (must select)

Field: Business Category
├─ Type: checkbox (multi-select)
├─ Required: Yes (at least one)
├─ Options (for Phase 1, only check those available):
│   ├─ ☑ Trading/Wholesale (included in MVP)
│   ├─ ☑ Retail (included in MVP with POS)
│   ├─ ☑ Distribution (included in MVP)
│   ├─ ☐ Manufacturing (Phase 3, BOM not yet)
│   ├─ ☐ Services (Phase 2)
│   └─ ☐ E-Commerce (Phase 3)
│
├─ Impact:
│   ├─ Determines initial module set shown in Step 4
│   ├─ Affects sample Chart of Accounts (trading GL ≠ retail GL)
│   └─ Later: Feature flagging (manufacturing features hidden for trading)
│
└─ Multiple selections allowed (e.g., Trading + Retail)

Field: GSTIN
├─ Type: text (uppercase, 15 chars)
├─ Required: Yes
├─ Validation (Real-time):
│   ├─ Format: [0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}[Z]{1}
│   │   └─ Example: 27AABCT1234H1Z5
│   ├─ Check digit: Verify using GST checksum algorithm (optional, Phase 2)
│   ├─ DB check: Unique (one company per GSTIN)
│   └─ Business name match: Warn if GSTIN name ≠ entered name (not blocking)
│
├─ On blur: 
│   ├─ API: POST /auth/validate-gstin {gstin}
│   ├─ Check against NSDL database (mock for now, real API later)
│   └─ Return: {valid: true, registered_name: "ABC TRADING", state: 27}
│
├─ Auto-fill from GSTIN:
│   ├─ Extract state code (first 2 digits) → populate State field
│   ├─ Extract PAN (digits 3-12) → populate PAN field
│   └─ Note: User can edit if necessary
│
├─ Error: "Invalid GSTIN format. Example: 27AABCT1234H1Z5"
└─ Field dependency: If GSTIN invalid, disable "Continue" button

Field: PAN
├─ Type: text (uppercase, 10 chars)
├─ Required: Yes (but often auto-filled from GSTIN)
├─ Validation:
│   ├─ Format: [A-Z]{5}[0-9]{4}[A-Z]{1}
│   │   └─ Example: AABCT1234H
│   ├─ Check digit: Last digit must be letter (A-Z)
│   └─ DB check: Unique per company
│
├─ Auto-fill from GSTIN (digits 3-12 of GSTIN = PAN):
│   ├─ GSTIN: 27AABCT1234H1Z5
│   ├─ PAN extracted: AABCT1234H
│   └─ Pre-populate field, allow edit
│
└─ Error: "Invalid PAN format. Example: AABCT1234H"

Field: Registration State
├─ Type: select (auto-filled from GSTIN)
├─ Required: Yes
├─ Validation:
│   ├─ Derived from GSTIN state code (first 2 digits)
│   ├─ Example: GSTIN 27... → State: Karnataka
│   └─ But shows ALL Indian states (user can correct if GSTIN office elsewhere)
│
├─ States (India, 28 states + 8 UTs):
│   ├─ Andhra Pradesh (36)
│   ├─ Kerala (32)
│   ├─ ... (28 others)
│   └─ (Sorted alphabetically)
│
└─ Dependency: Determines GST jurisdiction rules for this company

Field: Business Address
├─ Type: text (long)
├─ Required: Yes
├─ Max length: 300 characters
├─ Validation:
│   ├─ Not empty
│   ├─ Valid address characters (letters, numbers, spaces, commas)
│   └─ Include street, locality, landmarks
│
├─ Placeholder: "123 Main Road, Industrial Area, Kochi"
└─ Note: This address used for GST return, bank reconciliation, audit correspondence

Field: City/Town
├─ Type: select (dropdown)
├─ Required: Yes
├─ Dynamic population:
│   ├─ On State selection: Fetch cities in that state
│   ├─ Filter by type-ahead (user types "Koch..." → "Kochi" appears)
│   └─ Default: Top 10 cities in state (Bangalore, Mumbai, etc.)
│
├─ Data source: cities_master table (id, name, state, district, pincode_range)
└─ Example: Kochi, Thiruvananthapuram, Kozhikode (Kerala top cities)

Field: Pincode
├─ Type: text (numeric, 6 digits)
├─ Required: Yes
├─ Validation:
│   ├─ Format: [0-9]{6}
│   ├─ Valid Indian pincode (not test numbers)
│   └─ Optional: Verify against pincode_master (offline JSON for speed)
│
├─ On blur:
│   ├─ If invalid format: "Pincode must be 6 digits"
│   ├─ If not found in master: "Warning: Pincode not recognized. Verify?"
│   └─ If different city for pincode: "Pincode is in [actual city]. Update City?"
│
└─ Geo-IP prefill: On page load, detect browser location, suggest city (optional UX)

Field: Financial Year End (Month)
├─ Type: select (dropdown)
├─ Required: Yes
├─ Default: March (Indian FY = April to March)
├─ Options:
│   ├─ March (Default, 04-2026 to 03-2027)
│   ├─ December (01-2026 to 12-2026)
│   ├─ June (07-2025 to 06-2026)
│   └─ ... (other custom months)
│
├─ Impact:
│   ├─ Determines GL close period (monthly, quarterly, annual)
│   ├─ GSTR filing due date (20th next month)
│   ├─ Financial statement generation cycle
│   └─ Example: March FY end → GL close on March 31, file GSTR by April 20
│
└─ Immutable after company creation (to prevent GL year mismatch)

Field: GST Registration Status
├─ Type: checkbox
├─ Required: No (but for MVP, assume all are registered)
├─ Label: "This company is GST registered"
├─ Impact if checked:
│   ├─ Enable GST tracking in GL (Tax accounts created)
│   ├─ Enable GSTR-1 filing setup (Step 4)
│   ├─ Enable e-invoice generation (NSDL API setup)
│   └─ Enable Input Tax Credit (ITC tracking for purchases)
│
├─ Impact if unchecked:
│   ├─ Disable all GST features (for Phase 2, unregistered suppliers)
│   ├─ Simplified tax tracking (flat-rate GST only)
│   └─ No e-invoice (not required for unregistered)
│
└─ For MVP: Checked by default (assume registered)

Field: TDS Filer Status
├─ Type: checkbox
├─ Required: No
├─ Label: "This company files TDS (Form 26Q)"
├─ Impact if checked:
│   ├─ Enable TDS tracking (1% on goods, 2% on services)
│   ├─ Setup Form 26Q filing (quarterly)
│   ├─ Track TDS deposits with tax authority
│   └─ Generate TDS certificates for vendors
│
├─ Impact if unchecked:
│   ├─ No TDS deductions from vendor payments
│   ├─ Simplified AP module
│   └─ Useful for very small businesses (annual turnover < ₹10 lakhs)
│
└─ For MVP: Checked (assume company meets TDS threshold)
```

### Backend Trigger on Submit (Step 3)

```
User completes Company Profile, clicks "Continue"
    ↓
Frontend Validation:
├─ Company name not empty ✓
├─ Business type selected ✓
├─ Category selected ✓
├─ GSTIN format valid ✓
├─ PAN format valid ✓
├─ Address not empty ✓
├─ City selected ✓
├─ Pincode 6 digits ✓
└─ FY end month selected ✓
    ↓
API Call:
POST /auth/register-step-3
{
  "session_id": "sess_def456",
  "company_name": "ABC Trading Pvt Ltd",
  "business_type": "PRIVATE_LIMITED",
  "category": "TRADING",
  "gstin": "27AABCT1234H1Z5",
  "pan": "AABCT1234H",
  "state": "KERALA",
  "address": "XYZ Road, Kochi",
  "city": "KOCHI",
  "pincode": "682001",
  "fy_end_month": 3,  // March = month 3
  "gst_registered": true,
  "tds_filer": true
}

Backend Processing (Atomic Transaction):
├─ Rate limiting: Max 10 requests per session/minute
├─ Validate GSTIN:
│   ├─ Format check ✓
│   ├─ Check digit verification (optional, Phase 2)
│   ├─ DB check: GSTIN unique ✓
│   └─ If fails: 400 Error, "GSTIN already registered"
│
├─ Create company (CRITICAL — Tenant Initialization):
│   ├─ companies table:
│   │   ├─ company_id = auto-generated UUID
│   │   ├─ user_id = (FK) owner
│   │   ├─ company_name = "ABC Trading Pvt Ltd"
│   │   ├─ gstin = "27AABCT1234H1Z5" (UNIQUE)
│   │   ├─ pan = "AABCT1234H"
│   │   ├─ business_type = "PRIVATE_LIMITED"
│   │   ├─ category = "TRADING"
│   │   ├─ state = "KERALA"
│   │   ├─ address = "XYZ Road, Kochi"
│   │   ├─ city = "KOCHI"
│   │   ├─ pincode = "682001"
│   │   ├─ fy_end_month = 3
│   │   ├─ status = 'ACTIVE'
│   │   ├─ gst_registered = true
│   │   ├─ tds_filer = true
│   │   ├─ created_at = NOW()
│   │   └─ created_by_user_id = (same as user_id)
│   │
│   └─ [CRITICAL] This company_id is the TENANT_ID for all future data
│
├─ Create Chart of Accounts (based on business type + category):
│   ├─ chart_of_accounts table:
│   │   ├─ company_id = (FK, just created)
│   │   ├─ account_code = "1010" (asset)
│   │   ├─ account_name = "Trade Receivable"
│   │   ├─ account_group = "ASSET"
│   │   ├─ balance = 0.00
│   │   ├─ created_at = NOW()
│   │   └─ ... (50-100 default accounts inserted)
│   │
│   └─ Default accounts (sample for trading company):
│       ├─ 1010: Trade Receivable
│       ├─ 1020: Cash at Bank
│       ├─ 1040: Inventory
│       ├─ 2010: Accounts Payable
│       ├─ 2020: SGST Payable (if GST registered)
│       ├─ 4010: Sales Revenue (by HSN/rate)
│       ├─ 5010: Cost of Goods Sold
│       ├─ 5020: Salaries & Wages
│       └─ ... (standard GL structure)
│
├─ Create Tax Rules (company-specific):
│   ├─ tax_rules table:
│   │   ├─ company_id = (FK)
│   │   ├─ tax_type = "GST"
│   │   ├─ state = "KERALA"
│   │   ├─ intra_state_rules = "SGST + CGST" (if same state)
│   │   ├─ inter_state_rules = "IGST" (if different state)
│   │   ├─ fy_end_month = 3
│   │   ├─ fy_start_month = 4
│   │   ├─ gstr1_due_date = "20th next month"
│   │   ├─ gstr3b_due_date = "20th next month"
│   │   └─ created_at = NOW()
│   │
│   └─ Tax rates by HSN (lookup table):
│       ├─ hsn_tax_rates table (global, not per-company)
│       │   ├─ hsn_code = "3208" (Paint)
│       │   ├─ tax_rate = 18
│       │   └─ cess = 0
│       │
│       └─ Default rates for India:
│           ├─ 0% (Exempt goods)
│           ├─ 5% (Essential items)
│           ├─ 12% (Common goods)
│           ├─ 18% (Standard rate)
│           └─ 28% (Luxury goods)
│
├─ Create Banks & Bank Accounts (optional, can be added later):
│   └─ bank_accounts table:
│       ├─ company_id = (FK)
│       ├─ bank_name = NULL (to be added in Step 6)
│       ├─ account_number = NULL
│       ├─ gl_account_id = "1020" (Cash at Bank)
│       └─ status = 'PENDING_SETUP'
│
├─ Create Warehouse (default):
│   ├─ warehouses table:
│   │   ├─ company_id = (FK)
│   │   ├─ warehouse_id = "WH-001" (auto-generated)
│   │   ├─ name = "Main Warehouse"
│   │   ├─ address = (same as company address)
│   │   ├─ status = 'ACTIVE'
│   │   └─ created_at = NOW()
│   │
│   └─ stock_summary table (empty initially):
│       ├─ company_id = (FK)
│       ├─ warehouse_id = (FK)
│       ├─ product_id = NULL (products added later)
│       ├─ available_qty = 0
│       ├─ reserved_qty = 0
│       └─ inventory_value = 0.00
│
├─ Create Default Settings:
│   ├─ company_settings table:
│   │   ├─ company_id = (FK)
│   │   ├─ currency = "INR"
│   │   ├─ locale = "en_IN"
│   │   ├─ timezone = "Asia/Kolkata"
│   │   ├─ date_format = "DD-MM-YYYY"
│   │   ├─ decimal_places = 2
│   │   ├─ fiscal_year_end = 3
│   │   ├─ invoice_prefix = "INV-" (customizable)
│   │   ├─ po_prefix = "PO-"
│   │   ├─ credit_days_default = 30
│   │   └─ created_at = NOW()
│
├─ Update onboarding session:
│   ├─ onboarding_sessions table:
│   │   ├─ company_id = (FK, just created)
│   │   ├─ stage = 3
│   │   ├─ data = {...all step 3 data...}
│   │   └─ updated_at = NOW()
│
├─ Log audit:
│   ├─ audit_logs table:
│   │   ├─ user_id = (FK)
│   │   ├─ company_id = (FK)
│   │   ├─ action = "company_created"
│   │   ├─ details = "ABC Trading Pvt Ltd, GSTIN 27AABCT1234H1Z5"
│   │   ├─ timestamp = NOW()
│   │   └─ ip_address = 203.0.113.45
│
└─ [CRITICAL] Database triggers:
    └─ On company creation, also:
        ├─ Create company_roles table (role templates for this company)
        ├─ Create user_roles table (initialized with admin role for owner)
        ├─ Create permissions_cache (denormalized for fast API checks)
        └─ Create rate_limit_buckets (for API throttling per company)

Response (201 Created):
{
  "status": "success",
  "message": "Company created successfully",
  "company_id": "comp_abc123",
  "company_name": "ABC Trading Pvt Ltd",
  "gstin": "27AABCT1234H1Z5",
  "coa_created": true,
  "tax_rules_initialized": true,
  "default_warehouse_created": true,
  "session_id": "sess_ghi789",
  "next_step": 4
}

Frontend Actions:
├─ Store company_id in session (for all future requests)
├─ Show success toast: "Company profile saved ✓"
├─ Redirect to Step 4 (Module Selection)
└─ Update session JWT: Add company_id claim
```

---

## Step 4: Module Selection

### UI Layout - Step 4

```
┌─────────────────────────────────────────────────────┐
│  Niyanthra Logo          [EN ▼] [Help] [Login →]    │
├─────────────────────────────────────────────────────┤
│                                                       │
│   Step 4 of 6: Choose Your Modules                  │
│   ─────────────────────────────────────────────────  │
│   Customize your workspace for your business needs  │
│   (You can enable/disable later, anytime)           │
│                                                       │
│   ┌──────────────────────────────────────────┐       │
│   │ CORE MODULES (Required for MVP)          │       │
│   │                                          │       │
│   │ ☑ Sales & Distribution                 │       │ ← Pre-selected
│   │   • Create quotations & sales orders    │       │
│   │   • Invoice & delivery tracking         │       │
│   │   • e-Invoice generation & QR           │       │
│   │   Status: Ready                         │       │
│   │                                          │       │
│   │ ☑ Purchase Management                   │       │ ← Pre-selected
│   │   • Purchase orders & GRN                │       │
│   │   • Vendor bill matching                 │       │
│   │   • TDS tracking                        │       │
│   │   Status: Ready                         │       │
│   │                                          │       │
│   │ ☑ Inventory & Stock Control             │       │ ← Pre-selected
│   │   • Multi-warehouse management          │       │
│   │   • Stock transfers & adjustments       │       │
│   │   • FIFO costing & batch tracking       │       │
│   │   Status: Ready                         │       │
│   │                                          │       │
│   │ ☑ General Ledger & Accounting           │       │ ← Pre-selected
│   │   • Real-time GL posting                │       │
│   │   • Bank reconciliation                 │       │
│   │   • AR/AP aging & subledgers            │       │
│   │   Status: Ready                         │       │
│   │                                          │       │
│   └──────────────────────────────────────────┘       │
│                                                       │
│   ┌──────────────────────────────────────────┐       │
│   │ SPECIALIZED MODULES (Optional, Phase 1)  │       │
│   │                                          │       │
│   │ ☑ Point of Sale (POS)                   │       │ ← Pre-selected
│   │   • Counter billing with barcode        │       │
│   │   • Real-time inventory sync            │       │
│   │   • Shift reconciliation                │       │
│   │   Status: Available                     │       │
│   │   Category match: Your business type    │       │
│   │                                          │       │
│   │ ☐ Customer Relationship Management      │       │ ← Optional
│   │   • Customer profiles & order history  │       │
│   │   • Lead tracking (coming soon)         │       │
│   │   Status: Beta (Phase 2)                │       │
│   │                                          │       │
│   │ ☐ Manufacturing & BOM                   │       │ ← Disabled
│   │   • Bill of Materials                   │       │
│   │   • Work-in-Progress (WIP) tracking     │       │
│   │   Status: Coming in Phase 3             │       │
│   │   Reason: Not applicable for trading    │       │
│   │                                          │       │
│   │ ☐ Payroll & HR                          │       │ ← Disabled
│   │   • Salary calculation & payslips       │       │
│   │   • Attendance tracking                 │       │
│   │   Status: Coming in Phase 2             │       │
│   │                                          │       │
│   └──────────────────────────────────────────┘       │
│                                                       │
│   ┌──────────────────────────────────────────┐       │
│   │ YOUR MODULE SUMMARY                     │       │
│   │                                          │       │
│   │ Enabled Modules: 5                      │       │
│   │ • Sales, Purchase, Inventory, GL, POS  │       │
│   │                                          │       │
│   │ Dashboard Preview:                      │       │
│   │ └─ Sales Dashboard (KPIs)              │       │
│   │ └─ Inventory Dashboard (Stock levels)   │       │
│   │ └─ POS Terminals (Cash drawer status)   │       │
│   │ └─ GL Reconciliation (Bank status)      │       │
│   │ └─ Finance Dashboard (P&L summary)      │       │
│   │                                          │       │
│   └──────────────────────────────────────────┘       │
│                                                       │
│           [Continue →]                              │
│                                                       │
│   [← Back to Step 3]                               │
│                                                       │
│   Progress Indicator:                                │
│   ●───●───●───●───○───○                             │
│    1   2   3   4   5   6                             │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Module Selection Logic

```
Module Selection Rules (Backend):

Rule 1: Business Category → Auto-Enable Modules
├─ Category: TRADING
│   ├─ Auto-enable: Sales, Purchase, Inventory, GL
│   ├─ Auto-disable: Manufacturing, Payroll
│   └─ Optional: POS (only if retail), CRM
│
├─ Category: RETAIL
│   ├─ Auto-enable: Sales, Purchase, Inventory, GL, POS
│   ├─ Auto-disable: Manufacturing, Payroll
│   └─ Optional: CRM
│
├─ Category: DISTRIBUTION
│   ├─ Auto-enable: Sales, Purchase, Inventory, GL
│   ├─ Auto-disable: Manufacturing, Payroll
│   └─ Optional: POS, CRM
│
└─ Category: MANUFACTURING
    ├─ Auto-enable: Sales, Purchase, Inventory, GL, Manufacturing
    ├─ Auto-disable: POS (unless also retail)
    └─ Optional: Payroll, CRM

Rule 2: Module Interdependencies
├─ Sales depends on: Inventory, GL (can't disable)
├─ Purchase depends on: Inventory, GL (can't disable)
├─ Inventory depends on: GL (can't disable)
├─ GL is always enabled (no checkbox)
├─ POS depends on: Sales, Inventory (can't disable POS without disabling these)
└─ CRM depends on: Sales (can't use CRM without sales tracking)

Rule 3: Feature Gating (MVP vs Phase 2+)
├─ Phase 1 (Go-live): Sales, Purchase, Inventory, GL, POS
├─ Phase 2: CRM, Payroll (beta)
├─ Phase 3: Manufacturing, Advanced HR
└─ Greyed-out modules show "Coming in Phase X" status

Rule 4: License & Pricing (Future, not MVP)
├─ Core modules: Included in all plans
├─ Optional modules: Additional cost per user/month
└─ Button: "View Pricing" (for transparency)
```

### Backend Trigger on Submit (Step 4)

```
User selects modules, clicks "Continue"
    ↓
Frontend Validation:
├─ At least core modules selected ✓
├─ Dependencies satisfied ✓
├─ No disabled modules forced ✓
    ↓
API Call:
POST /auth/register-step-4
{
  "session_id": "sess_ghi789",
  "company_id": "comp_abc123",
  "modules_enabled": {
    "SALES": true,
    "PURCHASE": true,
    "INVENTORY": true,
    "GL": true,
    "POS": true,
    "CRM": false,
    "MANUFACTURING": false,
    "PAYROLL": false
  }
}

Backend Processing:
├─ Validate module selection:
│   ├─ Core modules (Sales, Purchase, Inventory, GL) required ✓
│   ├─ Check dependencies satisfied ✓
│   ├─ Verify phase eligibility (can enable Phase 1 modules only)
│   └─ If invalid: Return 400 error with reason
│
├─ Store module preferences:
│   ├─ company_modules table:
│   │   ├─ company_id = (FK)
│   │   ├─ module_code = "SALES"
│   │   ├─ enabled = true
│   │   ├─ enabled_at = NOW()
│   │   ├─ enabled_by_user_id = (FK)
│   │   └─ ... (repeat for each module)
│   │
│   └─ Used for:
│       ├─ Dashboard widget visibility
│       ├─ API endpoint access control
│       ├─ Feature flag checks
│       └─ License/pricing calculations
│
├─ Create default module settings:
│   ├─ If POS enabled:
│   │   ├─ pos_terminals table: Create template for 1 terminal
│   │   ├─ pos_settings table: Default tax, pricing rules
│   │   └─ cash_drawer_logs table: Initialize
│   │
│   ├─ If CRM enabled:
│   │   ├─ crm_settings table: Create default pipeline stages
│   │   └─ lead_sources table: Populate defaults
│   │
│   └─ If Manufacturing enabled:
│       ├─ bom_master table: Initialize
│       └─ production_orders table: Create workflow
│
├─ Configure dashboard widgets:
│   ├─ user_dashboard_widgets table:
│   │   ├─ user_id = (owner user)
│   │   ├─ widget_code = "SALES_KPI"
│   │   ├─ module = "SALES"
│   │   ├─ enabled = true
│   │   ├─ position = 1
│   │   └─ created_at = NOW()
│   │
│   └─ Create default widgets:
│       ├─ For Sales: Sales KPI, AR Aging
│       ├─ For Inventory: Stock Levels, Low Stock Alerts
│       ├─ For GL: Trial Balance, Bank Reconciliation Status
│       ├─ For POS: Terminal Status, Cash Drawer Status
│       └─ (Customizable later)
│
├─ Update onboarding session:
│   ├─ onboarding_sessions table:
│   │   ├─ stage = 4
│   │   ├─ data = {...module selections...}
│   │   └─ updated_at = NOW()

Response (200 OK):
{
  "status": "success",
  "message": "Modules enabled",
  "modules_enabled": 5,
  "next_step": 5,
  "dashboard_widgets_created": 5
}

Frontend Actions:
├─ Show success toast: "Modules enabled ✓"
├─ Redirect to Step 5 (User Provisioning)
└─ Preserve module selection in session
```

---

## Step 5: Initial User Provisioning (Admin Invite)

### UI Layout - Step 5 (Condensed)

```
┌─────────────────────────────────────────────────────┐
│  Niyanthra Logo          [EN ▼] [Help] [Login →]    │
├─────────────────────────────────────────────────────┤
│                                                       │
│   Step 5 of 6: Invite Your Team                     │
│   ─────────────────────────────────────────────────  │
│   Add accountants, warehouse managers, or sales     │
│   staff (optional for now, can add anytime)         │
│                                                       │
│   ┌──────────────────────────────────────────┐       │
│   │ First Team Member (Example)              │       │
│   │                                          │       │
│   │ Full Name *                              │       │
│   │ [Priya Sharma                            ]       │
│   │                                          │       │
│   │ Email Address *                          │       │
│   │ [priya@abctrading.com                    ]       │
│   │                                          │       │
│   │ Role *                                   │       │
│   │ [Accountant ▼]                          ]       │
│   │ Permissions: GL posting, AR/AP, Tax     │       │
│   │                                          │       │
│   │ ☑ Send invite email (auto-checked)      │       │
│   │                                          │       │
│   │ [+ Add Another User]                    │       │
│   │                                          │       │
│   │ ────────────────────────────────────────│       │
│   │                                          │       │
│   │ Invited Users Summary:                   │       │
│   │ • Priya Sharma (Accountant)              │       │
│   │ • Kumar Nair (Warehouse Manager) [opt.]  │       │
│   │                                          │       │
│   │ [Skip for Now]    [Continue →]          │       │
│   │                                          │       │
│   └──────────────────────────────────────────┘       │
│                                                       │
│   Progress Indicator:                                │
│   ●───●───●───●───●───○                             │
│    1   2   3   4   5   6                             │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Role Selection & RBAC Initialization

```
Role Options (Dropdown):
├─ Company Admin
│   ├─ Permissions: All (company_admin role)
│   └─ Limited to: Typically owner only
│
├─ Accountant
│   ├─ Permissions: GL posting, AR/AP management, tax filing, bank recon
│   └─ Modules: GL, Accounting (not Sales/Inventory creation)
│
├─ Warehouse Manager
│   ├─ Permissions: Inventory management, GRN, stock transfers, dispatch
│   └─ Modules: Inventory, Warehouse (not Finance/GL)
│
├─ Sales Manager
│   ├─ Permissions: Create SO, invoicing, customer management
│   └─ Modules: Sales, CRM (not GL, Purchase)
│
├─ Finance Manager
│   ├─ Permissions: GL, AR/AP, bank recon, financial close, GSTR filing
│   └─ Modules: GL, Accounting, Reporting (full access like accountant+)
│
├─ Viewer (Read-Only)
│   ├─ Permissions: View only (no create/edit/delete)
│   └─ Modules: All (read-only access)
│
└─ Custom Role
    ├─ Permissions: Admin selects specific permissions
    └─ (For Phase 2 advanced setup)

Database: user_roles Initialization
├─ Primary Admin (User from Step 1):
│   ├─ user_id = (from Step 1)
│   ├─ company_id = (from Step 3)
│   ├─ role = "COMPANY_ADMIN"
│   ├─ permissions = [all]
│   └─ created_at = NOW()
│
├─ Secondary User (From Step 5 invite):
│   ├─ user_id = (new user, created on invite accept)
│   ├─ company_id = (same as admin)
│   ├─ role = "ACCOUNTANT" (selected in Step 5)
│   ├─ permissions = ['gl_post', 'ar_view', 'ap_view', 'tax_file']
│   └─ created_at = (when user accepts invite)
```

### Backend Trigger on Submit (Step 5)

```
User enters team member(s), clicks "Continue"
    ↓
API Call:
POST /auth/register-step-5
{
  "session_id": "sess_ijk012",
  "company_id": "comp_abc123",
  "invited_users": [
    {
      "full_name": "Priya Sharma",
      "email": "priya@abctrading.com",
      "role": "ACCOUNTANT",
      "send_invite": true
    }
  ]
}

Backend Processing:
├─ Validate each invite:
│   ├─ Email format valid ✓
│   ├─ Email not already registered ✓
│   ├─ Email not same as primary admin ✓
│   ├─ Role exists in system ✓
│   └─ Admin has permission to invite (always yes on Step 5)
│
├─ Create invite records:
│   ├─ user_invites table:
│   │   ├─ invite_id = UUID
│   │   ├─ company_id = (FK)
│   │   ├─ invited_by_user_id = (primary admin)
│   │   ├─ invitee_email = "priya@abctrading.com"
│   │   ├─ full_name = "Priya Sharma"
│   │   ├─ role = "ACCOUNTANT"
│   │   ├─ status = 'PENDING'
│   │   ├─ invite_token = secure_random_token()
│   │   ├─ expires_at = NOW() + 7 days
│   │   ├─ sent_at = NULL (sent after commit)
│   │   └─ created_at = NOW()
│   │
│   └─ invite_token_hash table (secure storage):
│       ├─ invite_id = (FK)
│       ├─ token_hash = bcrypt(invite_token)
│       └─ created_at = NOW()
│
├─ Send invite emails (async, post-commit):
│   ├─ For each invitee:
│   │   ├─ Email: priya@abctrading.com
│   │   ├─ Subject: "Priya, join ABC Trading on Niyanthra ERP"
│   │   ├─ Body:
│   │   │   "Hi Priya,
│   │   │    Raj Kumar has invited you to manage accounting at ABC Trading.
│   │   │    Click below to accept and join:
│   │   │    [Accept Invite] (button link with token)
│   │   │    OR
│   │   │    https://niyanthra.io/accept-invite?token=[token]
│   │   │
│   │   │    Your role: Accountant
│   │   │    Permissions: GL posting, AR/AP, Tax Filing
│   │   │
│   │   │    This invitation expires in 7 days.
│   │   │    If you didn't expect this, please ignore."
│   │   │
│   │   └─ Send via email service (with retry logic)
│   │
│   └─ Mark as sent:
│       └─ user_invites.sent_at = NOW()
│
├─ Update onboarding session:
│   ├─ onboarding_sessions table:
│   │   ├─ stage = 5
│   │   ├─ data = {...invitee info...}
│   │   └─ updated_at = NOW()

Response (201 Created):
{
  "status": "success",
  "message": "Invites sent successfully",
  "invites_sent": 1,
  "invitees": [
    {
      "email": "priya@abctrading.com",
      "role": "ACCOUNTANT",
      "status": "PENDING",
      "expires_at": "2026-09-28T14:30:00Z"
    }
  ],
  "next_step": 6
}

Frontend Actions:
├─ Show success toast: "Invites sent ✓"
├─ Show invitee list (confirming who was invited)
├─ Provide link: "Copy invite link to share manually" (optional)
├─ Redirect to Step 6 (Workspace Initialization)
└─ Note: "You can invite more team members anytime from Settings"
```

---

## Step 6: Workspace Initialization & Guided First-Run Experience

### UI Layout - Step 6 (Dashboard Empty State)

```
┌─────────────────────────────────────────────────────────────────┐
│  Niyanthra Logo     ABC Trading (Raj Kumar)    [⚙ Settings]     │  ← Header
│  [Logout]                                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Welcome to Your Niyanthra Workspace!                            │
│  ────────────────────────────────────────────────────────────   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │                                                            │   │
│  │  👋 Welcome, Raj Kumar                                    │   │
│  │                                                            │   │
│  │  Your workspace is ready. Let's get started with your     │   │
│  │  first transaction!                                       │   │
│  │                                                            │   │
│  │  ┌────────────────────────────────────────────────────┐  │   │
│  │  │ Quick Start Guide (Step 1 of 5)                   │  │   │
│  │  │                                                    │  │   │
│  │  │ ☑ Step 1: Add Your First Product                │  │   │
│  │  │ ○ Step 2: Record an Opening Balance (Inventory)  │  │   │
│  │  │ ○ Step 3: Create Your First Sales Order         │  │   │
│  │  │ ○ Step 4: Generate an Invoice                   │  │   │
│  │  │ ○ Step 5: Reconcile Your Bank Account           │  │   │
│  │  │                                                    │  │   │
│  │  └────────────────────────────────────────────────────┘  │   │
│  │                                                            │   │
│  │  [Start with Adding Products ▶]                          │   │
│  │                                                            │   │
│  │  ──────────────────────────────────────────────────────   │   │
│  │  Or skip for now and explore:                            │   │
│  │  [Go to Dashboard] [View Settings] [Invite Team]        │   │
│  │                                                            │   │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌───────────┬───────────┬───────────┬────────────┐             │
│  │ Sales     │ Purchase  │ Inventory │ Accounting │  ← Sidebar  │
│  │           │           │           │            │  Navigation │
│  │ ○ Draft   │ ○ Draft   │ ○ Empty   │ ○ No GL    │             │
│  │ Quotations│ Orders    │ Stock     │ Entries    │             │
│  │ ○ No Sales│ ○ No Bills│           │            │             │
│  │ Orders    │ Payable   │           │            │             │
│  │           │           │           │            │             │
│  │ [+ Create │ [+ Create │ [+ Add    │ [+ Manual  │             │
│  │  New SO]  │  New PO]  │  Products]│  Entry]    │             │
│  └───────────┴───────────┴───────────┴────────────┘             │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Recommended Next Steps                                     │  │
│  │                                                            │  │
│  │ 1. [Complete Company Setup] (Bank, Tax Settings)         │  │
│  │    • Add bank account for cash reconciliation             │  │
│  │    • Verify GST & TDS settings                            │  │
│  │                                                            │  │
│  │ 2. [Import Products] (if migrating from Excel/Tally)     │  │
│  │    • Download CSV template                               │  │
│  │    • Upload product list                                 │  │
│  │                                                            │  │
│  │ 3. [Set Opening Balances] (Starting Inventory, AR, AP)   │  │
│  │    • Opening stock from previous system                   │  │
│  │    • Customer outstanding balances                       │  │
│  │    • Vendor outstanding bills                            │  │
│  │                                                            │  │
│  │ 4. [Invite Team Members] (Accountant, Sales, Warehouse)  │  │
│  │    • [Manage Team] or [Resend Invite Link]              │  │
│  │                                                            │  │
│  │ [View Complete Setup Checklist]                           │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Need Help?                                                 │  │
│  │ • [Read Documentation] (Guides, FAQs)                     │  │
│  │ • [Watch Videos] (5-min getting started, module overviews)│  │
│  │ • [Contact Support] (Email, Chat, Knowledge Base)        │  │
│  │ • [Schedule Demo] (Live walkthrough with specialist)     │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Backend Trigger on Submit (Step 6)

```
User views Step 6 Dashboard
    ↓
API Call:
GET /auth/register-step-6
{
  "session_id": "sess_lmn345",
  "company_id": "comp_abc123",
  "user_id": "user_xyz789"
}

Backend Processing:
├─ Mark onboarding as COMPLETE:
│   ├─ onboarding_sessions table:
│   │   ├─ stage = 6 (final)
│   │   ├─ status = 'COMPLETED'
│   │   ├─ completed_at = NOW()
│   │   └─ updated_at = NOW()
│   │
│   └─ onboarding_status table:
│       ├─ user_id = (FK)
│       ├─ company_id = (FK)
│       ├─ status = 'ACTIVATED'
│       ├─ activation_date = NOW()
│       └─ trial_expires_at = NOW() + 14 days (2-week free trial)
│
├─ Create session context:
│   ├─ Generate long-lived JWT (24 hours for dashboard use):
│   │   ├─ sub: user_id
│   │   ├─ org: company_id
│   │   ├─ roles: ['COMPANY_ADMIN']
│   │   ├─ permissions: ['all']
│   │   ├─ aud: 'niyanthra-app'
│   │   ├─ iat: NOW()
│   │   ├─ exp: NOW() + 24 hours
│   │   └─ verified: true
│   │
│   └─ Store in secure HTTP-only cookie (not localStorage)
│
├─ Initialize dashboard state:
│   ├─ company_dashboards table:
│   │   ├─ company_id = (FK)
│   │   ├─ dashboard_name = "Default Dashboard"
│   │   ├─ layout = "grid_4_cols"
│   │   ├─ created_by_user_id = (admin)
│   │   └─ created_at = NOW()
│   │
│   └─ dashboard_widgets table:
│       ├─ (Populate based on enabled modules from Step 4)
│       ├─ For Sales: Sales KPI, AR Aging
│       ├─ For Inventory: Stock Levels, Low Stock Alerts
│       ├─ For GL: Trial Balance, Bank Reconciliation
│       ├─ For POS: Terminal Status, Cash Drawer
│       └─ Each widget reads data (empty at start, shows "No data yet")
│
├─ Create first-run checklist:
│   ├─ onboarding_checklist table:
│   │   ├─ company_id = (FK)
│   │   ├─ checklist_item = "Add Bank Account"
│   │   ├─ priority = 'HIGH'
│   │   ├─ completed = false
│   │   ├─ help_link = "/help/add-bank-account"
│   │   └─ created_at = NOW()
│   │
│   └─ Sample checklist items:
│       ├─ Complete Company Setup (Bank, Tax Verification)
│       ├─ Import/Add Products
│       ├─ Set Opening Balances (Stock, AR, AP)
│       ├─ Invite Team Members
│       ├─ Configure Payment Terms
│       ├─ Set Up Billing Preferences
│       └─ Verify GST/TDS Settings
│
├─ Enable API access:
│   ├─ api_keys table:
│   │   ├─ company_id = (FK)
│   │   ├─ key = secure_random_key()
│   │   ├─ secret = secure_random_secret()
│   │   ├─ permissions = ['read', 'write'] (full access for admin)
│   │   ├─ rate_limit = 1000_requests_per_hour (generous for setup)
│   │   └─ created_at = NOW()
│   │
│   └─ (Not shown to user on Day 1, available in Settings → API)
│
├─ Start trial tracking:
│   ├─ trial_tracking table:
│   │   ├─ company_id = (FK)
│   │   ├─ trial_start_date = NOW()
│   │   ├─ trial_end_date = NOW() + 14 days
│   │   ├─ trial_status = 'ACTIVE'
│   │   ├─ feature_quota = {users: 5, products: 1000, invoices: unlimited}
│   │   └─ created_at = NOW()
│
├─ Log audit trail:
│   ├─ audit_logs table:
│   │   ├─ user_id = (FK)
│   │   ├─ company_id = (FK)
│   │   ├─ action = "onboarding_completed"
│   │   ├─ details = "Company ABC Trading onboarded, trial started"
│   │   ├─ timestamp = NOW()
│   │   ├─ ip_address = 203.0.113.45
│   │   └─ user_agent = "Mozilla/5.0..."
│
├─ Send welcome email:
│   ├─ Email: raj@abctrading.com
│   ├─ Subject: "🎉 Welcome to Niyanthra, ABC Trading!"
│   ├─ Body:
│   │   "Hi Raj,
│   │
│   │   Your Niyanthra workspace is ready to go!
│   │
│   │   What's next:
│   │   1. Add your products (inventory master)
│   │   2. Set opening balances (if migrating from another system)
│   │   3. Create your first sales order
│   │   4. Invite your team (accountant, warehouse manager, sales rep)
│   │
│   │   Start a free 14-day trial → expires [date].
│   │
│   │   [Go to Workspace] [Schedule a Demo] [Contact Support]
│   │
│   │   —Niyanthra Support"
│
└─ Enable onboarding features:
    ├─ Feature flags:
    │   ├─ 'onboarding_completed' = true (show dashboard, hide signup flow)
    │   ├─ 'first_run_mode' = true (show guided tooltips, checklist)
    │   ├─ 'trial_active' = true (show trial countdown)
    │   └─ 'api_enabled' = true (allow API calls, webhooks)

Response (200 OK):
{
  "status": "success",
  "message": "Onboarding completed!",
  "company_id": "comp_abc123",
  "user_id": "user_xyz789",
  "authorization": "Bearer eyJhbGciOi...",
  "trial_expires_at": "2026-09-28T14:30:00Z",
  "features_enabled": ["SALES", "PURCHASE", "INVENTORY", "GL", "POS"],
  "dashboard_ready": true,
  "checklist_items": 7,
  "next_action": "dashboard"
}

Frontend Actions:
├─ Store JWT in memory (auto-login, no login page needed)
├─ Navigate to dashboard home
├─ Display welcome overlay (Step 6 UI)
├─ Show first-run onboarding mode:
│   ├─ Highlight key sections (sidebar modules)
│   ├─ Show tooltips on hover
│   ├─ Display guided checklist
│   └─ Enable "Quick Start" tutorial
│
├─ Log successful signup event (analytics):
│   ├─ Event: "signup_completed"
│   ├─ company_id: comp_abc123
│   ├─ user_id: user_xyz789
│   ├─ modules_enabled: 5
│   ├─ business_category: "TRADING"
│   └─ signup_duration: 7_minutes (from Step 1 to 6)
│
└─ Set up auto-logout:
    ├─ Session expires after 24 hours of inactivity
    ├─ Warn user 5 minutes before expiry
    └─ Redirect to login if session expires
```

---

# PART 3: SECURITY, DATA ISOLATION & MULTI-TENANCY RULES

---

## 3.1 Multi-Tenant Data Isolation

### Tenant Partitioning Model

```
Niyanthra (Platform)
├─ Tenant 1: ABC Trading (company_id = comp_abc123)
│   ├─ Users: Raj (admin), Priya (accountant), Kumar (warehouse)
│   ├─ Data:
│   │   ├─ Sales Orders (100 records, all company_id = comp_abc123)
│   │   ├─ Invoices (150 records, all company_id = comp_abc123)
│   │   ├─ GL Entries (5000 records, all company_id = comp_abc123)
│   │   └─ Inventory (500 products, all company_id = comp_abc123)
│   │
│   └─ [No data can cross company_id boundary]
│
├─ Tenant 2: XYZ Retail (company_id = comp_xyz456)
│   ├─ Users: Anjali (admin), Rohan (sales), Neha (inventory)
│   ├─ Data:
│   │   ├─ Sales Orders (50 records, all company_id = comp_xyz456)
│   │   ├─ Invoices (75 records, all company_id = comp_xyz456)
│   │   └─ Inventory (300 products, all company_id = comp_xyz456)
│   │
│   └─ [Completely isolated from Tenant 1]
│
└─ Tenant 3: MNO Distribution (company_id = comp_mno789)
    └─ ...
```

### Query-Level Isolation (Critical)

```
Enforce company_id filter on EVERY query:

WRONG (Security Violation):
┌─ SELECT * FROM sales_orders WHERE customer_id = 101
│ ├─ Returns orders from ALL companies (data leak!)
│ └─ NEVER allow this
└─

CORRECT (Safe Query):
┌─ SELECT * FROM sales_orders 
│ WHERE company_id = 'comp_abc123' 
│   AND customer_id = 101
│ └─ Only returns data for ABC Trading
└─

Implementation Pattern (Backend):
├─ Every API endpoint checks current_user.company_id
├─ Every database query adds WHERE company_id = current_user.company_id
├─ Middleware enforces company_id from JWT claims
└─ Example:

    @app.route('/api/sales-orders', methods=['GET'])
    @require_auth
    def get_sales_orders():
        user = get_current_user()  # JWT claims
        company_id = user['company_id']  # Extract from token
        
        # Query with company_id filter (ALWAYS)
        orders = db.query(SalesOrder)
            .filter(SalesOrder.company_id == company_id)
            .all()
        
        return jsonify(orders)
```

### Database Constraint Enforcement

```
Foreign Key Constraints with company_id:

-- Example: Sales Orders can only reference customers from same company
ALTER TABLE sales_orders ADD CONSTRAINT fk_so_customer
FOREIGN KEY (company_id, customer_id) 
REFERENCES customers(company_id, id);
-- This ensures: If SO created for comp_abc123, 
--              customer MUST also have company_id = comp_abc123
--              (No cross-company order possible)

-- Example: GL Entries must reference accounts in same company
ALTER TABLE gl_entries ADD CONSTRAINT fk_gl_account
FOREIGN KEY (company_id, account_id) 
REFERENCES chart_of_accounts(company_id, id);

-- Example: Invoices must reference customers in same company
ALTER TABLE tax_invoices ADD CONSTRAINT fk_invoice_customer
FOREIGN KEY (company_id, customer_id)
REFERENCES customers(company_id, id);

-- Pattern: All composite FKs include company_id
-- Result: Physical database constraint prevents cross-tenant data contamination
```

### Index Strategy (Performance + Security)

```
Indices for fast, safe queries:

CREATE INDEX idx_sales_orders_company_customer
ON sales_orders(company_id, customer_id);
-- Supports: SELECT * FROM sales_orders 
--           WHERE company_id = ? AND customer_id = ?

CREATE INDEX idx_invoices_company_status
ON tax_invoices(company_id, status);
-- Supports: SELECT * FROM tax_invoices
--           WHERE company_id = ? AND status = 'UNPAID'

CREATE INDEX idx_gl_entries_company_account
ON gl_entries(company_id, account_id);
-- Supports: SELECT * FROM gl_entries
--           WHERE company_id = ? AND account_id = ?

CREATE INDEX idx_users_company
ON users(company_id);
-- Supports: SELECT * FROM users
--           WHERE company_id = ? (for team member listing)

Strategy: Every company_id filter must have a supporting index
Result: Queries are fast (< 10ms for 1M records)
```

---

## 3.2 Session Management & Authentication

### JWT Token Structure

```
Header:
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "key_2026_09"  // Key ID for rotation
}

Payload (Claims):
{
  "sub": "user_xyz789",                    // Subject (user ID)
  "org": "comp_abc123",                    // Organization (company ID) — CRITICAL
  "iat": 1694856000,                       // Issued at
  "exp": 1694942400,                       // Expires in (24 hours)
  "aud": "niyanthra-app",                  // Audience (this app only)
  "iss": "https://niyanthra.io",           // Issuer
  "email": "raj@abctrading.com",           // Email
  "roles": ["COMPANY_ADMIN"],              // Roles (array)
  "permissions": ["*"],                    // Permissions (all for admin)
  "verified": true,                        // Email verified
  "session_id": "sess_ghi789"              // Session tracking
}

Signature:
HMAC256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)

Storage:
├─ HTTP-only cookie (not JavaScript accessible, protects against XSS)
│   ├─ Name: 'auth_token'
│   ├─ Path: '/'
│   ├─ Domain: '.niyanthra.io'
│   ├─ Secure: true (HTTPS only)
│   ├─ HttpOnly: true (no JS access)
│   ├─ SameSite: Strict (prevents CSRF)
│   └─ Max-Age: 86400 (24 hours)
│
└─ Memory (for SPA, in case cookie not available)
    ├─ Store in JS memory (cleared on page refresh)
    ├─ Never persist to localStorage (XSS risk)
    └─ Auto-clear on logout or session expiry
```

### Token Lifecycle

```
Step 1-2 (Anonymous):
├─ No token yet (user unverified)
├─ Temporary session_id used (single-use, 1-hour TTL)
└─ Can only call /auth/register-* endpoints

Step 3 (Company Created):
├─ Token created after company_id assigned
├─ Claims include: company_id = "comp_abc123"
├─ Token valid for Steps 3-6 (4 more steps)
└─ TTL: 24 hours (enough to complete onboarding)

Step 6 (Onboarding Complete):
├─ New token issued (long-lived session)
├─ Role: COMPANY_ADMIN (full permissions)
├─ Permissions: * (all)
├─ TTL: 24 hours (auto-refresh on activity)
└─ Refresh mechanism:
    ├─ After 12 hours of inactivity: Require re-login
    ├─ After 20 hours active: Token auto-rotated (new key issued)
    └─ On logout: Token blacklisted (added to denylist)

Token Rotation (Security):
├─ Every 24 hours: Issue new token, revoke old one
├─ Old token denylist: Stored in Redis (1-day TTL)
├─ Prevents: Token replay attacks, stolen token reuse
└─ Transparent to user (automatic refresh in background)

Logout Process:
├─ Clear HTTP-only cookie (server-side)
├─ Add token to denylist (Redis, 24-hour TTL)
├─ Clear browser session storage
├─ Redirect to login page
└─ Token cannot be reused (blacklisted)
```

---

## 3.3 Password Security

### Password Requirements & Hashing

```
Requirements (Enforced at registration):
├─ Minimum 8 characters
├─ At least 1 uppercase letter (A-Z)
├─ At least 1 number (0-9)
├─ At least 1 special character (optional, but encouraged)
│   └─ Allowed: !@#$%^&*()-_=+[]{}|;:',.<>?/~`
├─ Not in common password list (checked against rockyou.txt, 10M passwords)
├─ Not previously breached (checked via haveibeenpwned.com API)
└─ Not the same as email or company name

Validation (Real-time feedback):
├─ Length: ✓ 8+ characters
├─ Uppercase: ✓ Contains A-Z
├─ Number: ✓ Contains 0-9
├─ Special: ○ Add special char (encouraged)
└─ Strength meter: Weak → Fair → Good → Strong

Hashing (Server-side):
├─ Algorithm: bcrypt (industry standard)
├─ Salt rounds: 12 (cost factor = 2^12 = 4096 iterations)
├─ Process:
│   1. Generate random salt: salt = bcrypt.genSalt(12)
│   2. Hash password: hash = bcrypt.hash(password, salt)
│   3. Store hash: users.password_hash = hash
│   └─ Example hash: $2b$12$WQvkI5Uy89E3OiYJ5D9w4emR8S2bqc7gMvF.zN.5JJ0.d8q9TBG7m
│
└─ Note: Original password is NEVER stored or logged

Verification (Login):
├─ User enters password
├─ Retrieve hash from DB: stored_hash = users.password_hash
├─ Compare: bcrypt.compare(entered_password, stored_hash)
├─ Result: Boolean (true if match, false if not)
└─ Time: ~100ms per check (intentional slowness prevents brute force)

Password Reset (Future):
├─ User requests reset (forgot password)
├─ Generate reset token (secure random, 32 bytes)
├─ Store token_hash in password_resets table (with 1-hour TTL)
├─ Send reset link to verified email
├─ User clicks link, enters new password
├─ Verify token, update password_hash
└─ Invalidate all existing tokens (force re-login)
```

---

## 3.4 Rate Limiting & Abuse Prevention

### Brute Force Protection

```
Login Attempt Tracking:
├─ failed_login_attempts table:
│   ├─ ip_address
│   ├─ email
│   ├─ attempt_timestamp
│   └─ user_agent

Rules:
├─ Max 5 failed attempts from same IP/5 minutes → Block IP (5-min timeout)
├─ Max 3 failed attempts for same email/5 minutes → Account locked (15-min timeout)
├─ Max 10 failed attempts from same IP/hour → Block IP (1-hour timeout)
├─ Exponential backoff: 1st fail (1s delay), 2nd (2s), 3rd (4s), etc.
└─ Log attempt: Include timestamp, IP, user-agent for audit trail

OTP Brute Force Protection:
├─ Max 5 wrong OTP attempts per user/5 minutes → Expire OTP, force resend
├─ Max 3 resend OTP requests per user/hour → Rate limited
├─ Throttle: Wait 30 seconds between OTP requests
└─ Log attempt: timestamp, IP, outcome
```

### Signup & Email Spam Prevention

```
Signup Rate Limiting:
├─ Max 5 signup attempts per IP address/hour
├─ Max 1 signup per email address (globally unique)
├─ Max 1 signup per phone number (if provided)
├─ Verify email before proceeding (OTP gate)
└─ Timeout: 24 hours to complete onboarding (else session expires)

Email Verification Throttle:
├─ Max 3 OTP resend requests per email/hour
├─ Cooldown: 30 seconds between resends
└─ Auto-expire OTP after 5 minutes (force new request)

Company Duplicate Prevention:
├─ GSTIN unique globally (can't register same GSTIN twice)
├─ PAN unique per state (prevent duplicates)
├─ Company name checked for similarity (warn if duplicate likely)
└─ Manual review if suspicious
```

---

# PART 4: IMPLEMENTATION GUIDELINES FOR DEVELOPERS

---

## 4.1 API Endpoints Required for Each Step

### Authentication Endpoints

```
Step 1: Register (Identity Capture)
────────────────────────────────────

POST /auth/register-step-1
Request:
{
  "email": "raj@abctrading.com",
  "phone": "+919876543210",
  "password_hash": "bcrypt(...)",
  "source": "organic" | "invite_link"
}

Response (201 Created):
{
  "status": "success",
  "session_id": "sess_abc123",
  "message": "OTP sent to raj@abctrading.com",
  "otp_delivery_method": "email",
  "expires_in_seconds": 300
}

Error (400 Bad Request):
{
  "error_code": "EMAIL_INVALID",
  "message": "Invalid email format",
  "field": "email"
}

Error (409 Conflict):
{
  "error_code": "EMAIL_ALREADY_EXISTS",
  "message": "This email is already registered",
  "field": "email"
}

POST /auth/check-email (for Step 1 form validation)
Request:
{ "email": "raj@abctrading.com" }

Response:
{
  "exists": false,
  "available": true,
  "free_email": false  // true if @gmail.com, @yahoo.com (warn but allow)
}


Step 2: OTP Verification
────────────────────────

POST /auth/verify-otp
Request:
{
  "session_id": "sess_abc123",
  "otp_code": "123456",
  "method": "email" | "sms"
}

Response (200 OK):
{
  "status": "success",
  "message": "Email verified successfully",
  "user_id": "user_xyz789",
  "authorization": "Bearer eyJhbGciOi...",
  "next_step": 3
}

Error (400 Bad Request):
{
  "error_code": "INVALID_OTP",
  "message": "Incorrect OTP. Attempts remaining: 2",
  "attempts_remaining": 2
}

Error (429 Too Many Requests):
{
  "error_code": "RATE_LIMIT_EXCEEDED",
  "message": "Too many OTP attempts. Try again in 5 minutes.",
  "retry_after_seconds": 300
}

POST /auth/resend-otp
Request:
{ "session_id": "sess_abc123" }

Response:
{
  "status": "success",
  "message": "New OTP sent to raj@abctrading.com",
  "expires_in_seconds": 300
}

Error (429 Too Many Requests):
{
  "error_code": "RESEND_RATE_LIMITED",
  "message": "You can resend OTP in 30 seconds",
  "retry_after_seconds": 30
}

POST /auth/resend-otp-sms
Request:
{ "session_id": "sess_abc123" }

Response:
{
  "status": "success",
  "message": "OTP sent via SMS to +91 9876543210",
  "expires_in_seconds": 300
}


Step 3: Company Profile
───────────────────────

POST /auth/register-step-3
Request:
{
  "session_id": "sess_def456",
  "company_name": "ABC Trading Pvt Ltd",
  "business_type": "PRIVATE_LIMITED",
  "category": "TRADING",
  "gstin": "27AABCT1234H1Z5",
  "pan": "AABCT1234H",
  "state": "KERALA",
  "address": "XYZ Road, Kochi",
  "city": "KOCHI",
  "pincode": "682001",
  "fy_end_month": 3,
  "gst_registered": true,
  "tds_filer": true
}

Response (201 Created):
{
  "status": "success",
  "company_id": "comp_abc123",
  "company_name": "ABC Trading Pvt Ltd",
  "gstin": "27AABCT1234H1Z5",
  "coa_created": true,
  "authorization": "Bearer eyJhbGciOi...",
  "next_step": 4
}

Error (400 Bad Request):
{
  "error_code": "GSTIN_INVALID",
  "message": "GSTIN format invalid. Expected: SSGGAAAXNNCNNNNNC",
  "field": "gstin"
}

Error (409 Conflict):
{
  "error_code": "GSTIN_ALREADY_REGISTERED",
  "message": "This GSTIN is already registered in the system",
  "field": "gstin"
}

GET /auth/validate-gstin
Request:
{ "gstin": "27AABCT1234H1Z5" }

Response:
{
  "valid": true,
  "registered_name": "ABC TRADING",
  "state": "27",
  "state_name": "KARNATAKA",
  "pan": "AABCT1234H"
}

GET /auth/cities/{state_code}
Request:
{ "state_code": "32" }  // Kerala

Response:
{
  "state": "KERALA",
  "cities": [
    { "id": "C001", "name": "Kochi" },
    { "id": "C002", "name": "Thiruvananthapuram" },
    { "id": "C003", "name": "Kozhikode" }
  ]
}


Step 4: Module Selection
────────────────────────

POST /auth/register-step-4
Request:
{
  "session_id": "sess_ghi789",
  "company_id": "comp_abc123",
  "modules_enabled": {
    "SALES": true,
    "PURCHASE": true,
    "INVENTORY": true,
    "GL": true,
    "POS": true,
    "CRM": false,
    "MANUFACTURING": false,
    "PAYROLL": false
  }
}

Response (200 OK):
{
  "status": "success",
  "modules_enabled": 5,
  "dashboard_widgets_created": 5,
  "next_step": 5
}


Step 5: User Provisioning (Invite)
──────────────────────────────────

POST /auth/register-step-5
Request:
{
  "session_id": "sess_ijk012",
  "company_id": "comp_abc123",
  "invited_users": [
    {
      "full_name": "Priya Sharma",
      "email": "priya@abctrading.com",
      "role": "ACCOUNTANT",
      "send_invite": true
    }
  ]
}

Response (201 Created):
{
  "status": "success",
  "invites_sent": 1,
  "invitees": [
    {
      "email": "priya@abctrading.com",
      "role": "ACCOUNTANT",
      "status": "PENDING",
      "expires_at": "2026-09-28T14:30:00Z"
    }
  ],
  "next_step": 6
}


Step 6: Workspace Initialization
─────────────────────────────────

GET /auth/register-step-6
Request:
{ "session_id": "sess_lmn345" }

Response (200 OK):
{
  "status": "success",
  "company_id": "comp_abc123",
  "user_id": "user_xyz789",
  "authorization": "Bearer eyJhbGciOi...",
  "trial_expires_at": "2026-09-28T14:30:00Z",
  "features_enabled": ["SALES", "PURCHASE", "INVENTORY", "GL", "POS"],
  "dashboard_ready": true,
  "next_action": "dashboard"
}
```

---

## 4.2 Database State Changes Through Onboarding

### State Transition Diagram

```
BEGIN → Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 6 → ACTIVE
  ✗      ✗        ✓        ✓        ✓        ✓        ✓        ✓
  │      │        │        │        │        │        │        │
  └──────┴────────┴────────┴────────┴────────┴────────┴────────┘
       (Any step failure rolls back)

Step 1 Tables Created:
├─ users (status = 'UNVERIFIED')
├─ onboarding_sessions (stage = 1)
└─ otp_store (for email verification)

Step 2 Tables Updated:
├─ users (status = 'VERIFIED')
├─ onboarding_sessions (stage = 2)
└─ otp_store (deleted after verification)

Step 3 Tables Created:
├─ companies (NEW TENANT!)
├─ chart_of_accounts (100+ rows for this company)
├─ tax_rules
├─ warehouses
├─ bank_accounts
├─ company_settings
└─ onboarding_sessions (stage = 3, company_id set)

Step 4 Tables Created:
├─ company_modules (5 rows: Sales, Purchase, Inventory, GL, POS)
├─ user_dashboard_widgets (5 rows: default dashboard)
└─ onboarding_sessions (stage = 4)

Step 5 Tables Created:
├─ user_invites (1 row per invitee)
├─ user_invites_tokens (secure token hash)
└─ onboarding_sessions (stage = 5)

Step 6 Tables Updated/Created:
├─ users (status = 'ACTIVE')
├─ onboarding_sessions (stage = 6, status = 'COMPLETED', completed_at = NOW())
├─ onboarding_status (status = 'ACTIVATED', trial_expires_at)
├─ onboarding_checklist (7 items)
├─ api_keys (for API access)
├─ trial_tracking
├─ audit_logs (onboarding_completed)
└─ All data ready for use!
```

---

## 4.3 Code Examples (Backend)

### Step 1: Register Handler (Pseudocode)

```python
@app.route('/auth/register-step-1', methods=['POST'])
@rate_limit(max_requests=5, window_minutes=60)  # 5 per hour
def register_step_1():
    data = request.get_json()
    
    # Validation
    email = data.get('email', '').lower()
    password = data.get('password')
    phone = data.get('phone', '')
    
    # Email validation
    if not is_valid_email(email):
        return {"error_code": "EMAIL_INVALID"}, 400
    
    # Check if email already registered
    if db.query(User).filter(User.email == email).first():
        return {"error_code": "EMAIL_ALREADY_EXISTS"}, 409
    
    # Password validation
    password_strength = check_password_strength(password)
    if not password_strength['valid']:
        return {"error_code": "PASSWORD_WEAK", "reasons": password_strength['reasons']}, 400
    
    # Hash password using bcrypt
    password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12)).decode()
    
    # Create user (unverified)
    user = User(
        user_id=generate_uuid(),
        email=email,
        phone=phone,
        password_hash=password_hash,
        status='UNVERIFIED',
        created_at=datetime.now()
    )
    db.session.add(user)
    db.session.flush()  # Get user_id
    
    # Generate OTP (6 digits)
    otp_code = generate_random_otp(length=6)
    otp_hash = bcrypt.hashpw(str(otp_code).encode(), bcrypt.gensalt()).decode()
    
    # Store OTP (expires in 5 minutes)
    otp_record = OTP(
        user_id=user.user_id,
        otp_hash=otp_hash,
        method='EMAIL',
        expires_at=datetime.now() + timedelta(minutes=5),
        attempts=0
    )
    db.session.add(otp_record)
    
    # Create onboarding session
    session_id = generate_session_id()
    onboarding_session = OnboardingSession(
        session_id=session_id,
        user_id=user.user_id,
        stage=1,
        data=json.dumps({"email": email, "phone": phone}),
        expires_at=datetime.now() + timedelta(hours=1)
    )
    db.session.add(onboarding_session)
    
    # Commit all changes atomically
    db.session.commit()
    
    # Send OTP email (async, after commit)
    send_otp_email_async(email, otp_code)
    
    return {
        "status": "success",
        "session_id": session_id,
        "message": f"OTP sent to {email}",
        "otp_delivery_method": "email",
        "expires_in_seconds": 300
    }, 201
```

### Step 3: Company Creation Handler (Pseudocode)

```python
@app.route('/auth/register-step-3', methods=['POST'])
@require_onboarding_session(stage=2)  # Only from Step 2
def register_step_3():
    data = request.get_json()
    session_id = data['session_id']
    
    # Fetch session (verify still valid)
    onboarding_session = db.query(OnboardingSession)\
        .filter(OnboardingSession.session_id == session_id).first()
    if not onboarding_session or onboarding_session.expires_at < datetime.now():
        return {"error_code": "SESSION_EXPIRED"}, 401
    
    # Get user from session
    user = db.query(User).filter(User.user_id == onboarding_session.user_id).first()
    if not user or user.status != 'VERIFIED':
        return {"error_code": "USER_NOT_VERIFIED"}, 401
    
    # Validate GSTIN format
    gstin = data.get('gstin', '').upper()
    if not is_valid_gstin_format(gstin):
        return {"error_code": "GSTIN_INVALID"}, 400
    
    # Check GSTIN uniqueness
    if db.query(Company).filter(Company.gstin == gstin).first():
        return {"error_code": "GSTIN_ALREADY_REGISTERED"}, 409
    
    # Extract PAN from GSTIN if not provided
    pan = data.get('pan') or extract_pan_from_gstin(gstin)
    
    # START ATOMIC TRANSACTION
    try:
        # 1. Create company (CRITICAL — new tenant)
        company_id = generate_uuid()
        company = Company(
            company_id=company_id,
            user_id=user.user_id,  # Ownership
            company_name=data['company_name'],
            gstin=gstin,
            pan=pan,
            business_type=data['business_type'],
            category=data['category'],
            state=data['state'],
            address=data['address'],
            city=data['city'],
            pincode=data['pincode'],
            fy_end_month=data['fy_end_month'],
            gst_registered=data.get('gst_registered', True),
            tds_filer=data.get('tds_filer', True),
            status='ACTIVE',
            created_at=datetime.now(),
            created_by_user_id=user.user_id
        )
        db.session.add(company)
        db.session.flush()  # Get company_id
        
        # 2. Create Chart of Accounts for this company
        default_coa = get_default_coa_template(business_type=data['business_type'])
        for account in default_coa:
            coa_entry = ChartOfAccounts(
                company_id=company_id,
                account_code=account['code'],
                account_name=account['name'],
                account_group=account['group'],
                balance=0.00,
                created_at=datetime.now()
            )
            db.session.add(coa_entry)
        
        # 3. Create tax rules for this company
        tax_rules = TaxRules(
            company_id=company_id,
            state=data['state'],
            gst_registered=data.get('gst_registered', True),
            intra_state_rules='SGST+CGST',
            inter_state_rules='IGST',
            fy_end_month=data['fy_end_month'],
            created_at=datetime.now()
        )
        db.session.add(tax_rules)
        
        # 4. Create default warehouse
        warehouse = Warehouse(
            company_id=company_id,
            warehouse_id='WH-001',
            name='Main Warehouse',
            address=data['address'],
            city=data['city'],
            pincode=data['pincode'],
            status='ACTIVE',
            created_at=datetime.now()
        )
        db.session.add(warehouse)
        
        # 5. Create company settings
        company_settings = CompanySettings(
            company_id=company_id,
            currency='INR',
            timezone='Asia/Kolkata',
            date_format='DD-MM-YYYY',
            created_at=datetime.now()
        )
        db.session.add(company_settings)
        
        # 6. Initialize user roles for this company
        user_role = UserRole(
            user_id=user.user_id,
            company_id=company_id,
            role='COMPANY_ADMIN',
            permissions=['*'],
            created_at=datetime.now()
        )
        db.session.add(user_role)
        
        # 7. Update onboarding session
        onboarding_session.company_id = company_id
        onboarding_session.stage = 3
        onboarding_session.data = json.dumps(data)
        onboarding_session.updated_at = datetime.now()
        
        # COMMIT ALL CHANGES
        db.session.commit()
        
    except Exception as e:
        db.session.rollback()
        return {"error_code": "COMPANY_CREATION_FAILED", "message": str(e)}, 500
    
    # Log audit trail
    log_audit(
        user_id=user.user_id,
        company_id=company_id,
        action='company_created',
        details=f"Created company {data['company_name']}, GSTIN {gstin}"
    )
    
    # Generate JWT with company_id
    jwt_token = create_jwt_token(
        user_id=user.user_id,
        company_id=company_id,
        role='COMPANY_ADMIN',
        expires_in_hours=24
    )
    
    return {
        "status": "success",
        "company_id": company_id,
        "company_name": data['company_name'],
        "gstin": gstin,
        "authorization": f"Bearer {jwt_token}",
        "next_step": 4
    }, 201
```

---

## 4.4 Database Schema (DDL)

### Key Tables

```sql
-- Users Table (Global)
CREATE TABLE users (
  user_id VARCHAR(36) PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(20),
  password_hash VARCHAR(255) NOT NULL,
  status ENUM('UNVERIFIED', 'VERIFIED', 'ACTIVE', 'SUSPENDED') DEFAULT 'UNVERIFIED',
  email_verified_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_email (email),
  INDEX idx_status (status)
);

-- Companies Table (Tenants)
CREATE TABLE companies (
  company_id VARCHAR(36) PRIMARY KEY,
  user_id VARCHAR(36) NOT NULL,
  company_name VARCHAR(150) NOT NULL,
  gstin VARCHAR(15) UNIQUE NOT NULL,
  pan VARCHAR(10) NOT NULL,
  business_type ENUM('PROPRIETORSHIP', 'PARTNERSHIP', 'PRIVATE_LIMITED', 'PUBLIC_LIMITED', 'LLP', 'SOLE_TRADER', 'OTHER'),
  category VARCHAR(50),
  state VARCHAR(50),
  address TEXT,
  city VARCHAR(100),
  pincode VARCHAR(6),
  fy_end_month INT,
  gst_registered BOOLEAN DEFAULT TRUE,
  tds_filer BOOLEAN DEFAULT TRUE,
  status ENUM('ACTIVE', 'SUSPENDED', 'CLOSED') DEFAULT 'ACTIVE',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by_user_id VARCHAR(36),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  UNIQUE KEY uk_gstin (gstin),
  UNIQUE KEY uk_pan_state (pan, state),
  INDEX idx_company_gstin (company_id, gstin)
);

-- User Roles (RBAC)
CREATE TABLE user_roles (
  role_id VARCHAR(36) PRIMARY KEY,
  user_id VARCHAR(36) NOT NULL,
  company_id VARCHAR(36) NOT NULL,
  role VARCHAR(50) NOT NULL,
  permissions JSON NOT NULL,  -- ['gl_post', 'ar_view', ...]
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  FOREIGN KEY (company_id) REFERENCES companies(company_id),
  UNIQUE KEY uk_user_company_role (user_id, company_id, role),
  INDEX idx_company_id (company_id)
);

-- Onboarding Sessions (Temporary)
CREATE TABLE onboarding_sessions (
  session_id VARCHAR(36) PRIMARY KEY,
  user_id VARCHAR(36) NOT NULL,
  company_id VARCHAR(36),
  stage INT NOT NULL,
  status ENUM('IN_PROGRESS', 'COMPLETED', 'EXPIRED') DEFAULT 'IN_PROGRESS',
  data JSON NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  completed_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  FOREIGN KEY (company_id) REFERENCES companies(company_id),
  INDEX idx_session_expires (expires_at)
);

-- OTP Storage (Temporary)
CREATE TABLE otp_store (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id VARCHAR(36) NOT NULL,
  otp_hash VARCHAR(255) NOT NULL,
  method ENUM('EMAIL', 'SMS') DEFAULT 'EMAIL',
  expires_at TIMESTAMP NOT NULL,
  attempts INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  INDEX idx_user_expires (user_id, expires_at)
);

-- User Invites (for team provisioning)
CREATE TABLE user_invites (
  invite_id VARCHAR(36) PRIMARY KEY,
  company_id VARCHAR(36) NOT NULL,
  invited_by_user_id VARCHAR(36) NOT NULL,
  invitee_email VARCHAR(255) NOT NULL,
  full_name VARCHAR(150),
  role VARCHAR(50),
  status ENUM('PENDING', 'ACCEPTED', 'EXPIRED') DEFAULT 'PENDING',
  invite_token VARCHAR(255) NOT NULL UNIQUE,
  expires_at TIMESTAMP NOT NULL,
  accepted_at TIMESTAMP NULL,
  sent_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (company_id) REFERENCES companies(company_id),
  FOREIGN KEY (invited_by_user_id) REFERENCES users(user_id),
  INDEX idx_company_email (company_id, invitee_email),
  INDEX idx_expires (expires_at)
);

-- Chart of Accounts (Per company)
CREATE TABLE chart_of_accounts (
  coa_id VARCHAR(36) PRIMARY KEY,
  company_id VARCHAR(36) NOT NULL,
  account_code VARCHAR(20) NOT NULL,
  account_name VARCHAR(150) NOT NULL,
  account_group ENUM('ASSET', 'LIABILITY', 'EQUITY', 'INCOME', 'EXPENSE'),
  balance DECIMAL(15, 2) DEFAULT 0.00,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (company_id) REFERENCES companies(company_id),
  UNIQUE KEY uk_company_code (company_id, account_code),
  INDEX idx_company_group (company_id, account_group)
);

-- Audit Logs (For compliance)
CREATE TABLE audit_logs (
  log_id VARCHAR(36) PRIMARY KEY,
  user_id VARCHAR(36),
  company_id VARCHAR(36),
  action VARCHAR(100) NOT NULL,
  details JSON,
  ip_address VARCHAR(45),
  user_agent TEXT,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  FOREIGN KEY (company_id) REFERENCES companies(company_id),
  INDEX idx_timestamp (timestamp),
  INDEX idx_company_action (company_id, action)
);
```

---

# PART 5: ERROR HANDLING & EDGE CASES

---

## 5.1 Common Error Scenarios

### Session Expiry During Onboarding

```
Scenario: User in Step 4, session expires (1-hour TTL from Step 1)

Flow:
├─ User fills Step 4 form
├─ Tries to submit → Session check fails
├─ Session expired (> 1 hour since Step 1)
│
├─ Response (401 Unauthorized):
│   {
│     "error_code": "SESSION_EXPIRED",
│     "message": "Your session has expired. Please start over.",
│     "action": "redirect_to_step_1"
│   }
│
└─ Frontend Action:
    ├─ Show error: "Session expired, please start over"
    ├─ Clear session storage
    └─ Redirect to Step 1 (can use same email, skip re-verification)
```

### Email Already Exists (Multiple Signup Attempts)

```
Scenario: User tries to sign up twice with same email

Flow 1 (First attempt):
├─ Sign up with raj@abctrading.com
├─ Complete Step 1-6, company created
│
Flow 2 (User forgets, tries again):
├─ Sign up with raj@abctrading.com (same email)
├─ Step 1 validation:
│   └─ Query: SELECT * FROM users WHERE email = 'raj@abctrading.com'
│   └─ Result: User already exists
│
├─ Response (409 Conflict):
│   {
│     "error_code": "EMAIL_ALREADY_EXISTS",
│     "message": "This email is already registered. Did you want to log in instead?"
│   }
│
└─ Frontend Action:
    ├─ Show error with suggestion
    └─ Provide [Log In] button instead
```

### GSTIN Already Registered

```
Scenario: Two users try to register same GSTIN (shouldn't happen, but safeguard)

Flow:
├─ User A: Step 3, GSTIN = 27AABCT1234H1Z5
├─ User B: Step 3, same GSTIN
│
├─ DB check (Step 3 backend):
│   └─ Query: SELECT * FROM companies WHERE gstin = '27AABCT1234H1Z5'
│   └─ Result: Already exists
│
├─ Response (409 Conflict):
│   {
│     "error_code": "GSTIN_ALREADY_REGISTERED",
│     "message": "This GSTIN is already registered. If this is your company, please log in or contact support.",
│     "contact": "support@niyanthra.io"
│   }
│
└─ Frontend Action:
    ├─ Show error message
    └─ Provide [Login] button or [Contact Support] link
```

### OTP Expired

```
Scenario: User waits > 5 minutes to enter OTP

Flow:
├─ Step 1 submitted, OTP sent
├─ Expires in 5 minutes
├─ User enters OTP after 6 minutes
│
├─ Step 2 OTP verification:
│   ├─ Fetch OTP record
│   ├─ Check: expires_at < NOW()
│   └─ OTP expired
│
├─ Response (400 Bad Request):
│   {
│     "error_code": "OTP_EXPIRED",
│     "message": "Verification code has expired",
│     "action": "show_resend_button"
│   }
│
└─ Frontend Action:
    ├─ Show error: "Code expired, request a new one"
    ├─ Enable [Resend Code] button
    └─ Reset OTP input boxes
```

---

## 5.2 Security Edge Cases

### Prevent Token Reuse

```
Scenario: Attacker tries to use captured JWT multiple times

Flow:
├─ Legitimate user completes onboarding, JWT issued: {exp: 2026-09-20 15:00}
├─ Token used for legitimate login
├─ Attacker obtains token (via network sniff, etc.)
├─ Attacker tries to login with same token at 2026-09-20 15:30
│
├─ Token Validation:
│   ├─ Check signature: Valid ✓
│   ├─ Check expiry: exp (15:00) > now (15:30)? NO ✗
│   └─ Token expired (or in denylist if within 24h)
│
├─ Response (401 Unauthorized):
│   { "error_code": "TOKEN_EXPIRED", "message": "Session expired, please log in again" }
│
└─ Attacker blocked ✓
```

### Prevent Cross-Tenant Data Access

```
Scenario: User from Company A tries to access Company B's data

Flow:
├─ User A JWT: { org: comp_abc123, role: COMPANY_ADMIN }
├─ Tries: GET /api/sales-orders?company_id=comp_xyz456
│
├─ Backend validation:
│   ├─ Extract JWT: org = comp_abc123
│   ├─ Check request: ?company_id = comp_xyz456
│   ├─ Compare: comp_abc123 ≠ comp_xyz456
│   └─ Unauthorized ✗
│
├─ Response (403 Forbidden):
│   { "error_code": "UNAUTHORIZED", "message": "You don't have access to this resource" }
│
├─ Optional: Log suspicious activity
│   ├─ audit_logs: action = 'unauthorized_access_attempt'
│   ├─ Alert if repeated (potential attack)
│   └─ IP address: 203.0.113.99
│
└─ User blocked ✓
```

---

## CONCLUSION

This specification provides **production-ready architectural and technical guidance** for Niyanthra's sign-up, authentication, and onboarding:

1. ✅ **Organization-first model** (mandatory for ERP multi-tenancy)
2. ✅ **6-stage UI/UX flow** (detailed wireframes, form specs, backend triggers)
3. ✅ **Security & isolation** (ACID transactions, role-based access, tenant partitioning)
4. ✅ **Implementation ready** (API endpoints, SQL schema, code examples)
5. ✅ **Error handling** (session expiry, rate limits, edge cases)

**For Engineers:**
- Build APIs from Part 4 specs
- Implement SQL schema from Part 4
- Use atomic transactions (Part 1.2 guarantees)
- Test security edge cases (Part 5)

**For Product:**
- UI flows ready to design (Part 2)
- Error messages pre-written (Part 5)
- Success metrics: <10 min signup, <2% bounce per step

**For Security:**
- Password hashing, OTP, rate limiting (Part 3)
- Audit trail on every action (Part 4)
- Cross-tenant access prevention (Part 5)

**Ready for production soft launch 🚀**

