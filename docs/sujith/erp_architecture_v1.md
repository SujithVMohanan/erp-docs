# Global ERP Organization, Multi-Tenant & Tax Architecture

## 1. ERP Platform

The ERP Platform is a global, multi-tenant business platform designed to support different organizations, companies, branches, business types, industries, currencies, tax jurisdictions, users, permissions, and business operations.

```text
ERP PLATFORM
├── Authentication
├── Organizations
├── Companies
├── Branches
├── Departments
├── Warehouses
├── Users & Roles
├── Permissions & Access Control
├── Business Profiles
├── Modules & Capabilities
├── Tax & Compliance
├── Currency & Localization
├── Tenant Management
├── Database Routing
├── Notifications
├── Audit & Security
└── Business Operations
```

## 2. Organization

An **Organization** is the top-level customer or business group using the ERP. An organization may contain one or multiple companies.

```text
Organization
├── Companies
├── Users
├── Roles
├── Permissions
├── Business Configuration
├── Subscription
└── ERP Settings
```

Example:

```text
Global Holdings Group
├── Global Retail Ltd
├── Global Manufacturing Ltd
└── Global Services Ltd
```

A small customer may have only one company:

```text
Bright Foods
└── Bright Foods LLC
```

## 3. Company

A **Company** represents a legal or operational business entity. A company owns its financial and legal identity.

```text
Organization
└── Company
    ├── Legal Information
    ├── Financial Configuration
    ├── Tax Registrations
    ├── Branches
    ├── Business Profile
    ├── Accounting
    └── Business Operations
```

Company-level configuration can include:

```text
Company
├── Legal Name
├── Registration Information
├── Fiscal Year
├── Base Currency
├── Accounting Settings
├── Tax Registrations
└── Compliance Settings
```

## 4. Branch

A **Branch** represents a physical or operational location of a company.

```text
Company
├── Branch A
├── Branch B
├── Branch C
└── Branch D
```

A branch can contain:

```text
Branch
├── Address
├── Country
├── State / Province / Region
├── Departments
├── Employees
├── Warehouses
└── Local Operational Configuration
```

Branches should not automatically be treated as separate legal entities.

## 5. Business Profile

The **Business Profile** defines what a company does and how it operates.

```text
Company
└── Business Profile
    ├── Business Type
    ├── Industry
    ├── Business Model
    ├── Operational Model
    └── Capabilities
```

Example:

```text
Business Type: Retail
Industry: Apparel
Business Model: B2B + B2C
Operational Model: Multi-Branch
```

## 6. Business Type & Industry

Business Type and Industry should be configurable rather than hard-coded into the ERP.

```text
Business Type
├── Retail
├── Wholesale
├── Distribution
├── Manufacturing
├── Services
├── Construction
├── Hospitality
├── Healthcare
├── Education
├── E-Commerce
└── Other
```

Industry can provide more specific classification:

```text
Industry
├── Apparel
├── Automotive
├── Food & Beverage
├── Electronics
├── Software
├── Logistics
├── Real Estate
└── Other
```

A company may support multiple business types:

```text
Company
├── Retail
└── Wholesale
```

## 7. Business Model

Business Model describes how the company conducts business.

```text
Business Model
├── B2B
├── B2C
├── B2B2C
├── Marketplace
├── Subscription
├── Project-Based
├── Distribution
└── Hybrid
```

## 8. Capabilities & Enabled Modules

Business configuration determines which ERP capabilities and modules are enabled.

```text
Business Profile
       ↓
Capabilities
       ↓
Enabled Modules
```

Examples:

```text
Retail
├── Sales
├── POS
├── Customers
├── Purchase
├── Inventory
├── Suppliers
└── Accounting
```

```text
Manufacturing
├── Sales
├── Purchase
├── Inventory
├── BOM
├── Production
├── Work Orders
├── Quality
├── Maintenance
└── Accounting
```

```text
Services
├── Customers
├── Projects
├── Tasks
├── Employees
├── Timesheets
├── Billing
└── Accounting
```

Modules should be enabled or disabled through configuration rather than creating a separate ERP application for every business type.

## 9. Tax & Compliance

Tax should be treated as an independent **global Tax & Compliance layer**.

Tax should NOT simply be:

```text
Company → Tax
```

or:

```text
Branch → Tax
```

Instead:

```text
Company
│
├── Branches
├── Tax Registrations
├── Tax Profiles
├── Tax Rules
└── Compliance Configuration
```

Tax can depend on several factors:

```text
Transaction
    ↓
Seller Location / Registration
    ↓
Customer Location / Tax Status
    ↓
Supply / Transaction Jurisdiction
    ↓
Product / Service Tax Category
    ↓
Applicable Tax Rules
    ↓
Tax Calculation
```

## 10. Tax Registration

A company may have one or multiple tax registrations.

```text
Company
│
├── Tax Registration A
│   ├── Country
│   ├── Jurisdiction
│   ├── Registration Number
│   ├── Tax Authority
│   ├── Effective From
│   └── Effective To
│
└── Tax Registration B
    ├── Country
    ├── Jurisdiction
    ├── Registration Number
    ├── Tax Authority
    ├── Effective From
    └── Effective To
```

A branch can reference its appropriate registration:

```text
Company
│
├── Tax Registration - Jurisdiction A
├── Tax Registration - Jurisdiction B
│
├── Branch A
│   └── Default Tax Registration → A
│
└── Branch B
    └── Default Tax Registration → B
```

This supports companies operating across multiple jurisdictions.

## 11. Global Tax Engine

The ERP should have a configurable Tax Engine rather than hard-coding a particular country's tax system.

```text
TAX ENGINE
├── Tax Jurisdictions
├── Tax Authorities
├── Tax Registrations
├── Tax Types
├── Tax Rates
├── Tax Categories
├── Tax Rules
├── Exemptions
├── Thresholds
├── Effective Dates
├── Tax Inclusive / Exclusive Rules
├── Withholding Rules
└── Compliance Rules
```

Potential tax types include:

```text
VAT
GST
Sales Tax
Consumption Tax
Excise Tax
Withholding Tax
Import / Export Duties
Digital / Service Taxes
```

The actual names and rules depend on the country and jurisdiction.

## 12. Tax Rules

Tax rates should not be stored as a single fixed company value.

```text
Tax Rule
├── Jurisdiction
├── Tax Type
├── Tax Category
├── Rate
├── Effective From
├── Effective To
├── Transaction Type
├── Customer Type
└── Product / Service Category
```

This allows tax rates and rules to change over time without changing historical transactions.

## 13. Currency & Localization

Global ERP should separate tax from localization.

```text
Company
├── Base Currency
├── Operating Countries
├── Tax Configuration
├── Date Format
├── Number Format
├── Language
└── Regional Settings
```

A transaction can have:

```text
Transaction Currency
        ↓
Exchange Rate
        ↓
Company Base Currency
```

## 14. Users, Roles & Permissions

Users should not automatically receive access to all companies and branches.

```text
User
 ↓
Organization Access
 ↓
Company Access
 ↓
Branch Access
 ↓
Role
 ↓
Permissions
 ↓
Access Scope
```

Permission scopes can include:

```text
Organization
Company
Branch
Department
Warehouse
Own Records
```

Example:

```text
Branch Manager
├── Sales → View/Create/Edit → Branch
├── Inventory → View → Branch
├── Customers → View/Create/Edit → Branch
└── Accounting → No Access
```

## 15. Multi-Company User

One user may work across multiple companies.

```text
User: Finance Manager
│
├── Company A
│   └── Finance Role
│
├── Company B
│   └── Finance Role
│
└── Company C
    └── View-Only Role
```

Access should therefore be configurable per company and/or branch.

## 16. Tenant

A **Tenant** represents the technical data-isolation boundary.

Organization and Tenant should not necessarily be the same concept.

```text
Organization
      ↓
Tenant Configuration
      ↓
Database Routing
      ↓
Business Data
```

## 17. Multi-Tenant Database Strategy

The ERP can support different isolation levels depending on customer size, requirements, security, and subscription.

### Small Business

```text
Tenant
   ↓
Shared Database
   ↓
tenant_id
   ↓
PostgreSQL RLS
```

### Medium Business

```text
Tenant
   ↓
Dedicated Database
```

### Large / Enterprise

```text
Tenant
   ↓
Dedicated Database
   ↓
Dedicated Infrastructure
```

### Highly Sensitive / Special Requirements

```text
Tenant
   ↓
Dedicated Application
   ↓
Dedicated Database
   ↓
Dedicated Infrastructure
```

The business hierarchy remains the same regardless of the technical isolation model.

## 18. Control Plane

The Control Plane contains global configuration and routing information.

```text
CONTROL PLANE
├── Organizations
├── Companies
├── Branches
├── Users
├── Roles
├── Permissions
├── Business Profiles
├── Business Types
├── Industries
├── Enabled Modules
├── Tax Registrations
├── Tax Configuration
├── Tenant Configuration
├── Database Routing
├── Subscriptions
├── Feature Flags
├── Localization
└── System Configuration
```

## 19. Business Data Plane

The Business Data Plane contains operational business information.

```text
BUSINESS DATA PLANE
├── Customers
├── Suppliers
├── Products
├── Services
├── Sales
├── Purchase
├── Inventory
├── Warehouses
├── Accounting
├── Payments
├── Employees
├── Payroll
├── Projects
├── Manufacturing
└── Reports
```

## 20. Complete Business Structure

```text
ERP PLATFORM
│
▼
ORGANIZATION
│
├── COMPANY
│   │
│   ├── BUSINESS PROFILE
│   │   ├── Business Type
│   │   ├── Industry
│   │   ├── Business Model
│   │   ├── Operational Model
│   │   └── Capabilities
│   │
│   ├── BRANCH
│   │   ├── Department
│   │   ├── Warehouse
│   │   └── Employees
│   │
│   ├── TAX & COMPLIANCE
│   │   ├── Tax Registrations
│   │   ├── Tax Profiles
│   │   ├── Tax Categories
│   │   ├── Tax Rules
│   │   └── Compliance
│   │
│   ├── CURRENCY & LOCALIZATION
│   │   ├── Base Currency
│   │   ├── Operating Currency
│   │   ├── Language
│   │   └── Regional Settings
│   │
│   └── ENABLED MODULES
│       ├── Sales
│       ├── Purchase
│       ├── Inventory
│       ├── Accounting
│       ├── HR
│       ├── Payroll
│       ├── Manufacturing
│       └── Reporting
│
├── USERS
│   └── ROLES
│       └── PERMISSIONS
│           └── ACCESS SCOPE
│
└── TENANT CONFIGURATION
    └── DATABASE ROUTING
        ├── Shared DB + RLS
        ├── Dedicated DB
        └── Dedicated / Silo Infrastructure
```

## 21. Complete Global ERP Architecture

```text
                                      ERP PLATFORM
                                           │
                                           ▼
                              ┌────────────────────────┐
                              │      CONTROL PLANE     │
                              │                        │
                              │ Organizations          │
                              │ Companies              │
                              │ Branches               │
                              │ Users                  │
                              │ Roles                  │
                              │ Permissions            │
                              │ Business Profiles      │
                              │ Modules                │
                              │ Tax Configuration      │
                              │ Localization           │
                              │ Tenant Routing         │
                              │ Feature Flags          │
                              └────────────┬───────────┘
                                           │
             ┌─────────────────────────────┼─────────────────────────────┐
             │                             │                             │
             ▼                             ▼                             ▼
       ORGANIZATION A               ORGANIZATION B               ORGANIZATION C
       Small Business               Medium Business              Enterprise Group
             │                             │                             │
             ▼                             ▼                             ▼
          COMPANY                     COMPANY                     COMPANIES
             │                             │                    ┌────────┼────────┐
             │                             │                    ▼        ▼        ▼
          BRANCHES                    BRANCHES               Company A Company B Company C
             │                             │                    │        │        │
        WAREHOUSES                   WAREHOUSES              Branches Branches Branches
             │                             │                    │        │        │
             └──────────────┬──────────────┴────────────────────┴────────┴────────┘
                            │
                            ▼
                    BUSINESS PROFILE
                            │
             ┌──────────────┼───────────────┐
             ▼              ▼               ▼
        Business Type    Industry     Business Model
             │              │               │
             └──────────────┼───────────────┘
                            ▼
                       CAPABILITIES
                            │
                            ▼
                      ENABLED MODULES
                            │
       ┌──────────┬─────────┼──────────┬──────────┐
       ▼          ▼         ▼          ▼          ▼
     Sales     Purchase  Inventory  Accounting    HR
       │          │         │          │
       └──────────┴─────────┼──────────┘
                            ▼
                     BUSINESS DATA
                            │
                            ▼
                       TAX ENGINE
                            │
             ┌──────────────┼───────────────┐
             ▼              ▼               ▼
        Jurisdiction    Tax Rules       Tax Registration
             │              │               │
             └──────────────┼───────────────┘
                            ▼
                       TAX RESULT
```

## 22. User Login & Tenant Routing Flow

```text
USER
 │
 ▼
LOGIN / SSO
 │
 ▼
AUTHENTICATION
 │
 ▼
IDENTIFY ORGANIZATION
 │
 ▼
CHECK COMPANY ACCESS
 │
 ▼
CHECK BRANCH ACCESS
 │
 ▼
LOAD ROLE
 │
 ▼
LOAD PERMISSIONS
 │
 ▼
CREATE ERP CONTEXT
 │
 ├── Organization
 ├── Company
 ├── Branch
 ├── User
 ├── Role
 └── Permissions
 │
 ▼
LOAD TENANT CONFIGURATION
 │
 ▼
DATABASE ROUTING
 │
 ├── Shared DB + RLS
 ├── Dedicated DB
 └── Silo Infrastructure
 │
 ▼
BUSINESS OPERATION
 │
 ▼
PERMISSION CHECK
 │
 ▼
TAX / BUSINESS RULES
 │
 ▼
DATABASE
 │
 ▼
RESPONSE
```

## 23. Transaction & Tax Flow

For every financial transaction, the ERP should determine the applicable tax dynamically.

```text
Sales / Purchase Transaction
          │
          ▼
      Company
          │
          ▼
       Branch
          │
          ▼
 Seller Tax Registration
          │
          ▼
 Customer / Supplier
 Tax Information
          │
          ▼
 Transaction Location
 / Tax Jurisdiction
          │
          ▼
 Product / Service
 Tax Category
          │
          ▼
       Tax Engine
          │
          ▼
     Applicable Rules
          │
          ▼
      Tax Calculation
          │
          ▼
 Invoice / Transaction
          │
          ▼
      Accounting
          │
          ▼
      Reporting
          │
          ▼
     Compliance
```

## 24. Core Architecture Principles

```text
Organization
= Top-level business/customer account

Company
= Legal/business entity

Branch
= Physical/operational location

Department
= Functional business unit

Warehouse
= Inventory/storage location

Business Profile
= What the company does

Business Type
= Type of business operation

Industry
= Industry classification

Business Model
= How the company generates/handles business

Module
= ERP functionality enabled for the company

Tax Registration
= Legal registration with a tax authority

Tax Jurisdiction
= Geographic/legal area where tax rules apply

Tax Engine
= Determines applicable tax dynamically

Tenant
= Technical data-isolation boundary

User
= Person accessing the ERP

Role
= Collection of permissions

Permission
= What a user can do

Scope
= Where the user can do it

Control Plane
= Configuration, identity, routing and governance

Business Data Plane
= Actual operational business data
```

## 25. Future-Proof Goal

The architecture should support:

```text
Small Business
      ↓
Single Company
      ↓
Multiple Branches
      ↓
Medium Business
      ↓
Multiple Companies
      ↓
Multi-Country Business
      ↓
Multi-Tax-Jurisdiction
      ↓
Enterprise Group
      ↓
Different Database Isolation
      ↓
Global ERP Platform
```

### Final Design Principle

```text
BUSINESS STRUCTURE
Organization
    ↓
Company
    ↓
Branch
    ↓
Department / Warehouse


BUSINESS CLASSIFICATION
Company
    ↓
Business Profile
    ↓
Business Type + Industry + Business Model
    ↓
Capabilities
    ↓
Modules


TAX STRUCTURE
Company
    ↓
Tax Registrations
    ↓
Tax Jurisdictions
    ↓
Tax Rules
    ↓
Tax Engine
    ↓
Transaction Tax


ACCESS STRUCTURE
User
    ↓
Organization Access
    ↓
Company Access
    ↓
Branch Access
    ↓
Role
    ↓
Permission
    ↓
Scope


TECHNICAL STRUCTURE
Organization
    ↓
Tenant Configuration
    ↓
Database Routing
    ↓
Shared / Dedicated / Silo
```

This separation allows the ERP to remain globally usable, multi-company, multi-branch, multi-country, multi-currency, multi-tax, and multi-tenant without tying the architecture to one country's business or tax model.
