# Niyanthra ERP — Implementation Assessment

> **Simplify. Manage. Grow.**

This page records what already exists in the documentation set before Sales, Purchase, and Inventory are specified. This repository is **MkDocs only**. Nothing below is running application code.

Phases 1–4 in this folder are **architecture and API contracts**. They do not implement Django services.

---

## Existing infrastructure

| Area | Status | Where documented | For Sales / Purchase / Inventory |
| ---- | ------ | ---------------- | -------------------------------- |
| Authentication | Documented | [Login Flow](../../sujith/doc1-loginflow.md) | Reuse. Do not redesign login, OAuth, or sessions. |
| Organization | Documented | Login Flow | Every operational document stores `organization_id`. |
| Company | Documented | Login Flow + [Company Onboarding](../doc1-company-onboarding.md) | Every document stores `company_id`. Subscription is company-level. |
| Branch | Documented | Login Flow; onboarding seeds `HO` | Every document stores `branch_id`. Warehouses belong to a branch. |
| User / member | Documented | `users`, `organization_memberships`, `company_memberships` | Created/approved/posted-by FKs point at Master users. |
| Permissions | Documented | [Permissions](../../sujith/doc2-permissions.md) — Permission → Role → Group → User, scoped by company/branch | Add resource.action codes (`sales.order.approve`). **Backend enforces.** UI hiding is not security. |
| Database | Documented | [Tenant Flow](../../sujith/doc3-tenant-flow.md) — PostgreSQL Master + tenant (`SHARED` / `DEDICATED` / `SILO`) | Sales, purchase, inventory, stock ledger, and journals live in the **tenant** database. Identity and billing stay in Master. |
| API | Not implemented | Tech stack is **Django + DRF** ([Technology Stack](../../Technology_Stack_Overview_and_Justification.md)) | Contracts use REST `/api/v1/...`. Not FastAPI. |
| Frontend | Documented only | Next.js + Flutter | UX rules and contextual AI Assist. No screens in this repo. |
| Background jobs | Documented | Celery; onboarding staged provision | AI, OCR, forecasting, alerts, and notifications run **after** posting, never on the critical path. |
| AI infrastructure | Mentioned only | Product overview, Dhamodhar roadmap | See [AI Architecture](ai-architecture.md). Phase 7. Draft-only writes. |
| Notification infrastructure | Missing | — | Named domain events in module docs; reusable notification product is Phase 6. |
| Audit infrastructure | Missing as an engine | Scattered created_at fields | See [Audit](audit.md). |

---

## What this pass adds

| Phase | Content |
| ----- | ------- |
| 1 | Document engine, workflow, relationships, calculation/pricing, inventory ledger, posting contract, audit, AI architecture |
| 2 | Inventory documents, warehouses, products, movements, valuation |
| 3 | Purchase documents, partial receipts, three-way match |
| 4 | Sales documents, optional stages, reservations, partial delivery |

**Out of this pass:** subscription/plans catalog, Phase 5 full general ledger, Phase 6 automation product, Phase 7 AI implementation, Phase 8 performance engineering.

---

## Architectural conflict (do not rewrite onboarding now)

[Company Onboarding](../doc1-company-onboarding.md) still seeds Quarto-style catalogs (`entry_master`, `voucher_master`, formula tables). That is **not** the Niyanthra operational model.

Sales, Purchase, and Inventory use **explicit documents** (sales order, delivery, stock transfer, vendor bill). They do not share one generic voucher row keyed by an entry type id.

Onboarding seed can remain as a legacy catalog note until a later cleanup. New module docs **replace** that voucher model for business operations.

---

## Reuse rules

1. Do not duplicate organization, company, tenant, or permission schemas.
2. Do not introduce a second tenancy model.
3. Tax types, rates, and rules stay as in the login-flow tax engine; calculation calls them, it does not fork them.
4. Default branch `HO` from onboarding is the first operational location; warehouses are additional inventory locations under a branch.
5. Django + DRF REST is the API surface. Idempotency, pagination, and error envelopes are shared conventions in the document engine.

---

## Stack reminder

```text
Next.js / Flutter
        │
        ▼
Django REST  /api/v1/...
        │
 Permission + validation
        │
 Business services  (Sales / Purchase / Inventory)
        │
 Document + workflow + calculation
        │
        ├── Inventory posting  →  stock ledger
        └── Accounting posting →  journal (Phase 5)
        │
 Celery: AI, OCR, alerts, notifications
```

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
