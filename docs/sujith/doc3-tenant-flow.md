# Niyanthra — Organization, Company & Tenant Architecture

> **Niyanthra**
> **Simplify. Manage. Grow.**

---

## 1. Overview

Niyanthra uses a **Company-based multi-tenant architecture**.

The core structure is:

```text
Organization
    │
    ├── Company A
    │      └── Tenant A
    │             ├── Branch 1
    │             ├── Branch 2
    │             └── Branch 3
    │
    ├── Company B
    │      └── Tenant B
    │             ├── Branch 1
    │             └── Branch 2
    │
    └── Company C
           └── Tenant C
                  └── Branch 1
```

The important principle is:

> **One Company gets one Tenant by default.**

However, **Company and Tenant are separate concepts**.

```text
Company ≠ Tenant
```

A company represents the actual business entity, while a tenant represents the **data-isolation boundary** used by the Niyanthra platform.

---

# 2. Core Concepts

| Concept            | Purpose                                                    |
| ------------------ | ---------------------------------------------------------- |
| **Organization**   | Top-level business owner, group, or business container     |
| **Company**        | Actual legal or operational business entity                |
| **Tenant**         | Data-isolation boundary for a company                      |
| **Branch**         | Operational location belonging to a company                |
| **Tenant Storage** | Database/storage where company business data is maintained |

### Relationship

```text
Organization
      │
      ▼
   Company
      │
      ├──────────────┐
      ▼              ▼
    Tenant         Branches
      │              │
      │              ├── Branch 1
      │              ├── Branch 2
      │              └── Branch 3
      │
      ▼
 Business Data
```

---

# 3. Organization

An **Organization** is the highest-level business owner/container in Niyanthra.

An organization may contain one or more companies.

Example:

```text
ABC Group
│
├── ABC Paints
├── ABC Hotels
└── ABC Manufacturing
```

The organization itself does not represent the operational business data.

It provides a common ownership and management boundary for companies.

---

# 4. Company

A **Company** represents the actual business entity.

Examples:

```text
ABC Paints
ABC Hotels
ABC Manufacturing
```

A company owns its operational and business data, including:

* Products
* Customers
* Suppliers
* Sales
* Purchases
* Inventory
* Employees
* Payroll
* Accounting
* Tax transactions
* Reports
* Other enabled ERP modules

Therefore, the company is the natural default boundary for tenant isolation.

---

# 5. Tenant

A **Tenant** represents the data-isolation boundary of a company.

By default:

```text
One Company
      │
      ▼
One Tenant
```

Example:

```text
ABC Paints
    │
    ▼
Tenant: abc-paints
```

The tenant determines where the company's business data is stored and how that data is isolated from other companies.

---

# 6. Company and Tenant Relationship

The recommended relationship is:

```text
Organization
     │
     ├───────────────┐
     ▼               ▼
  Company           Tenant
     │               │
     └──── tenant_id ┘
```

Database relationship:

```text
companies.tenant_id → tenants.id
```

Therefore:

```text
Company 1 ────── 1 Tenant
```

This is the normal relationship.

However, they should remain separate entities because the tenant represents the infrastructure/data-isolation layer, while the company represents the business layer.

---

# 7. Why Company-Based Tenancy?

Company-based tenancy provides a natural separation of business data.

For example:

```text
ABC Group
│
├── ABC Paints
│     └── Tenant A
│
├── ABC Hotels
│     └── Tenant B
│
└── ABC Manufacturing
      └── Tenant C
```

Each company can have completely different:

* Products
* Customers
* Employees
* Inventory
* Accounting
* Tax configuration
* Business workflows
* ERP modules
* Branches

This prevents unrelated businesses under the same organization from unnecessarily sharing the same business-data boundary.

---

# 8. Why Organization Should Not Normally Be the Tenant

An alternative design would be:

```text
Organization
└── Tenant
    ├── Company A
    ├── Company B
    └── Company C
```

This can create a problem when companies have significantly different business operations.

For example:

```text
ABC Group
├── ABC Paints
├── ABC Hotels
└── ABC Manufacturing
```

These companies may have completely different:

```text
Products
Inventory
Accounting
Employees
Tax
Workflows
Modules
Business Rules
```

Putting all of this inside a single tenant can make data isolation and tenant-specific configuration more complicated.

Therefore, Niyanthra uses:

```text
Organization
    ↓
Company
    ↓
Tenant
```

instead of:

```text
Organization
    ↓
Tenant
    ↓
Multiple Companies
```

as the default architecture.

---

# 9. Branches

Branches belong to a company and are **not separate tenants by default**.

Example:

```text
ABC Paints
│
├── Tenant: abc-paints
│
├── Thrissur Branch
├── Kochi Branch
└── Bangalore Branch
```

All branches operate within the same company tenant.

```text
Tenant: ABC Paints
│
├── Thrissur
├── Kochi
└── Bangalore
```

This means the company can maintain shared business data while still applying branch-specific access, configuration, tax registration, inventory, and operational rules where required.

---

# 10. Why Branches Should Not Normally Be Tenants

Avoid:

```text
ABC Paints
├── Tenant Thrissur
├── Tenant Kochi
└── Tenant Bangalore
```

This creates unnecessary complexity.

For example, common company-level data could become duplicated or require cross-tenant synchronization:

```text
Products
Customers
Suppliers
Accounting
Employees
Reports
```

Instead:

```text
ABC Paints
└── Tenant
     ├── Thrissur Branch
     ├── Kochi Branch
     └── Bangalore Branch
```

The tenant provides the company-level data boundary, while branches provide the operational boundary.

---

# 11. Hybrid Multi-Tenancy

Niyanthra supports different infrastructure strategies through `tenant_type`.

```text
Company
   │
   ▼
Tenant
   │
   ├── SHARED
   │     └── Shared Database + tenant isolation
   │
   ├── DEDICATED
   │     └── Separate Database
   │
   └── SILO
         └── Separate Application
             + Database
             + Redis
             + Infrastructure
```

This allows Niyanthra to support small, medium, and enterprise customers without changing the business model.

---

# 12. Shared Tenant

In a shared model, multiple tenants use a common database infrastructure.

```text
Shared Database
│
├── Tenant A
├── Tenant B
├── Tenant C
└── Tenant D
```

Tenant isolation can be implemented using a tenant identifier and PostgreSQL Row-Level Security where appropriate.

Example:

```text
Tenant A
tenant_id = 101

Tenant B
tenant_id = 102
```

Business records are associated with the appropriate tenant.

```text
products
--------------------------------
id | tenant_id | name
--------------------------------
1  | 101       | Red Paint
2  | 101       | Blue Paint
3  | 102       | Hotel Room
```

Tenant A must never be able to access Tenant B's records.

---

# 13. Dedicated Tenant

A dedicated tenant has its own database.

```text
Tenant A
   │
   ▼
tenant_abc_paints
```

Another company:

```text
Tenant B
   │
   ▼
tenant_abc_hotels
```

Example:

```text
ABC Paints
    ↓
Tenant A
    ↓
Dedicated Database

ABC Hotels
    ↓
Tenant B
    ↓
Dedicated Database
```

This provides stronger physical database separation while still using the same application architecture.

---

# 14. Silo Tenant

A silo tenant receives stronger infrastructure isolation.

```text
Tenant
│
├── Application
├── Database
├── Redis
├── Celery
└── Infrastructure
```

This should normally be reserved for customers with special requirements such as:

* Strong isolation requirements
* Enterprise requirements
* Regulatory requirements
* Large-scale customers
* Customer-specific infrastructure requirements

---

# 15. Tenant Type Can Change

A major benefit of keeping `Company` and `Tenant` separate is that the infrastructure strategy can change without changing the company's identity.

For example:

```text
ABC Paints
    │
    ▼
Tenant
    │
    ▼
SHARED
```

Later:

```text
ABC Paints
    │
    ▼
Same Tenant
    │
    ▼
Change Tenant Type
    │
    ▼
DEDICATED
```

The tenant identity remains the same.

Only the underlying infrastructure changes.

```text
Company
   │
   ▼
Same Tenant
   │
   ├── Before → Shared
   │
   └── Later  → Dedicated
```

This makes the architecture easier to evolve.

---

# 16. Master Database

Niyanthra uses a **Master Database** to manage platform-level information.

The Master DB does not store all business transactions.

It primarily stores:

```text
Identity
Organization
Company
Branch
Tenant
Access Control
Platform Configuration
Routing Information
```

---

# 17. Master Database Structure

```text
MASTER DATABASE
│
├── AUTHENTICATION
│   ├── users
│   ├── authentication_methods
│   └── sessions
│
├── ORGANIZATION
│   ├── organizations
│   └── organization_memberships
│
├── TENANCY
│   ├── tenants
│   └── tenant_resources
│
├── COMPANY
│   ├── companies
│   ├── company_types
│   ├── company_addresses
│   ├── company_memberships
│   ├── branches
│   └── branch_locations
│
├── BUSINESS CONFIGURATION
│   ├── business_types
│   ├── company_business_types
│   ├── industries
│   ├── company_industries
│   ├── business_models
│   ├── company_business_models
│   └── business_profiles
│
├── ERP CONFIGURATION
│   ├── modules
│   ├── company_modules
│   ├── company_financial_settings
│   └── company_localization_settings
│
├── ACCESS CONTROL
│   ├── membership_positions
│   ├── permissions
│   ├── access_roles
│   ├── access_role_permissions
│   ├── permission_groups
│   ├── permission_group_roles
│   └── user_group_assignments
│
└── TAX CONFIGURATION
    ├── tax_types
    ├── tax_components
    ├── tax_rates
    ├── tax_rate_components
    ├── tax_rules
    ├── company_tax_registrations
    └── branch_tax_registrations
```

---

# 18. Tenant Database

The tenant database stores actual business/operational data.

Example:

```text
TENANT DATABASE
│
├── products
├── product_categories
│
├── customers
├── customer_addresses
│
├── suppliers
│
├── sales
├── sales_items
├── invoices
├── invoice_items
│
├── purchases
├── purchase_items
│
├── inventory
├── stock
├── warehouses
│
├── employees
├── payroll
│
├── accounting
├── payments
│
└── reports / transactional data
```

The exact tables can vary depending on the enabled ERP modules.

---

# 19. Tenant Creation

A tenant should normally be created as part of the company onboarding process.

The recommended flow is:

```text
User Registration
       │
       ▼
Create User
       │
       ▼
Email Verification
       │
       ▼
Create Organization
       │
       ▼
Create Organization Membership
       │
       ▼
Create Company
       │
       ▼
Create Tenant
       │
       ▼
Link Company → Tenant
       │
       ▼
Provision Tenant Resources
       │
       ▼
Create Default Branch
       │
       ▼
Configure Business Profile
       │
       ▼
Configure ERP Modules
       │
       ▼
Configure Financial Settings
       │
       ▼
Configure Localization
       │
       ▼
Configure Tax
       │
       ▼
Tenant Ready
       │
       ▼
ERP Dashboard
```

---

# 20. Tenant Provisioning

After creating the tenant record, Niyanthra provisions the required resources.

```text
Create Tenant
      │
      ▼
Determine Tenant Type
      │
      ├── SHARED
      │      └── Register tenant in shared infrastructure
      │
      ├── DEDICATED
      │      └── Create/allocate dedicated database
      │
      └── SILO
             └── Provision isolated infrastructure
```

The tenant should only become `ACTIVE` after required provisioning succeeds.

Possible tenant states:

```text
PROVISIONING
ACTIVE
SUSPENDED
ARCHIVED
FAILED
```

---

# 21. Recommended `tenants` Table

### `tenants`

| Field             | Type                | Description               |
| ----------------- | -------------------- | -------------------------- |
| `id`              | BIGINT PK           | Internal ID               |
| `public_id`       | UUID UNIQUE         | Public/API-safe ID        |
| `organization_id` | BIGINT FK           | Organization              |
| `name`            | VARCHAR(150)        | Tenant name               |
| `slug`            | VARCHAR(100) UNIQUE | Tenant identifier         |
| `tenant_type`     | VARCHAR(30)         | shared / dedicated / silo |
| `status`          | VARCHAR(30)         | Tenant lifecycle status   |
| `created_at`      | TIMESTAMP           | Created date              |
| `updated_at`      | TIMESTAMP           | Updated date              |
| `deleted_at`      | TIMESTAMP NULL      | Soft deletion             |

---

# 22. Recommended `tenant_resources` Table

Infrastructure information should preferably be separated from the tenant identity.

### `tenant_resources`

| Field             | Type              | Description             |
| ----------------- | ----------------- | ------------------------ |
| `id`              | BIGINT PK         | Internal ID             |
| `tenant_id`       | BIGINT FK         | Tenant                  |
| `database_alias`  | VARCHAR(100)      | Application DB alias    |
| `database_name`   | VARCHAR(150) NULL | Database name           |
| `database_host`   | VARCHAR(255) NULL | Database host/reference |
| `database_schema` | VARCHAR(100) NULL | Schema if applicable    |
| `redis_namespace` | VARCHAR(100) NULL | Redis namespace         |
| `celery_queue`    | VARCHAR(100) NULL | Celery queue            |
| `status`          | VARCHAR(30)       | Resource status         |
| `provisioned_at`  | TIMESTAMP NULL    | Provisioning completion |
| `created_at`      | TIMESTAMP         | Created date            |
| `updated_at`      | TIMESTAMP         | Updated date            |

Sensitive infrastructure credentials should not be stored directly in ordinary database columns. They should be managed through appropriate secrets/configuration management.

---

# 23. Recommended `companies` Table Relationship

The company maintains the link to its tenant.

### `companies`

```text
id
public_id
organization_id
tenant_id
name
code
registration_number
company_type_id
status
description
logo
website
timezone
date_format
created_by
updated_by
created_at
updated_at
deleted_at
```

Relationship:

```text
organization_id → organizations.id
tenant_id       → tenants.id
```

---

# 24. Complete Organization → Tenant Structure

```text
                         ORGANIZATION
                              │
               ┌──────────────┼──────────────┐
               │              │              │
               ▼              ▼              ▼
           COMPANY A      COMPANY B      COMPANY C
               │              │              │
               ▼              ▼              ▼
           TENANT A       TENANT B       TENANT C
               │              │              │
        ┌──────┼──────┐       │              │
        ▼      ▼      ▼       ▼              ▼
     Branch  Branch  Branch  Branch        Branch
        1       2      3       1              1
```

---

# 25. User Context After Login

A user may belong to multiple organizations and companies.

Example:

```text
User
 │
 ├── Organization: ABC Group
 │      │
 │      ├── ABC Paints
 │      │      └── Tenant A
 │      │
 │      └── ABC Hotels
 │             └── Tenant B
 │
 └── Organization: XYZ Group
        │
        └── XYZ Traders
               └── Tenant C
```

After login, the user selects their working context.

```text
Login
  │
  ▼
Authenticate User
  │
  ▼
Load Organizations
  │
  ▼
Select Organization
  │
  ▼
Select Company
  │
  ▼
Resolve Company → Tenant
  │
  ▼
Select Branch
  │
  ▼
Resolve Access
  │
  ▼
Load Tenant Resources
  │
  ▼
Connect to Tenant Storage
  │
  ▼
ERP Dashboard
```

---

# 26. Request Context

Once the user selects the company and branch, Niyanthra maintains the current ERP context.

Example:

```json
{
  "user_id": 25,
  "organization_id": 1,
  "company_id": 10,
  "tenant_id": 100,
  "branch_id": 5
}
```

The application uses this context to determine:

```text
Who?
  ↓
user_id

Which organization?
  ↓
organization_id

Which company?
  ↓
company_id

Which tenant?
  ↓
tenant_id

Which branch?
  ↓
branch_id
```

---

# 27. Tenant Resolution

The application must resolve the tenant from trusted Master DB relationships.

```text
Authenticated User
       │
       ▼
Selected Company
       │
       ▼
Master DB
       │
       ▼
Company
       │
       └── tenant_id
              │
              ▼
           Tenant
              │
              ▼
       Tenant Resources
              │
              ▼
      Database Connection
              │
              ▼
        Business Data
```

The application should **not** trust an arbitrary database name supplied by the client.

The client selects a company/branch, but the server resolves the corresponding tenant and resources from the Master DB.

---

# 28. Tenant Isolation Principle

Every business request must have a valid tenant context.

```text
Request
   │
   ▼
Authentication
   │
   ▼
User Validation
   │
   ▼
Company Validation
   │
   ▼
Tenant Resolution
   │
   ▼
Branch Validation
   │
   ▼
Authorization
   │
   ▼
Tenant Database
   │
   ▼
Business Operation
```

If the tenant context is invalid, the request must not reach tenant business data.

---

# 29. Final Architecture

The complete Niyanthra architecture is:

```text
                         NIYANTHRA
                             │
                             ▼
                       MASTER DATABASE
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
 Authentication       Organization            Tenancy
        │                    │                    │
        │                    ▼                    ▼
        │                 Company              Tenant
        │                    │                    │
        │          ┌─────────┴─────────┐          │
        │          │                   │          │
        │          ▼                   ▼          ▼
        │       Branches          Configuration  Resources
        │                              │
        │                              │
        └──────────── Access Control ──┘
                             │
                             ▼
                    Tenant Infrastructure
                             │
             ┌───────────────┼────────────────┐
             │               │                │
             ▼               ▼                ▼
          SHARED          DEDICATED          SILO
             │               │                │
             ▼               ▼                ▼
       Shared Storage    Tenant DB       Isolated Stack
             │               │                │
             └───────────────┼────────────────┘
                             ▼
                      BUSINESS DATA
```

---

# 30. Final Design Principles

Niyanthra follows these principles:

1. **Organization is the top-level business owner/container.**
2. **Company represents the actual business entity.**
3. **Tenant represents the data-isolation boundary.**
4. **One company gets one tenant by default.**
5. **Branches belong to a company and are not tenants by default.**
6. **Business data belongs to the tenant.**
7. **Tenant infrastructure can be shared, dedicated, or siloed.**
8. **Company and tenant remain separate concepts.**
9. **Tenant identity should remain stable even if infrastructure changes.**
10. **Master DB manages identity, organization, company, tenant, access, configuration, and routing.**
11. **Tenant storage manages operational/business data.**
12. **Tenant resolution must always happen through trusted Master DB relationships.**

## Final Model

```text
Organization
    │
    ├── Company A
    │      │
    │      └── Tenant A
    │             ├── Branch 1
    │             ├── Branch 2
    │             └── Branch 3
    │
    ├── Company B
    │      │
    │      └── Tenant B
    │             ├── Branch 1
    │             └── Branch 2
    │
    └── Company C
           │
           └── Tenant C
                  └── Branch 1
```

### Core Relationship

```text
Organization
      ↓
Company
      ↓
Tenant
      ↓
Branches
      ↓
Business Data
```

### Core Infrastructure Model

```text
Tenant
  ├── Shared
  ├── Dedicated
  └── Silo
```

### Core Identity Model

```text
User
  ↓
Organization Membership
  ↓
Company Membership
  ↓
Company / Branch Context
  ↓
Tenant
  ↓
Business Data
```

This structure provides Niyanthra with a clean separation between **business ownership, business entities, operational branches, user access, and technical data isolation**, while keeping the platform flexible enough to support future growth and enterprise-level tenant isolation.
