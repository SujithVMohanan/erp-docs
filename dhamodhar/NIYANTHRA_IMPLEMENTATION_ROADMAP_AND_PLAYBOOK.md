# 🚀 Niyanthra ERP — Implementation Roadmap & Team Playbook

**Version:** 1.0  
**Date:** September 2026  
**Purpose:** Execution guide for Phase 1 go-live and beyond  
**Audience:** Development team leads, product managers, stakeholders

---

## Table of Contents

1. [Phase 1 Timeline & Milestones](#phase-1-timeline--milestones)
2. [Team Structure & Responsibilities](#team-structure--responsibilities)
3. [Critical Path Items (No-Go-Live Blockers)](#critical-path-items-no-go-live-blockers)
4. [Technical Debt & Risk Mitigation](#technical-debt--risk-mitigation)
5. [Go-Live Checklist](#go-live-checklist)
6. [Post-Launch (Month 4+) Roadmap](#post-launch-month-4-roadmap)

---

## Phase 1 Timeline & Milestones

### Month 1: Foundation & Core APIs

**Goals:**
- Complete architecture agreement (Sujith's Phase 1, your agentic Phase 2)
- Build foundational APIs
- Database schema finalized and deployed

**Week 1-2: Architecture Lockdown**
- [ ] Finalize tech stack (Django, PostgreSQL, Redis, Celery, Next.js, Flutter)
- [ ] Confirm multi-tenancy strategy (shared DB + RLS for Phase 1)
- [ ] Lock permission model (5-layer: Permission → Role → Group → User → Scope)
- [ ] Confirm tax engine design (tax types, rates, rules, registrations)
- [ ] **Decision Point:** Vector DB choice (Pinecone, Weaviate, or PostgreSQL pgvector)

**Week 3-4: Database & APIs**
- [ ] Database schemas deployed (Master DB + sample Tenant DB)
- [ ] Migration scripts tested (empty database → initialized schema)
- [ ] User authentication API (registration, login, SSO-ready)
- [ ] Company setup API (create company, set business profile, enable modules)
- [ ] Invoice API (create, retrieve, list, update, delete)
- [ ] Inventory API (stock levels, transfers, receipts)

**Deliverables:**
- API documentation (OpenAPI/Swagger)
- Database schema diagram
- Authentication flow diagram
- First test invoices end-to-end (via API, no UI yet)

---

### Month 2: Core Features & Agents

**Goals:**
- Build Invoice & Inventory modules (API + UI)
- Implement Sales & Inventory agents
- GST calculation engine ready

**Week 5-6: Invoice Module (Backend + Frontend)**
- [ ] Invoice creation API (customer, items, tax calculation)
- [ ] Invoice listing & retrieval
- [ ] Invoice PDF generation (printable, email-ready)
- [ ] GST calculation (intra-state vs. inter-state, HSN lookup)
- [ ] e-Invoice JSON generation (NSDL format, QR code)
- [ ] Next.js UI: Invoice creation form (guided by agent)
- [ ] Next.js UI: Invoice list view (search, filter, drill-down)

**Week 7-8: Inventory Module (Backend + Frontend)**
- [ ] Inventory ledger API (stock in, stock out, transfers, adjustments)
- [ ] Stock levels API (current balance per warehouse)
- [ ] Stock alert logic (when stock <= reorder point, trigger agent)
- [ ] Next.js UI: Inventory dashboard (stock levels, warehouse-wise)
- [ ] Next.js UI: Stock transfer form
- [ ] Warehouse master setup API

**Agent Implementation (Parallel Track)**
- [ ] Sales Task Agent (Claude Sonnet) — Guide invoice creation
- [ ] Inventory Query Agent (Mistral) — Answer "What's our stock?"
- [ ] Suggestion Agent (Mistral) — "Time to reorder Red Paint?"
- [ ] RAG Engine — Context building from Master DB + Tenant DB
- [ ] Guardrails Framework (basic) — Confirmation for > ₹50,000

**Deliverables:**
- Invoice module (API + UI) feature-complete
- Inventory module (API + UI) feature-complete
- 3 agents integrated and tested
- GST tax engine validated against 10+ scenarios

---

### Month 3: Accounting & Compliance

**Goals:**
- Accounting module ready (GL, AP, AR, bank recon)
- GSTR-1 auto-population working
- TDS framework designed (not full implementation)
- Internal testing with 2-3 beta customers

**Week 9-10: Accounting & GL**
- [ ] GL posting from invoices, purchases, payments (automatic)
- [ ] GL ledger API (retrieve account balances, transactions)
- [ ] Trial balance API
- [ ] Month-end close checklist (via API)
- [ ] Bank reconciliation API (manual matching first, auto-matching Phase 2)
- [ ] AP aging, AR aging reports
- [ ] Next.js UI: GL dashboard, trial balance, P&L report

**Week 11-12: GST & Compliance**
- [ ] GSTR-1 report generation (from sales invoices)
- [ ] GSTR-1 data validation (against GST portal requirements)
- [ ] e-Invoice batch generation (for multiple invoices)
- [ ] TDS tracking (database structure, GL postings)
- [ ] Compliance checklist (GST registration, invoicing requirements)
- [ ] Next.js UI: GSTR-1 preview (what will be filed)

**Beta Testing**
- [ ] Onboard 2-3 beta customers (friendly accounts)
- [ ] End-to-end testing: Order → Invoice → Payment → AR aging
- [ ] Invoice PDF quality check
- [ ] e-Invoice QR code validation
- [ ] GSTR-1 data accuracy vs. manual calculations
- [ ] Bug log + triage

**Deliverables:**
- Accounting module complete
- GSTR-1 report (ready to file)
- Beta feedback incorporated
- Known issues documented for Phase 1.1

---

### Month 4: Polish, Launch, & Scaling to 3+ Customers

**Goals:**
- Fix critical bugs from beta testing
- Launch Phase 1 to 3+ paying customers
- Establish support playbook

**Week 13-14: Bug Fixes & Polish**
- [ ] Fix critical bugs from beta testing
- [ ] UI/UX refinement (based on beta user feedback)
- [ ] Performance optimization (invoice creation <2 seconds)
- [ ] Security hardening (SQL injection, XSS, CSRF checks)
- [ ] Accessibility review (keyboard nav, screen reader support)

**Week 15: Go-Live Preparation**
- [ ] Staging environment parity with production
- [ ] Backup & recovery procedures tested
- [ ] Monitoring setup (error tracking, performance alerts)
- [ ] Customer onboarding playbook (guide new users)
- [ ] Support ticket template & triage process

**Week 16: Launch & Scale to 3+ Customers**
- [ ] Deploy to production
- [ ] Onboard 3+ paying customers
- [ ] Daily standup (support issues, rollback plan on standby)
- [ ] Collect NPS feedback (target: >30)
- [ ] Document learnings (what went well, what to improve for Phase 2)

**Deliverables:**
- Production go-live
- 3+ paying customers
- NPS > 30 (or clear action items to improve)
- Phase 1.1 bug fix roadmap

---

## Team Structure & Responsibilities

### Core Team (Months 1-4)

#### Backend Team (4 engineers)
**Lead:** Backend Tech Lead  
**Responsibility:**
- Django APIs (users, companies, invoices, inventory, accounting)
- Database schema, migrations, indexing
- Celery tasks (async invoice PDF generation, GSTR-1 export)
- Integration with Anthropic Claude + Mistral

**Sprint Breakdown:**
- **Backend 1:** User authentication, company setup, multi-tenancy middleware
- **Backend 2:** Invoice module (creation, GL posting, GST calculation)
- **Backend 3:** Inventory module (stock ledger, warehouse management)
- **Backend 4:** Accounting module (GL, AP, AR, reconciliation)

#### Frontend Team (2 engineers)
**Lead:** Frontend Tech Lead  
**Responsibility:**
- Next.js app (customer-facing dashboard)
- UI/UX design collaboration
- Mobile responsiveness
- Form validation, error handling

**Focus Areas:**
- **Frontend 1:** Invoice creation flow (guided by agent), invoice list
- **Frontend 2:** Inventory dashboard, accounting reports

#### AI/Agent Team (1 engineer)
**Lead:** AI Engineer  
**Responsibility:**
- Claude Sonnet + Mistral integration
- RAG engine (context building)
- Guardrails framework
- Prompt management

**Tasks:**
- Implement agent router (which LLM for which task)
- Build RAG data retrieval (company context)
- Test agent workflows (invoice creation, stock query)
- Implement audit logging (what agent suggested, what user did)

#### QA Team (1 engineer)
**Lead:** QA Lead  
**Responsibility:**
- Test plan creation (what to test in each module)
- Manual testing (invoicing workflow, GST calculation, inventory transfers)
- Regression testing (bug fixes don't break other features)
- Beta customer onboarding support

**Test Focus:**
- **GST Accuracy:** 20+ scenarios (intra-state, inter-state, mixed HSN codes)
- **Inventory Consistency:** Stock after invoice creation matches subledger
- **GL Balancing:** Trial balance always balances, AR aging = GL balance
- **Agent Behavior:** Suggestions are accurate, confidence scores honest

#### Product Manager (1 person)
**Lead:** Product Manager  
**Responsibility:**
- Requirements translation (from Functional Spec to Dev tasks)
- Priority management (what gets built in Phase 1 vs. Phase 2)
- Customer feedback collection (beta users)
- Go-live readiness (checklist, communication)

**Deliverables:**
- Sprint planning (what ships each sprint)
- Beta customer communication (progress updates)
- Competitive analysis (vs. other SMB ERPs)
- Roadmap communication (to stakeholders)

---

### Supporting Roles

#### DevOps / Infrastructure (0.5 FTE initially, 1 FTE post-launch)
- Database setup (Master DB, Tenant DB schema provisioning)
- Hosting setup (AWS/GCP, Docker, CI/CD)
- Monitoring & alerting (Sentry, Datadog, CloudWatch)
- Backup procedures (daily backups, disaster recovery)

#### Customer Support (0 FTE initially, 1 FTE at launch)
- Onboarding new customers (setup, training)
- Bug triage (support tickets, categorization)
- Feature requests (log, communicate with product)

---

## Critical Path Items (No-Go-Live Blockers)

### Must-Have for Phase 1

| Item | Owner | Status | Risk |
|---|---|---|---|
| **Authentication & Multi-Tenancy** | Backend Lead | Month 1 | If fails: Can't secure data, can't launch |
| **Invoice API + GST Calculation** | Backend 2 | Month 2 | If fails: Core business model broken |
| **Invoice UI Form** | Frontend 1 | Month 2 | If fails: Can't create invoices manually |
| **GL Posting from Invoice** | Backend 4 | Month 2 | If fails: Accounting doesn't work |
| **GSTR-1 Report Generation** | Backend 4 | Month 3 | If fails: Can't file taxes |
| **e-Invoice (QR, IRN)** | Backend 2 | Month 3 | If fails: e-invoice mandate risk |
| **Sales Agent (Claude)** | AI Engineer | Month 2 | If fails: AI differentiation lost (but invoice still works) |
| **Inventory API** | Backend 3 | Month 2 | If fails: Stock visibility broken |
| **Banking Integration Demo** | DevOps | Month 3 | If fails: Demo-only, but good-to-have for beta |

### Nice-to-Have for Phase 1

- [ ] Mobile app (can defer to Phase 2)
- [ ] Banking auto-reconciliation (manual matching sufficient)
- [ ] PO creation (can manage via notes/email initially)
- [ ] Payroll (not critical for traders/wholesale)
- [ ] Multi-currency (rarely needed in Phase 1 customer base)

---

## Technical Debt & Risk Mitigation

### Identified Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **GST rule changes mid-month** | GSTR-1 incorrect, compliance risk | Keep tax rules in DB (not hardcoded), quick update mechanism |
| **e-Invoice IRN failure** | Invoices not e-invoiced, NSDL penalty | Fallback to offline e-invoice submission (manual), retry queue |
| **Multi-tenancy data leak** | Customer data exposed, breach, company dies | Code review all tenant_id filters, automated tests, penetration testing |
| **Invite-only SaaS, no backup** | Data loss, customer unrecoverable | Automatic daily backups, restore test quarterly |
| **Agent hallucination (wrong GST)** | Incorrect invoices filed, audit risk | Agent always validates against tax engine, human confirmation for > ₹50,000 |
| **Performance degrades with scale** | Slow invoicing, poor UX | Index on company_id + created_at, query optimization early, load testing |
| **Vendor lock-in (Anthropic Claude)** | Expensive, single point of failure | Hybrid LLM (Mistral fallback), invoice still works if Claude down |

### Technical Debt Strategy

**Month 1-4 (Phase 1):** Zero debt allowed
- Code review all PRs (no merging without review)
- Write tests for core business logic (invoice GST, GL posting)
- No hardcoded values (use DB config)

**Month 5+ (Phase 2):** Controlled debt acceptance
- Document known issues (log in backlog)
- Track debt (time-to-fix estimates)
- Re-prioritize quarterly (is debt slowing development? address it)

---

## Go-Live Checklist

### 1 Week Before Launch

- [ ] **Database**: Production DB created, backups tested (restore from backup in <1 hour)
- [ ] **Code**: All critical bugs fixed, main branch is release-ready
- [ ] **Monitoring**: Error tracking (Sentry), performance alerts (Datadog) enabled
- [ ] **Support**: Support tickets system ready, team trained on triage
- [ ] **Documentation**: User guide drafted (invoice creation, stock check, payment tracking)
- [ ] **Customers**: 3 customers onboarded, data migrated, systems tested

### Launch Day

- [ ] **Deployment**: Code deployed to production, monitoring active
- [ ] **Smoke Tests**: 
  - [ ] Can create user account
  - [ ] Can create company
  - [ ] Can create invoice (end-to-end)
  - [ ] Can view stock levels
  - [ ] Can generate GSTR-1 report
- [ ] **Communication**: Email to 3 customers: "Live now, go ahead and test"
- [ ] **On-Call**: Engineering team on standby (2-person rotation)

### Day 1-7 (Launch Week)

- [ ] **Monitoring**: Check error logs hourly (first 3 days)
- [ ] **Customer Check-ins**: Daily call with each beta customer (any blockers?)
- [ ] **Metrics**: Track invoice creation count, API response times, error rate
- [ ] **Hotfixes**: Critical bugs fixed same-day (e.g., invoice won't save)
- [ ] **Rollback Plan Ready**: If major issue, can rollback in <30 min

### Week 2-4 (Stabilization)

- [ ] **Documentation**: Customer feedback incorporated into user guide
- [ ] **Performance**: Invoice creation latency <2 seconds (measure and optimize)
- [ ] **Support**: All customer issues triaged and tracked
- [ ] **Feedback Collection**: NPS survey sent to customers (target: >30)
- [ ] **Learning Document**: What went well, what to improve for Phase 2

---

## Post-Launch (Month 4+) Roadmap

### Month 5-6: Phase 2 Planning & Expansion

**Objectives:**
- Scale to 10-15 customers
- Add Phase 2 modules (Accounting agents, CRM, TDS)
- Improve based on Phase 1 learnings

**Key Additions:**
- [ ] Accounting reconciliation agent (auto-match bank statement to GL)
- [ ] TDS tracking & provisioning
- [ ] CRM basics (customer segmentation, repeat purchase analysis)
- [ ] Purchase automation (reorder suggestions from inventory)
- [ ] Mobile app MVP (offline invoicing, stock check)

**Team Growth:**
- Add 2 more backend engineers (scale to 6)
- Add 1 more QA engineer
- Add 1 customer success manager (dedicated)

---

### Month 7-8: Advanced Features

**Objectives:**
- Scale to 25-50 customers
- Add e-way bill integration
- Payroll basics

**Key Additions:**
- [ ] e-Way Bill generation (auto from inter-state invoices)
- [ ] Payroll basics (salary calculation, payslips)
- [ ] Banking integration (API feeds from banks)
- [ ] Advanced reports (custom dashboard, drill-down)

---

### Month 9-12: Enterprise Features

**Objectives:**
- Scale to 50-100+ customers
- Add Phase 3 features (Manufacturing, Advanced HR)
- Prepare for regional expansion

**Key Additions:**
- [ ] Manufacturing (BOM, production orders, job costing)
- [ ] Advanced HR (leave, appraisals, employee self-service)
- [ ] Fixed assets (depreciation, disposal)
- [ ] Multi-currency accounting
- [ ] Predictive analytics (demand forecasting)

---

## Decision Log

**Record of all critical decisions for phase 1:**

| Decision | Option Chosen | Rationale | Status |
|---|---|---|---|
| LLM for Phase 2 | Claude Sonnet + Mistral | Best accuracy/cost balance | ✅ LOCKED |
| Multi-tenancy DB model | Shared DB + RLS (Phase 1), dedicated DB option (Phase 2) | Cost-effective for startups, scale-ready | ✅ LOCKED |
| GST calculation | Stored in DB, not hardcoded | Rules change frequently, need flexibility | ✅ LOCKED |
| e-Invoice approach | Auto-generate JSON, submit to NSDL via API | Compliance, audit trail | ✅ LOCKED |
| Payment processing | Payments recorded manually (Phase 1), auto bank feed (Phase 2) | Simpler MVP, fast to build | ✅ LOCKED |
| Vector DB | PostgreSQL pgvector (Phase 1), Pinecone if needed (Phase 2) | No new infra, cost-effective | ✅ LOCKED |
| Mobile strategy | Defer to Phase 2, web-responsive (Phase 1) | Browser enough for MVP, mobile later | ✅ LOCKED |

---

## Success Metrics

### Phase 1 Success = All of the following

- ✅ **3+ paying customers live** (by Month 4, Week 4)
- ✅ **NPS > 30** (at least one customer gives >7 score)
- ✅ **$0 critical bugs after launch** (things that crash the system)
- ✅ **Uptime > 99.5%** (production available)
- ✅ **Invoice creation <2 seconds** (user experience)
- ✅ **GSTR-1 accuracy 100%** (tax filing correct)
- ✅ **$0 data loss incidents** (backups, recovery work)
- ✅ **Team velocity improving** (not slowing down as features added)

### Phase 2 Success = All of the following

- ✅ **10-15 paying customers** (churn <5%)
- ✅ **NPS > 40**
- ✅ **Revenue >₹2 lakhs/month** (by Month 8)
- ✅ **Zero regulatory compliance issues** (GST audit, no penalties)

---

## Conclusion

This playbook provides a **clear, executable path** to launch Niyanthra Phase 1 with a strong foundation for scaling.

**Key Success Factors:**
1. **Strict scope discipline** — Only Phase 1 features, defer everything else
2. **Daily communication** — Team sync, customer updates, support triage
3. **Quality gates** — Code review, testing, monitoring before launch
4. **Fast feedback loops** — Beta customers → learnings → next sprint
5. **Technical excellence** — Clean code, good architecture, maintainable

**Launch Target:** Month 4, Week 4 with 3+ paying customers and NPS > 30.

