# ERP Architecture Overview

## Multi-Tenant Hybrid Architecture

The ERP will be built using a secure multi-tenant hybrid architecture to ensure proper data management, scalability, flexibility, and security.

During company registration, the company can select its preferred tenancy and data architecture through a simple and user-friendly one-time setup process.

The available options can include:

* Shared tenancy
* Isolated tenancy with separate or multiple databases
* Completely separated platform architecture

The selected architecture will be configured during the company setup process based on the company's requirements and subscription plan.

The system must ensure complete tenant-level security and data isolation according to the selected architecture.

## User Registration and Login

The platform will include:

* User registration
* Secure login
* Authentication
* Company registration
* User account management
* Multi-company access
* Company-based permissions

The initial process will begin with user registration and login, followed by company registration.

During company registration, the system will collect and configure company-specific information based on the selected business type or business types.

## Company Selection at Login

Since a single user can belong to multiple companies, the system must handle login differently depending on how many companies a user is linked to:

* If a user is associated with only one company, the system logs them in directly into that company's workspace.
* If a user is associated with multiple companies, after successful authentication the system will prompt the user with a **company selection screen**, listing all companies they have access to.
* The user must select the specific company they want to work in before proceeding further.
* Once a company is selected, the system loads that company's role, permissions, modules, dashboard, and data context for the session.
* The user can switch companies at any time during their session (for example, via a company switcher in the navigation bar) without needing to log out, and the system will reload the appropriate role, permissions, and data context for the newly selected company.
* All actions, data access, and module visibility during the session will strictly correspond to the currently selected company and the user's role within it.

This ensures that even though a user's identity is shared across companies, their access, permissions, and data view remain fully isolated and company-specific at all times.

## Company-Based User Permissions

User permissions will be managed at the company level.

A single user can belong to multiple companies. The same user can have different roles and permissions in different companies.

For example, a user can be:

* Administrator in one company
* Manager in another company
* Employee in another company

Therefore, user roles, permissions, access control, and data access will be completely company-based.

The system must ensure that users can only access the data, modules, features, and actions allowed for their role within the currently selected company.

## Business Type Selection

During company registration, a company can select one or multiple business types based on its requirements.

For example, a company may operate multiple types of businesses under the same organization.

Each selected business type can have its own:

* Modules
* Features
* Workflows
* Database configurations
* Database migrations
* Forms
* Permissions
* Business rules
* UI configurations

If a company selects multiple business types, the system should support and configure all required business modules independently while maintaining a common core ERP system.

The subscription cost can increase based on:

* Number of selected business types
* Required modules
* Number of branches
* Selected tenancy architecture
* Storage requirements
* AI features
* Additional services

## Subscription-Based Platform

The ERP will operate as a subscription-based platform.

Each company will select an appropriate subscription plan based on its requirements.

Once a company has an active subscription, its branches can access the modules, features, and services included in the company's subscription.

The subscription system should support different plans and configurations based on:

* Business type
* Number of business types
* Number of branches
* Number of users
* Required modules
* Storage
* AI features
* Data architecture
* Additional services

The subscription will be managed at the company level.

## Branch Management

A company can have one or multiple branches.

All branches will belong to the same company and can access the features and modules included in the company's subscription.

Branch-level data, users, permissions, operations, and configurations can be managed separately when required.

The system should maintain proper company-level and branch-level data separation.

## Dynamic UI Based on Business Type

The user interface and user experience can dynamically change based on the selected business type or business types.

Common ERP components will remain shared across the platform, while business-specific components can change dynamically.

The following can be configured based on the business type:

* Dashboard
* Navigation menus
* Modules
* Forms
* Workflows
* Reports
* Screens
* Business processes
* User interface components

This approach will allow each business to have a more relevant and customized user experience while maintaining a common and scalable ERP platform.

The system should use a flexible and configurable UI architecture so that UI changes can be made based on business type without creating a completely separate application for every business.
