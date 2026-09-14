# 🔐 Niyanthra ERP — Roles, Groups & Permissions Architecture

> **Simplify. Manage. Grow.**

---

## 📑 Table of Contents

1. [User Membership Overview](#1-user-membership-overview)
2. [Access Example](#2-access-example)
3. [Why Not Put Permissions Directly on the User](#3-why-not-put-permissions-directly-on-the-user)
4. [The Permission Hierarchy](#4-the-permission-hierarchy)
5. [Permissions → Roles → Groups Example](#5-permissions--roles--groups-example)
6. [Database Schema — `user_group_assignments`](#6-database-schema--user_group_assignments)
7. [Assignment Examples](#7-assignment-examples)
8. [Complete Access Resolution Flow](#8-complete-access-resolution-flow)

---
---

## 1. User Membership Overview

Every user can belong to **multiple organizations and companies**, and within each company, to **multiple branches** — each with its own role.

```
👤 USER
   │
   ├── 🏛️  Organization Membership
   │
   └── 🏢 Company Membership
         │
         ├── 🏢 Company A
         │      │
         │      ├── 🏬 Branch Kerala
         │      │      └── 🏷️  Role: Manager
         │      │            └── 🔑 Permissions
         │      │
         │      └── 🏬 Branch Bangalore
         │             └── 🏷️  Role: Staff
         │                  └── 🔑 Permissions
         │
         └── 🏢 Company B
                │
                └── 🏬 Branch Kochi
                       └── 🏷️  Role: Accountant
                            └── 🔑 Permissions
```

> 💡 A single user can hold **different roles in different branches of different companies** — access is always scoped to `company + branch`.

---

## 2. Access Example

| Company      | Branch      | Role        | Access                        |
| ------------ | ----------- | ----------- | ------------------------------ |
| ABC Paints   | Thrissur    | Manager     | Sales + Inventory + Reports    |
| ABC Paints   | Bangalore   | Staff       | Sales only                     |
| ABC Hotels   | Kochi       | Accountant  | Accounting + Reports           |

---

## 3. Why Not Put Permissions Directly on the User

### ❌ Anti-Pattern — Avoid This

```
👤 User
  ├── is_admin
  ├── can_create_invoice
  ├── can_delete_product
  └── can_view_reports
```

> ⚠️ **Problem:** This breaks down as soon as a user belongs to **multiple companies** — a flat set of booleans can't represent "Manager in Company A, Staff in Company B."

### ✅ Correct Pattern — Layered Access Model

```
🔑 Permission
     │
     │ permissions assigned to
     ▼
🏷️  Role
     │
     │ roles assigned to
     ▼
👥 Permission Group
     │
     │ group assigned to
     ▼
👤 User
     │
     ├── 🏢 Company
     │
     └── 🏬 Branch
```

---

## 4. The Permission Hierarchy

```
Permissions ──▶ Roles ──▶ Groups ──▶ Users
```

| Layer          | Purpose                                              |
| -------------- | ----------------------------------------------------- |
| **Permission** | The smallest unit of access (e.g. `invoice.create`)   |
| **Role**       | A bundle of permissions for a job function             |
| **Group**      | A bundle of roles for a team or responsibility area     |
| **User**       | Assigned to one or more groups, scoped by company/branch |

---

## 5. Permissions → Roles → Groups Example

### 🔑 Permissions

```
Permissions
  ├── sales.invoice.view
  ├── sales.invoice.create
  ├── sales.invoice.update
  ├── sales.invoice.delete
  ├── inventory.product.view
  ├── inventory.product.create
  └── inventory.stock.adjust
```

⬇

### 🏷️ Roles

```
Roles
  ├── Sales Staff
  │     ├── invoice.view
  │     └── invoice.create
  │
  ├── Sales Manager
  │     ├── invoice.view
  │     ├── invoice.create
  │     ├── invoice.update
  │     └── invoice.delete
  │
  └── Inventory Manager
        ├── product.view
        ├── product.create
        └── stock.adjust
```

⬇

### 👥 Groups

```
Groups
  ├── Sales Team
  │     ├── Sales Staff
  │     └── Sales Manager
  │
  ├── Inventory Team
  │     └── Inventory Manager
  │
  └── Branch Manager
        ├── Sales Manager
        └── Inventory Manager
```

⬇

### 👤 Users

```
Users
  └── John
        └── Branch Manager
```

---

## 6. Database Schema — `user_group_assignments`

| Field          | Type            | Description      |
| -------------- | --------------- | ----------------- |
| `id`           | BIGINT PK       | Internal ID        |
| `public_id`    | UUID UNIQUE     | Public ID           |
| `user_id`      | BIGINT FK       | User               |
| `group_id`     | BIGINT FK       | Group              |
| `company_id`   | BIGINT FK       | Company            |
| `branch_id`    | BIGINT FK NULL  | Branch              |
| `status`       | VARCHAR(30)     | active/inactive    |
| `assigned_by`  | BIGINT FK       | Assigned by         |
| `created_at`   | TIMESTAMP       | Created date        |
| `updated_at`   | TIMESTAMP       | Updated date        |
| `deleted_at`   | TIMESTAMP NULL  | Soft delete         |

---

## 7. Assignment Examples

### 🏢 Company-Level Assignment

```
👤 John
   │
   └── 🏢 Company: ABC Paints
          │
          └── 👥 Group: Accounts Team
```

> No `branch_id` set → access applies **company-wide**.

### 🏬 Branch-Level Assignment

```
👤 John
   │
   └── 🏢 Company: ABC Paints
          │
          └── 🏬 Branch: Thrissur
                 │
                 └── 👥 Group: Branch Manager
```

> `branch_id` set → access is **scoped to that specific branch only**.

---

## 8. Complete Access Resolution Flow

```
                              👤 USER
                                 │
                                 ▼
                       COMPANY MEMBERSHIP
                                 │
                  ┌──────────────┴──────────────┐
                  ▼                              ▼
          COMPANY POSITION                 COMPANY / BRANCH
          "Who are they?"                  "Where do they work?"
                                                    │
                                                    ▼
                                          GROUP ASSIGNMENT
                                                    │
                                                    ▼
                                           PERMISSION GROUP
                                                    │
                                                    ▼
                                             ACCESS ROLES
                                                    │
                                                    ▼
                                              PERMISSIONS
```

> ✅ This model cleanly separates **identity** (who the user is), **scope** (company/branch), and **access** (roles → permissions) — allowing one user to hold different, independently-scoped access levels across every company and branch they belong to.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
