# 🏢 Niyanthra ERP — System Architecture & Database Schema

> **Simplify. Manage. Grow.**

---

## 📑 Table of Contents

1. [Login / Registration](#1-login--registration)
2. [OAuth / Passkey Authentication](#2-oauth--passkey-authentication)
3. [Organization](#3-organization)
4. [Organization Membership](#4-organization-membership)
5. [Company](#5-company)
6. [Company Types](#6-company-types)
7. [Company Addresses](#7-company-addresses)
8. [Branches](#8-branches)
9. [Branch Locations](#9-branch-locations)
10. [Business Types](#10-business-types)
11. [Industries](#11-industries)
12. [Business Models](#12-business-models)
13. [Business Profile](#13-business-profile)
14. [ERP Modules](#14-erp-modules)
15. [Financial Settings](#15-financial-settings)
16. [Localization Settings](#16-localization-settings)
17. [Tax Configuration](#17-tax-configuration)
18. [Tax Registrations](#18-tax-registrations)
19. [Branch Tax Registration](#19-branch-tax-registration)
20. [Complete Company Structure](#20-complete-company-structure)
21. [Final Relationships](#21-final-relationships)

---
---

## 1. Login / Registration

### 🔐 Normal Authentication Flow

```
┌──────────────────────────────────────────────────────┐
│                        USER                           │
└───────────────────────┬────────────────────────────────┘
                         │ 1. Open ERP
                         ▼
                ┌─────────────────┐
                │ Registration Page│
                └─────────┬────────┘
                          │ 2. Enter details
                          │    • Name
                          │    • Email
                          │    • Password
                          ▼
                  ┌───────────────┐
                  │    Backend     │
                  ├───────────────┤
                  │ • Validate data│
                  │ • Check email  │
                  │   duplication  │
                  │ • Hash password│
                  │ • Create User  │
                  └───────┬────────┘
                          │
                          ▼
                ┌────────────────────┐
                │ Email Verification  │
                │ 3. Send verify link │
                └──────────┬──────────┘
                           ▼
                User clicks verification link
                           │
                           ▼
                   ✅ Email Verified
                           │
                           ▼
                    ┌─────────────┐
                    │    Login     │
                    │  • Email     │
                    │  • Password  │
                    └──────┬───────┘
                           ▼
              Create Session / Access Token
                           │
                           ▼
                  🖥️  ERP Dashboard
```

---

## 2. OAuth / Passkey Authentication

```
                         LOGIN
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       Email / Password           OAuth / Passkey
              │                         │
              └────────────┬────────────┘
                            ▼
                          USER
                            │
                            ▼
                  Authentication OK
                            │
                            ▼
                Load user's memberships
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
          No company             Has companies
                │                       │
                ▼                       ▼
       Setup / Create Business   Company Selection
                                        │
                                        ▼
                                 Select Company
                                        │
                                        ▼
                                 Select Branch
                                        │
                                        ▼
                                Load Role / Access
                                        │
                                        ▼
                                 🖥️  ERP Dashboard
```

---

## 3. Organization

> The **Organization** is the top-level business owner/container.
> The **Company** is the actual business entity operating under the organization.

```
🏛️  Organization ── "ABC Group"  (Type: Business Group)
        │
        ├── 🏢 Company: ABC Paints          → Business Type: Retail
        ├── 🏨 Company: ABC Hotels          → Business Type: Hospitality
        └── 🏭 Company: ABC Manufacturing   → Business Type: Manufacturing
```

### 🗄️ `organizations`

| Field               | Type                | Notes                                |
| ------------------- | ------------------- | ------------------------------------ |
| `id`                | BIGINT PK           | Internal database ID                 |
| `public_id`         | UUID UNIQUE         | Public/API-safe ID                   |
| `name`              | VARCHAR(150)        | Organization name                    |
| `slug`              | VARCHAR(100) UNIQUE | URL/routing identifier               |
| `description`       | TEXT NULL           | Optional description                 |
| `organization_type` | VARCHAR/ENUM        | Organization classification          |
| `status`            | VARCHAR/ENUM        | active, pending, suspended, archived |
| `logo`              | VARCHAR(500) NULL   | Storage path/object key              |
| `website`           | VARCHAR(255) NULL   | Website                              |
| `contact_phone`     | VARCHAR(30) NULL    | Contact number                       |
| `created_by`        | FK → User           | Creator                              |
| `updated_by`        | FK → User           | Last updater                         |
| `created_at`        | TIMESTAMP           | Created time                         |
| `updated_at`        | TIMESTAMP           | Updated time                         |
| `deleted_at`        | TIMESTAMP NULL      | Soft delete                          |

**Organization Types**

| Code               |
| ------------------ |
| `GROUP`             |
| `SINGLE_BUSINESS`   |
| `ENTERPRISE`        |
| `NON_PROFIT`        |
| `FRANCHISE`         |
| `OTHER`             |

---

## 4. Organization Membership

### 🗄️ `organization_memberships`

| Field             | Type           |
| ----------------- | -------------- |
| `id`              | BIGINT PK      |
| `organization_id` | BIGINT FK      |
| `user_id`         | BIGINT FK      |
| `role`            | VARCHAR(30)    |
| `status`          | VARCHAR(30)    |
| `invited_by`      | BIGINT FK NULL |
| `created_at`      | TIMESTAMP      |
| `updated_at`      | TIMESTAMP      |
| `deleted_at`      | TIMESTAMP NULL |

**Status values:** `pending` · `active` · `suspended` · `revoked`

**Organization Roles:** `owner` · `partner` · `admin` · `member` · `employee`

> ⚠️ **Note:** Organization Role ≠ Company Role — these are two separate permission systems.

---

## 5. Company

> If an organization is registered, the user can proceed to company registration.

### 🗄️ `companies`

| Field                  | Type               | Notes                      |
| ---------------------- | ------------------ | -------------------------- |
| `id`                   | BIGINT PK          | Internal ID                |
| `public_id`            | UUID UNIQUE        | Public/API ID               |
| `organization_id`      | BIGINT FK          | Parent organization        |
| `name`                 | VARCHAR(150)       | Display/company name       |
| `code`                 | VARCHAR(50)        | Unique company code        |
| `registration_number`  | VARCHAR(100) NULL  | Legal registration number  |
| `company_type_id`      | BIGINT FK          | Company/legal type         |
| `status`               | VARCHAR(30)        | active/suspended/archived  |
| `description`          | TEXT NULL          | Optional                   |
| `logo`                 | VARCHAR(500) NULL  | Storage/object key         |
| `website`              | VARCHAR(255) NULL  | Website                    |
| `timezone`             | VARCHAR(100)       | Default timezone           |
| `date_format`          | VARCHAR(30)        | Default date format        |
| `created_by`           | BIGINT FK          | Creator                    |
| `updated_by`           | BIGINT FK NULL     | Last updater                |
| `created_at`           | TIMESTAMP          | Created                    |
| `updated_at`           | TIMESTAMP          | Updated                    |
| `deleted_at`           | TIMESTAMP NULL     | Soft delete                |

---

## 6. Company Types

### 🗄️ `company_types`

| Field         | Type               |
| ------------- | ------------------ |
| `id`          | BIGINT PK          |
| `public_id`   | UUID UNIQUE        |
| `name`        | VARCHAR(100)       |
| `code`        | VARCHAR(50) UNIQUE |
| `description` | TEXT NULL          |
| `status`      | VARCHAR(30)        |
| `created_at`  | TIMESTAMP          |
| `updated_at`  | TIMESTAMP          |

**Examples:** `private_company` · `public_company` · `partnership` · `sole_proprietorship` · `llc` · `non_profit` · `government` · `other`

---

## 7. Company Addresses

### 🗄️ `company_addresses`

| Field             | Type               |
| ----------------- | ------------------ |
| `id`              | BIGINT PK          |
| `public_id`       | UUID UNIQUE        |
| `company_id`      | BIGINT FK          |
| `address_type`    | VARCHAR(30)        |
| `address_line_1`  | VARCHAR(255)       |
| `address_line_2`  | VARCHAR(255) NULL  |
| `city`            | VARCHAR(100)       |
| `state_province`  | VARCHAR(100)       |
| `postal_code`     | VARCHAR(20)        |
| `country_id`      | BIGINT FK          |
| `is_primary`      | BOOLEAN            |
| `created_at`      | TIMESTAMP          |
| `updated_at`      | TIMESTAMP          |
| `deleted_at`      | TIMESTAMP NULL     |

**Example**

```
🏢 ABC Company
  ├── 📍 Registered Address
  ├── 📍 Head Office Address
  └── 📍 Billing Address
```

---

## 8. Branches

### 🗄️ `branches`

| Field          | Type               |
| -------------- | ------------------ |
| `id`           | BIGINT PK          |
| `public_id`    | UUID UNIQUE        |
| `company_id`   | BIGINT FK          |
| `name`         | VARCHAR(150)       |
| `code`         | VARCHAR(50)        |
| `branch_type`  | VARCHAR(50) NULL   |
| `status`       | VARCHAR(30)        |
| `email`        | VARCHAR(255) NULL  |
| `phone`        | VARCHAR(30) NULL   |
| `manager_id`   | BIGINT FK NULL     |
| `timezone`     | VARCHAR(100) NULL  |
| `created_by`   | BIGINT FK          |
| `updated_by`   | BIGINT FK NULL     |
| `created_at`   | TIMESTAMP          |
| `updated_at`   | TIMESTAMP          |
| `deleted_at`   | TIMESTAMP NULL     |

---

## 9. Branch Locations

### 🗄️ `branch_locations`

| Field             | Type                |
| ----------------- | ------------------- |
| `id`              | BIGINT PK           |
| `branch_id`       | BIGINT FK UNIQUE    |
| `address_line_1`  | VARCHAR(255)        |
| `address_line_2`  | VARCHAR(255) NULL   |
| `city`            | VARCHAR(100)        |
| `state_province`  | VARCHAR(100)        |
| `postal_code`     | VARCHAR(20)         |
| `country_id`      | BIGINT FK           |
| `latitude`        | DECIMAL(10,7) NULL  |
| `longitude`       | DECIMAL(10,7) NULL  |
| `created_at`      | TIMESTAMP           |
| `updated_at`      | TIMESTAMP           |

```
🏢 Company
  └── 🏬 Branch
        └── 📍 Location
```

---

## 10. Business Types

### 🗄️ `business_types`

| Field         | Type               |
| ------------- | ------------------ |
| `id`          | BIGINT PK          |
| `public_id`   | UUID UNIQUE        |
| `name`        | VARCHAR(100)       |
| `code`        | VARCHAR(50) UNIQUE |
| `description` | TEXT NULL          |
| `status`      | VARCHAR(30)        |
| `created_at`  | TIMESTAMP          |
| `updated_at`  | TIMESTAMP          |

**Examples:** Retail · Wholesale · Distribution · Manufacturing · Services · Construction · Hospitality · Healthcare · Education · E-Commerce

### 🗄️ `company_business_types`

| Field               | Type      |
| ------------------- | --------- |
| `id`                | BIGINT PK |
| `company_id`        | BIGINT FK |
| `business_type_id`  | BIGINT FK |
| `is_primary`        | BOOLEAN   |
| `created_at`        | TIMESTAMP |
| `updated_at`        | TIMESTAMP |

**Example**

```
🏢 ABC Paints
  ├── ⭐ Retail          (Primary)
  ├── ── Wholesale
  └── ── Distribution
```

---

## 11. Industries

### 🗄️ `industries`

| Field         | Type               |
| ------------- | ------------------ |
| `id`          | BIGINT PK          |
| `public_id`   | UUID UNIQUE        |
| `name`        | VARCHAR(100)       |
| `code`        | VARCHAR(50) UNIQUE |
| `parent_id`   | BIGINT FK NULL     |
| `description` | TEXT NULL          |
| `status`      | VARCHAR(30)        |
| `created_at`  | TIMESTAMP          |
| `updated_at`  | TIMESTAMP          |

**Example (hierarchical)**

```
🍽️ Food & Beverage
  ├── 🍞 Bakery
  ├── 🍴 Restaurant
  └── 🎪 Catering
```

### 🗄️ `company_industries`

| Field          | Type      |
| -------------- | --------- |
| `id`           | BIGINT PK |
| `company_id`   | BIGINT FK |
| `industry_id`  | BIGINT FK |
| `is_primary`   | BOOLEAN   |
| `created_at`   | TIMESTAMP |
| `updated_at`   | TIMESTAMP |

---

## 12. Business Models

### 🗄️ `business_models`

| Field         | Type               |
| ------------- | ------------------ |
| `id`          | BIGINT PK          |
| `public_id`   | UUID UNIQUE        |
| `name`        | VARCHAR(100)       |
| `code`        | VARCHAR(50) UNIQUE |
| `description` | TEXT NULL          |
| `status`      | VARCHAR(30)        |
| `created_at`  | TIMESTAMP          |
| `updated_at`  | TIMESTAMP          |

**Examples:** B2B · B2C · B2B2C · Marketplace · Subscription · Project-Based · Distribution · Hybrid

### 🗄️ `company_business_models`

| Field                | Type      |
| --------------------- | --------- |
| `id`                  | BIGINT PK |
| `company_id`          | BIGINT FK |
| `business_model_id`   | BIGINT FK |
| `is_primary`          | BOOLEAN   |
| `created_at`          | TIMESTAMP |
| `updated_at`          | TIMESTAMP |

---

## 13. Business Profile

### 🗄️ `business_profiles`

| Field                | Type              |
| --------------------- | ----------------- |
| `id`                  | BIGINT PK         |
| `public_id`           | UUID UNIQUE       |
| `company_id`          | BIGINT FK UNIQUE  |
| `operational_model`   | VARCHAR(50)       |
| `description`         | TEXT NULL         |
| `status`              | VARCHAR(30)       |
| `created_at`          | TIMESTAMP         |
| `updated_at`          | TIMESTAMP         |

```
🏢 Company
  └── 📋 Business Profile
        ├── Business Types
        ├── Industries
        ├── Business Models
        └── Operational Model
```

---

## 14. ERP Modules

### 🗄️ `modules`

| Field         | Type               |
| ------------- | ------------------ |
| `id`          | BIGINT PK          |
| `public_id`   | UUID UNIQUE        |
| `name`        | VARCHAR(100)       |
| `code`        | VARCHAR(50) UNIQUE |
| `description` | TEXT NULL          |
| `status`      | VARCHAR(30)        |
| `created_at`  | TIMESTAMP          |
| `updated_at`  | TIMESTAMP          |

**Examples:** sales · purchase · inventory · accounting · pos · hr · payroll · manufacturing · crm · projects

### 🗄️ `company_modules`

| Field          | Type            |
| -------------- | --------------- |
| `id`           | BIGINT PK       |
| `company_id`   | BIGINT FK       |
| `module_id`    | BIGINT FK       |
| `status`       | VARCHAR(30)     |
| `enabled_at`   | TIMESTAMP NULL  |
| `disabled_at`  | TIMESTAMP NULL  |
| `enabled_by`   | BIGINT FK NULL  |
| `created_at`   | TIMESTAMP       |
| `updated_at`   | TIMESTAMP       |

---

## 15. Financial Settings

### 🗄️ `company_financial_settings`

| Field                | Type              |
| --------------------- | ----------------- |
| `id`                  | BIGINT PK         |
| `company_id`          | BIGINT FK UNIQUE  |
| `base_currency_id`    | BIGINT FK         |
| `fiscal_year_start`   | DATE              |
| `fiscal_year_end`     | DATE              |
| `accounting_method`   | VARCHAR(30)       |
| `decimal_precision`   | SMALLINT          |
| `rounding_method`     | VARCHAR(30)       |
| `created_at`          | TIMESTAMP         |
| `updated_at`          | TIMESTAMP         |

---

## 16. Localization Settings

### 🗄️ `company_localization_settings`

| Field                | Type              |
| --------------------- | ----------------- |
| `id`                  | BIGINT PK         |
| `company_id`          | BIGINT FK UNIQUE  |
| `country_id`          | BIGINT FK         |
| `language`            | VARCHAR(20)       |
| `locale`              | VARCHAR(20)       |
| `timezone`            | VARCHAR(100)      |
| `date_format`         | VARCHAR(30)       |
| `time_format`         | VARCHAR(30)       |
| `number_format`       | VARCHAR(30)       |
| `first_day_of_week`   | SMALLINT          |
| `created_at`          | TIMESTAMP         |
| `updated_at`          | TIMESTAMP         |

---

## 17. Tax Configuration

> 🆕 **This is the new section added to the existing structure.**

The important conceptual separation:

| Concept              | Meaning                                                   |
| --------------------- | ---------------------------------------------------------- |
| **Tax Registration**  | Which legal tax registration does the company hold?        |
| **Tax Configuration** | How should Niyanthra calculate / apply tax on transactions? |

---

### 17.1 Tax Types

#### 🗄️ `tax_types`

| Field         | Type               |
| ------------- | ------------------ |
| `id`          | BIGINT PK          |
| `public_id`   | UUID UNIQUE        |
| `name`        | VARCHAR(100)       |
| `code`        | VARCHAR(50) UNIQUE |
| `description` | TEXT NULL          |
| `status`      | VARCHAR(30)        |
| `created_at`  | TIMESTAMP          |
| `updated_at`  | TIMESTAMP          |

**Examples:** `GST` · `VAT` · `TDS` · `TCS` · `CESS` · `OTHER`

---

### 17.2 Tax Components

#### 🗄️ `tax_components`

| Field             | Type               |
| ------------------ | ------------------ |
| `id`               | BIGINT PK          |
| `public_id`        | UUID UNIQUE        |
| `tax_type_id`      | BIGINT FK          |
| `name`             | VARCHAR(100)       |
| `code`             | VARCHAR(50) UNIQUE |
| `component_type`   | VARCHAR(50)        |
| `description`      | TEXT NULL          |
| `status`           | VARCHAR(30)        |
| `created_at`        | TIMESTAMP          |
| `updated_at`        | TIMESTAMP          |

**Example — GST Components**

```
💰 GST
  ├── CGST    (Central)
  ├── SGST    (State)
  ├── IGST    (Integrated / Inter-state)
  ├── UTGST   (Union Territory)
  └── CESS    (Additional levy)
```

---

### 17.3 Tax Rates

#### 🗄️ `tax_rates`

| Field             | Type               |
| ------------------ | ------------------ |
| `id`               | BIGINT PK          |
| `public_id`        | UUID UNIQUE        |
| `tax_type_id`      | BIGINT FK          |
| `name`             | VARCHAR(100)       |
| `code`             | VARCHAR(50) UNIQUE |
| `rate`             | DECIMAL(8,4)       |
| `rate_type`        | VARCHAR(30)        |
| `effective_from`   | DATE               |
| `effective_to`     | DATE NULL          |
| `status`           | VARCHAR(30)        |
| `created_at`        | TIMESTAMP          |
| `updated_at`        | TIMESTAMP          |

**Examples**

| Code       | Rate |
| ---------- | ---- |
| `GST_5`    | 5%   |
| `GST_12`   | 12%  |
| `GST_18`   | 18%  |
| `GST_28`   | 28%  |

---

### 17.4 Tax Rate Components

> Links a tax rate to its individual components (the "recipe" behind each rate).

#### 🗄️ `tax_rate_components`

| Field                | Type         |
| --------------------- | ------------ |
| `id`                  | BIGINT PK    |
| `tax_rate_id`         | BIGINT FK    |
| `tax_component_id`    | BIGINT FK    |
| `rate`                | DECIMAL(8,4) |
| `created_at`          | TIMESTAMP    |
| `updated_at`          | TIMESTAMP    |

**Example — Intra-state transaction**

```
💰 GST 18%
  ├── CGST  9%
  └── SGST  9%
```

**Example — Inter-state transaction**

```
💰 IGST 18%
  └── IGST 18%
```

---

### 17.5 Tax Rules

> Tax Rules determine **which tax configuration applies to a given transaction**, based on origin, destination, and transaction type.

#### 🗄️ `tax_rules`

| Field                          | Type            |
| ------------------------------- | --------------- |
| `id`                            | BIGINT PK       |
| `public_id`                     | UUID UNIQUE     |
| `company_id`                    | BIGINT FK       |
| `tax_type_id`                   | BIGINT FK       |
| `name`                          | VARCHAR(150)    |
| `code`                          | VARCHAR(50)     |
| `source_jurisdiction_id`        | BIGINT FK NULL  |
| `destination_jurisdiction_id`   | BIGINT FK NULL  |
| `tax_rate_id`                   | BIGINT FK       |
| `transaction_type`              | VARCHAR(50)     |
| `status`                        | VARCHAR(30)     |
| `effective_from`                | DATE            |
| `effective_to`                  | DATE NULL       |
| `created_by`                    | BIGINT FK       |
| `updated_by`                    | BIGINT FK NULL  |
| `created_at`                    | TIMESTAMP       |
| `updated_at`                    | TIMESTAMP       |

**Example — Rule Resolution**

```
📍 Kerala  →  📍 Kerala        (same state)
                  ↓
             CGST + SGST

📍 Kerala  →  📍 Karnataka     (different states)
                  ↓
                IGST
```

---

## 18. Tax Registrations

> Existing registration tables — unchanged from the current structure.

### 🗄️ `company_tax_registrations`

| Field                  | Type                |
| ----------------------- | ------------------- |
| `id`                    | BIGINT PK           |
| `public_id`             | UUID UNIQUE         |
| `company_id`            | BIGINT FK           |
| `country_id`            | BIGINT FK           |
| `jurisdiction_id`       | BIGINT FK           |
| `tax_authority_id`      | BIGINT FK           |
| `tax_type_id`           | BIGINT FK           |
| `registration_number`   | VARCHAR(100)        |
| `registration_name`     | VARCHAR(200) NULL   |
| `status`                | VARCHAR(30)         |
| `effective_from`        | DATE                |
| `effective_to`          | DATE NULL           |
| `is_primary`            | BOOLEAN             |
| `created_by`            | BIGINT FK           |
| `updated_by`            | BIGINT FK NULL      |
| `created_at`             | TIMESTAMP           |
| `updated_at`             | TIMESTAMP           |
| `deleted_at`             | TIMESTAMP NULL      |

---

## 19. Branch Tax Registration

### 🗄️ `branch_tax_registrations`

| Field                   | Type      |
| ------------------------ | --------- |
| `id`                     | BIGINT PK |
| `branch_id`              | BIGINT FK |
| `tax_registration_id`    | BIGINT FK |
| `is_primary`             | BOOLEAN   |
| `effective_from`         | DATE      |
| `effective_to`           | DATE NULL |
| `created_at`             | TIMESTAMP |
| `updated_at`             | TIMESTAMP |

**Example**

```
🏢 ABC Company
  │
  ├── 🧾 Kerala GST Registration
  │     └── 🏬 Thrissur Branch
  │
  └── 🧾 Karnataka GST Registration
        └── 🏬 Bangalore Branch
```

---

## 20. Complete Company Structure

```
🏛️  Organization
   │
   └── 🏢 Company
         │
         ├── 🏷️  Company Type
         │
         ├── 📍 Company Addresses
         │
         ├── 🏬 Branches
         │     └── 📍 Branch Location
         │
         ├── 📋 Business Profile
         │     ├── Business Types
         │     ├── Industries
         │     └── Business Models
         │
         ├── 🧩 ERP Modules
         │
         ├── 💵 Financial Settings
         │
         ├── 🌐 Localization Settings
         │
         └── 🧾 Tax
               │
               ├── Tax Registrations
               │     └── Branch Tax Registration
               │
               └── Tax Configuration
                     ├── Tax Types
                     ├── Tax Components
                     ├── Tax Rates
                     ├── Tax Rate Components
                     └── Tax Rules
```

---

## 21. Final Relationships

| Relationship                          | Cardinality     |
| -------------------------------------- | --------------- |
| Organization → Companies               | One-to-Many     |
| Company → Branches                     | One-to-Many     |
| Branch → Location                      | One-to-One      |
| Company → Company Type                 | Many-to-One     |
| Company → Business Types               | Many-to-Many    |
| Company → Industries                   | Many-to-Many    |
| Company → Business Models              | Many-to-Many    |
| Company → Business Profile             | One-to-One      |
| Company → Modules                      | Many-to-Many    |
| Company → Addresses                    | One-to-Many     |
| Company → Financial Settings           | One-to-One      |
| Company → Localization                 | One-to-One      |
| Company → Tax Registrations            | One-to-Many     |
| Branch → Tax Registrations             | Many-to-Many    |
| Tax Type → Tax Components              | One-to-Many     |
| Tax Type → Tax Rates                   | One-to-Many     |
| Tax Rate → Tax Components              | Many-to-Many    |
| Tax Rule → Tax Rate                    | Many-to-One     |
| Company → Tax Rules                    | One-to-Many     |

### 🧭 Final Tax Architecture

```
                              🏢 COMPANY
                                  │
                  ┌───────────────┴───────────────┐
                  ▼                                ▼
         🧾 TAX REGISTRATION               ⚙️ TAX CONFIGURATION
                  │                                │
                  │                    ┌────────────┴────────────┐
                  │                    ▼                         ▼
                  │               Tax Types                 Tax Rules
                  │                    │                         │
                  │                    ▼                         ▼
                  │             Tax Components              Tax Rates
                  │                                              │
                  │                                              ▼
                  │                                    Tax Rate Components
                  │
                  ▼
               🏬 BRANCH
                  │
                  ▼
              Tax Context
                  │
                  ▼
         🧾 Invoice / Transaction
```

> ✅ This structure preserves the **existing organization/company hierarchy unchanged**, while layering in a complete **Tax Registration + Tax Configuration engine**.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
