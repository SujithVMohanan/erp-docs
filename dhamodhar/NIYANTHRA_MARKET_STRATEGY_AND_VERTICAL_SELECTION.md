# 🎯 Niyanthra ERP — Market Strategy & Vertical Selection

**Version:** 1.0
**Date:** September 2026
**Document Series:** Part 0 of 3 — precedes *Implementation Roadmap & Team Playbook (v1.0)* and *Functional Module & Data Architecture Specification (v1.0)*
**Type:** Internal business case / decision document
**Purpose:** Establish the market position Niyanthra is being built into, select a beachhead vertical, and translate that selection into binding changes to Parts 1 and 2
**Audience:** Founders, product lead, engineering lead, commercial lead
**Decision required:** Approve or reject the beachhead recommendation in §7 by the end of Playbook Month 1

---

## Table of Contents

1. [The Decision This Document Makes](#1-the-decision-this-document-makes)
2. [Recommendation Summary](#2-recommendation-summary)
3. [Market Reality](#3-market-reality)
4. [Competitive Position](#4-competitive-position)
5. [The Central Tension: A Horizontal Product in a Vertical Market](#5-the-central-tension-a-horizontal-product-in-a-vertical-market)
6. [Vertical Selection Framework](#6-vertical-selection-framework)
7. [Recommendation and Rationale](#7-recommendation-and-rationale)
8. [Consequences for Parts 1 and 2](#8-consequences-for-parts-1-and-2)
9. [Commercial Model](#9-commercial-model)
10. [Risks and Kill Criteria](#10-risks-and-kill-criteria)
11. [Validation Plan Before Commitment](#11-validation-plan-before-commitment)
12. [Document Control](#12-document-control)

---

## 1. The Decision This Document Makes

Parts 1 and 2 describe a competent, well-architected horizontal ERP: Sales & Distribution, Purchase, Inventory, Financial Accounting, POS, CRM and HR, with GST compliance built in rather than bolted on. Neither document answers the question that determines whether any of it gets sold: **who, specifically, buys this, and why would they switch?**

Absent an answer, the default outcome is a general-purpose ERP sold to whoever will listen. That is the highest-failure-rate motion in this market. The purpose of this document is to close that gap before engineering momentum makes it expensive to close.

Two decisions are on the table:

1. **Positioning.** Does Niyanthra go to market as a horizontal ERP, or as a vertical product for one industry cluster?
2. **Beachhead.** If vertical, which one — and what does that cost in build time before the first rupee of revenue?

Everything else in this document exists to support those two.

> **A note on evidence.** This document contains no market-size figures, adoption percentages, or revenue projections, because we do not have primary data and inventing it would make the case feel stronger than it is. Every quantitative claim that matters is listed in §11 as something to go and find out. The scorecard in §6 is explicitly a structured judgment, not a measurement, and should be re-scored after field validation.

---

## 2. Recommendation Summary

**Position vertically, build horizontally, sell one industry.**

- **Beachhead: bakery chains and central-kitchen food production.** Multi-outlet bakeries and QSR-style chains operating a central production kitchen with daily distribution to retail counters.
- **Entry point: the counter, not the kitchen.** Phase 1 as currently specified already covers the retail and distribution half of this vertical — POS, multi-warehouse, two-step transfers via TRANSIT, batch tracking with expiry and FEFO, GST, credit notes. Sell that half at go-live.
- **Expansion: the kitchen, in Phase 2.** Production planning, BOM with yield variance, the indent engine and crate tracking become the vertical overlay that converts Niyanthra from "another billing system" into a product with no direct competitor in this cluster.
- **Consequence for the roadmap:** BOM and production orders move from P3 to P2, and their tables enter Phase 1 as schema-only (P1-S) so the Phase 2 build is additive rather than a migration.
- **Do not build:** a bakery-specific fork. The overlay is a module set on the same tenant schema, sold as an entitlement.

The rest of this document justifies each of those five points and names what would falsify them.

---

## 3. Market Reality

Four characteristics of the Kerala and wider Indian SME market shape every decision downstream. They are constraints, not opinions.

### 3.1 The incumbent is not an ERP

SMEs in this segment do not replace one ERP with another. They run Tally for statutory accounting and GST filing, and everything else — stock, production, orders, approvals, daily coordination — happens in Excel, in paper registers, and in WhatsApp groups. The buyer is therefore not evaluating Niyanthra against SAP. They are evaluating it against a system that already works well enough for the one thing the owner is legally obliged to get right.

The strategic implication is severe and it points in exactly one direction: **do not attack accounting.** The accountant is the single most effective blocker of an ERP rollout, and the fastest way to create one is to tell them their ledger is moving. Niyanthra's FI module is architecturally necessary — the three invariants in Part 2 §2.4 depend on it, and stock valuation without a GL is fiction — but commercially it should be presented as internal machinery, not as a replacement for the customer's accounting practice.

### 3.2 Customers want customisation and will not pay for implementation

Owners expect software shaped to their specific workflow, and simultaneously resist setup fees and per-seat subscriptions. This combination kills the standard enterprise playbook, where implementation revenue funds the customisation.

The resolution is pre-configuration rather than customisation: a vertical template that arrives already shaped like the customer's business, so that "tailored to my workflow" and "live in a week" are the same claim rather than opposing ones. This is only achievable if the set of customers is narrow enough for one template to fit most of them — which is the core argument for verticalisation, independent of market size.

### 3.3 Digital literacy is uneven at the point of data entry

Kerala's general literacy is not the relevant variable. The relevant variable is whether a delivery driver at 5 AM, a counter staffer during the morning rush, or a warehouse hand will reliably enter accurate data into the system. If they will not, every downstream report is wrong, and the customer concludes the software failed.

This makes the floor-level interface a first-class product decision, not a UI detail. Anything a non-office worker must do should be a barcode scan, a WhatsApp message, or a single-screen mobile action — never a form in a desktop ERP.

### 3.4 Time-to-value is the real sales cycle

A configuration process measured in months loses the owner's attention before it produces anything. A target of under seven days from signature to first useful output is not a marketing claim; it constrains the architecture. Part 2 already gestures at this — FI-18 (opening balance import) is flagged as an under-estimated blocker for exactly this reason, and the recommendation there to promote it to the critical path is correct and should be read as a market requirement rather than a technical nicety.

---

## 4. Competitive Position

| Layer | Who occupies it | What they do well | Where they leave room |
|---|---|---|---|
| Statutory accounting | Tally Prime, Busy, Zoho Books | GST filing, accountant familiarity, near-universal installed base | No operational depth: no production, no perishability, no multi-location operational control |
| Light inventory + billing | Zoho Inventory, Vyapar, Marg | Cheap, fast to start, good mobile | Stock is a list, not a ledger; no valuation integrity, no batch/expiry discipline, no production |
| Open-source horizontal ERP | ERPNext and its implementation partners | Genuinely broad functional coverage at low licence cost | Implementation-dependent; generic UX; no vertical opinion; support quality varies by partner |
| Mid-market suites | SAP Business One, NetSuite, Oracle | Depth, credibility | Priced and scoped out of this segment entirely |
| Local custom shops | Regional software vendors | Will build exactly what the customer asks for | One-off codebases, no product roadmap, key-person risk, no compliance upkeep |

**Where Niyanthra actually sits.** Its architectural strengths — an append-only stock ledger where valuation is a stored consequence rather than a recomputed opinion, machine-checked invariants tying subledgers to the GL, gapless transactional document numbering — are real and rare in this price band. But they are invisible in a demo and impossible to sell on directly. No bakery owner buys a product because its stock ledger is immutable.

**This is the crux of the positioning problem.** The build quality is a retention and trust asset, not an acquisition asset. Something else has to do the acquiring. That something is domain specificity: features the customer recognises as being about *their* business, that no incumbent offers at any price they would consider.

A candid note on ERPNext specifically: Part 2's conventions — `docstatus` as 0/1/2, naming series, the SLE-carries-resulting-state contract, GRNI clearing — are close to ERPNext's model. That is a sound engineering choice and it is not a liability, but it does mean the horizontal feature list is not a differentiator against a free product with a decade of maturity. Verticalisation, a genuinely simple floor-level UX, and hands-on regional support are the differentiators. The positioning must rest on those.

---

## 5. The Central Tension: A Horizontal Product in a Vertical Market

Stated plainly, so it is not glossed over:

- Part 2 §3 specifies a horizontal ERP, and explicitly places **Manufacturing / BOM outside Phase 1** (INV-13 is P3; Appendix C lists it as "Not in P1").
- The market analysis concludes that a generic ERP in this segment has very low odds, and that the viable path runs through a specific industry cluster.
- Every high-potential Kerala cluster identified — spices, plywood, bakery, seafood, cashew — is **production-centric**. Their defining pain is conversion of raw material into finished goods with yield loss.

So the product as scoped for Phase 1 cannot serve the production half of any recommended vertical. This is not a flaw in Part 2; it is an unmade decision surfacing.

**The resolution is not to abandon the horizontal core.** Every vertical still needs GST invoicing, a real stock ledger, payables, receivables and a closing balance sheet. Building that once and reusing it across verticals is the only way this becomes a product company rather than a consultancy. The horizontal core is the platform; it is simply not the pitch.

**The resolution is to change what is sold, and to sequence the build against the vertical rather than against module completeness.** Concretely:

> Niyanthra is not "an ERP that also handles bakeries." It is "the system that runs a bakery chain," which happens to contain a full ERP underneath.

Two corollaries follow, and both are binding:

1. **Marketing, onboarding templates, demo data, terminology and the entire first-run experience are vertical.** The customer should see *indents*, *batches*, *crates* and *outlets* — not *purchase requisitions*, *stock entries*, *containers* and *warehouses*.
2. **Roadmap priority is set by vertical completeness, not module completeness.** A half-built feature that closes the bakery loop outranks a fully-built feature that no launch customer uses. Payroll is the clearest example: Part 2 §3.6.2 already makes it conditional, and this document strengthens that to "out unless a signed customer blocks on it."

---

## 6. Vertical Selection Framework

### 6.1 Criteria and weights

| # | Criterion | Weight | What it measures |
|---|---|---|---|
| C1 | Cluster density and reachability | 15 | Can we reach 50 qualified prospects without building a sales team? Are they geographically concentrated? |
| C2 | Pain intensity | 20 | What does the status quo cost them daily, in money they can see? |
| C3 | Fit with the Phase 1 build | 20 | How much engineering stands between today's spec and a first sale? |
| C4 | Willingness and ability to pay | 15 | Margin structure, cash position, existing software spend |
| C5 | Reference effect within the cluster | 10 | Do operators in this industry talk to each other and copy each other? |
| C6 | Defensibility vs Tally / Zoho / ERPNext | 10 | How hard is this to replicate with a configuration of an existing product? |
| C7 | Expansion ceiling beyond Kerala | 10 | Does the same product sell in Coimbatore, Bengaluru, Hyderabad? |

C2 and C3 carry the heaviest weights deliberately. C2 determines whether anyone buys; C3 determines whether we survive long enough to find out.

### 6.2 Scorecard

Scores are structured founder judgment based on the cluster characteristics described in the source analysis and the Phase 1 scope in Part 2. They are a decision aid, not evidence. Re-score after §11.

| Vertical | C1 (15) | C2 (20) | C3 (20) | C4 (15) | C5 (10) | C6 (10) | C7 (10) | **Total** |
|---|---|---|---|---|---|---|---|---|
| **Bakery & central kitchen** | 13 | 18 | 11 | 11 | 9 | 9 | 9 | **80** |
| Plywood & hardware (Perumbavoor) | 13 | 16 | 8 | 12 | 8 | 8 | 5 | **70** |
| Textile & apparel retail | 12 | 13 | 13 | 9 | 6 | 5 | 8 | **66** |
| Seafood processing & export | 10 | 17 | 5 | 13 | 7 | 9 | 5 | **66** |
| Spices & agro-processing | 9 | 17 | 6 | 12 | 6 | 9 | 6 | **65** |
| Cashew processing (Kollam) | 10 | 15 | 6 | 8 | 6 | 8 | 4 | **57** |
| Hospitality & Ayur-wellness | 11 | 12 | 7 | 10 | 6 | 4 | 7 | **57** |

### 6.3 Notes on the runners-up

**Plywood & hardware** scores well on pain and on cluster density — the Perumbavoor belt is about as concentrated as an industrial cluster gets in Kerala, which makes reference selling unusually efficient. It loses on two counts. Cutting-yield and timber-grading logic is a deeper and more idiosyncratic build than bakery production, and the expansion ceiling is low: the cluster's density is exactly what makes it geographically bounded. It is the strongest second choice and should be held as the Phase 3 vertical.

**Textile & apparel retail** has the best Phase 1 fit of any candidate — it needs matrix inventory (size × colour × fabric as item variants), which is a contained schema addition rather than a new module. It fails on defensibility. This is the most heavily contested retail software segment in India, with multiple mature specialists, and nothing in Niyanthra's architecture wins that fight.

**Seafood and spices** both have severe, genuinely unsolved pain and customers with export revenue and real budgets. Both are ruled out for now on C3 alone: grade-based pricing, moisture-loss accounting, consignment purchase structures and export documentation constitute an entirely separate product. Revisit once the production overlay exists, since yield and batch tracing are shared foundations.

**Hospitality and Ayur-wellness** requires a booking and scheduling engine that shares almost nothing with the current spec, and competes against established property-management systems. Lowest strategic fit.

---

## 7. Recommendation and Rationale

### 7.1 The recommendation

**Adopt bakery chains and central-kitchen food production as the beachhead vertical, entered in two stages.**

**Stage 1 — sell the outlet side at Phase 1 go-live (Playbook Month 4).**

What a multi-outlet bakery needs at the retail end maps onto the current Phase 1 scope almost exactly:

| Bakery need | Phase 1 capability (Part 2 reference) |
|---|---|
| Counter billing across outlets | POS module, session + cash variance |
| Each outlet as a stock location | INV-02 multi-warehouse, hierarchical |
| Kitchen → outlet dispatch with in-transit accountability | INV-08 two-step transfer via `TRANSIT` warehouse type (INV-03) |
| Shelf-life control on perishables | INV-06 batch tracking with expiry, FEFO picking |
| Transport damage and shortage claims | SD-07 credit note / sales return, INV-11 variance posting |
| Unsold-stock write-off at end of day | INV-11 stock take with variance to Stock Adjustment |
| GST compliance across outlets | SD-06, SD-12, GSTR-1/3B |

This means a bakery chain is sellable at go-live without waiting for the production module. The pitch at this stage is outlet-level control: what each counter sold, what it holds, what expired, and what went missing between the kitchen and the shelf. That alone is more than Tally plus Excel provides, and it is enough for a paid pilot.

**Stage 2 — the production overlay in Phase 2 (Months 5–8).**

The overlay is the differentiator and the retention mechanism, built directly from the 24-hour operational loop already documented:

| Overlay capability | What it does |
|---|---|
| Indent engine | Outlets submit next-day demand against a system-suggested baseline; hard cut-off locks indents so production planning is stable |
| Production plan + BOM | Aggregates locked indents per SKU, explodes to raw material requirement, issues a picking list |
| Yield variance tracking | Standard vs actual yield per batch; deviation surfaces recipe drift, over-issue, or loss |
| Batch and expiry | Batch ID with manufacture time, operator, best-before — already P1 in INV-06 |
| Crate tracking | Reusable crates as returnable assets with QR identity and movement history |
| Receiving reconciliation | Scan-in at the outlet, damage and shortage logged at the gate, credit note raised to the kitchen automatically |
| Waste analytics | Return and expiry data feeds back into the next day's indent suggestion |

### 7.2 Why bakery wins

**Pain is daily, visible, and denominated in rupees.** Perishability means the cost of poor planning is destroyed inventory, every single day, in a form the owner can physically see in the returns crate. Most operational software must argue its ROI. Here the customer already knows the number and is already angry about it.

**The cluster is dense and highly reference-driven.** Kerala's bakery chains are numerous, geographically concentrated in and around Thrissur, Ernakulam and Kozhikode, frequently family-run, and their operators know each other. The cluster's structural habit of copying what works is precisely what makes a beachhead strategy compounding rather than linear.

**No incumbent competes on this ground.** Tally cannot model an indent. Zoho cannot track yield loss. Retail POS products stop at the counter and have nothing to say about a 2 AM production run. ERPNext has manufacturing, but no bakery opinion — it offers a generic BOM that someone must configure into a bakery, which is a consulting engagement, not a product. The overlay is genuinely uncontested.

**The build is reusable.** BOM, yield variance, batch genealogy and production orders are the shared foundation of every other production vertical on the shortlist — plywood, spices, seafood, cashew. Building it for bakery is not a detour; it is the prerequisite for the vertical after this one. This is the single strongest argument against choosing textile, where the matrix-inventory build transfers to nothing else on the list.

**Floor-level UX is forced, correctly.** Bakery operations have several non-office data entry points — the 6 PM indent, the 5 AM crate load, the 7 AM gate receipt, the end-of-day return. Designing for those is exactly the discipline §3.3 demands, and the vertical will not tolerate shortcuts on it.

**Expansion is unbounded.** Every Indian city has multi-outlet bakeries, and the same operational model describes cloud kitchens, sweet shops, dairy and QSR chains. Unlike Perumbavoor plywood, the second market is not geographically constrained.

### 7.3 What we are accepting by choosing this

Stated honestly, because a decision document that only lists advantages is advocacy, not analysis:

- **C3 is the weakest score for a reason.** The full vertical requires a module the current plan defers to Phase 3. Stage 1 mitigates this but does not eliminate it — the compelling product does not exist until Month 8.
- **Willingness to pay is mid-range.** Bakeries run thinner margins than seafood or spice exporters. Pricing must be metered per outlet so that cost scales with the customer's own scale, and it must stay well under the value of prevented waste.
- **Food safety compliance raises the stakes.** FSSAI expectations around batch traceability and labelling mean errors have regulatory consequence, not just commercial. Batch tracking must be treated as a hard go-live blocker for these customers, not the conditional item Appendix C currently lists.
- **The 2 AM to 7 AM operating window is unforgiving.** If the system is unavailable during dispatch, the business physically stops. Reliability expectations are closer to retail POS than to back-office ERP, and support hours must reflect that.

---

## 8. Consequences for Parts 1 and 2

If the recommendation is approved, these changes are binding on the companion documents and should be made in the same revision cycle.

### 8.1 Roadmap changes (Part 1 — Playbook)

| # | Change | Rationale |
|---|---|---|
| R1 | Move **BOM and production orders (INV-13) from P3 to P2**, scoped narrowly: single-level BOM, batch scaling, standard-vs-actual yield variance. **Not** full MRP, multi-level BOM, capacity planning or routing. | The vertical is not viable without it; the narrow scope is what keeps it deliverable in Phase 2 |
| R2 | Add **Indent Engine** as a Phase 2 module with its own owner | It is the entry point of the entire daily loop and the feature customers will describe to each other |
| R3 | Add **Tally two-way sync** as a named Phase 2 deliverable | §3.1: the accountant must not be threatened. This is currently absent from both companion documents and is a commercial requirement, not an integration nicety |
| R4 | Add **WhatsApp interaction layer** (indent submission, dispatch summary, approval) to Phase 2 | §3.3: the answer to uneven floor-level digital literacy. SD-11 already establishes WhatsApp share plumbing to build on |
| R5 | Promote **opening balance import (FI-18)** to the critical path | Already recommended in Part 2 Appendix C; §3.4 makes it a market requirement |
| R6 | Reclassify **POS offline mode** from "not in P1" to a **known go-live risk with a documented manual fallback** | A counter that cannot bill during an outage is an unacceptable failure mode in this vertical. Either ship a queue-and-sync mode or make the fallback procedure an explicit contractual term |
| R7 | Confirm **payroll is out of Phase 1** unless a signed customer blocks on it | Strengthens Part 2 §3.6.2; protects Phase 2 capacity for the overlay |

### 8.2 Schema changes (Part 2 — Functional Spec)

Per Part 2's own P1-S principle — schema and foreign keys exist in Phase 1 so that Phase 2 is additive rather than a destructive migration — the following tables should be **migrated in Phase 1 with no UI**:

| Table group | Tables | Phase |
|---|---|---|
| Production | `bom`, `bom_item`, `production_order`, `production_batch`, `yield_variance` | **P1-S** |
| Indenting | `indent`, `indent_item`, `indent_cutoff_policy` | **P1-S** |
| Returnable assets | `crate`, `crate_movement` | **P1-S** |
| Vertical config | `vertical_template`, `module_entitlement` extension | **P1** |

Two further specification-level changes:

- **`batch.best_before_at` as `TIMESTAMPTZ`, not `DATE`.** Fresh cream products expire in hours. A date-granular expiry field is a schema decision that cannot be corrected later without a migration on the busiest table in the vertical.
- **Batch tracking (INV-06) becomes an unconditional go-live blocker** in Appendix C rather than conditional, on the grounds that every launch customer in the chosen vertical is a food business.

### 8.3 Terminology layer

Add a vertical terminology mapping to the `vertical_template` configuration, so that the same schema presents industry language. This is a presentation-layer lookup, not a fork.

| Core term | Bakery presentation |
|---|---|
| Warehouse | Outlet / Kitchen store |
| Stock Transfer | Dispatch |
| Purchase Requisition | Indent |
| Batch | Production batch |
| Stock Reconciliation | End-of-day count |
| Credit Note (inward) | Damage / shortage claim |

---

## 9. Commercial Model

Structural recommendations. Every number below is a **hypothesis to be tested in §11**, not a price.

**Metering unit: per outlet, per month, with the central kitchen included in the base.** This aligns cost with the customer's own scale, is trivially explainable, and grows automatically as the chain expands — which is the only expansion revenue that does not require a second sale.

**No implementation fee.** §3.2 makes upfront setup charges the largest single source of friction. Onboarding cost is absorbed and recovered through subscription duration, which requires that onboarding be genuinely cheap — hence the seven-day target and the vertical template.

**Pilot structure:** one chain, three months, heavily discounted, with a written success definition agreed in advance (see kill criteria). A pilot without a pre-agreed success metric becomes an indefinite free trial.

**Land-and-expand sequence:** counter billing and outlet stock first, production overlay second, financial reporting third. The customer should be paying before they meet the FI module.

**Channel consideration:** accountants and regional Tally partners are the highest-leverage referral channel in this market, and R3 (Tally sync) is what makes Niyanthra a complement rather than a threat to them. Worth testing in §11 alongside direct outreach.

---

## 10. Risks and Kill Criteria

| Risk | Consequence | Mitigation |
|---|---|---|
| Stage 1 is not compelling enough to close a paid pilot without the production overlay | Eight months of build before first revenue | Validate §11.1 before committing engineering. If outlet-side control alone does not close a pilot, reconsider sequencing rather than the vertical |
| Phase 2 overlay slips | The differentiator arrives after pilot patience expires | R1's deliberately narrow BOM scope; protect Phase 2 capacity by holding R7 |
| Floor staff do not adopt the indent and receiving flows | Data quality collapses, all reporting becomes untrustworthy, customer blames the product | R4 WhatsApp layer; usability testing with actual outlet staff, not owners; scan-based entry everywhere |
| Bakery chains prove too price-sensitive to sustain per-outlet pricing | Unit economics fail at scale | Validate in §11.2 before pricing is published |
| A single bad reference in a tight cluster | Reference effect runs in reverse | Over-resource the first two customers; treat them as product development, not accounts |
| The horizontal core absorbs capacity the vertical needs | Never reach vertical completeness | Roadmap priority set by vertical completeness (§5, corollary 2) |

### Kill criteria

Pre-committing to these prevents the sunk-cost drift that keeps a failing beachhead alive. Reopen the vertical selection decision if:

- **K1.** After 20 structured conversations with bakery operators, fewer than 8 identify daily indent chaos or perishable waste as a top-three operational problem.
- **K2.** No bakery chain agrees to a paid pilot on Stage 1 scope by the end of Playbook Month 5.
- **K3.** After two full pilots, indent submission compliance from outlet staff is below 80% of operating days without manual chasing.
- **K4.** By Month 9, no pilot customer can point to a measurable reduction in waste or stock variance attributable to the system.

K1 is answerable in weeks and should be answered before R1 and R2 are committed to the roadmap.

---

## 11. Validation Plan Before Commitment

The recommendation above is a well-reasoned hypothesis built on secondary analysis. The following converts it into a decision grounded in evidence. All of it is field work, none of it requires code.

### 11.1 Demand validation — Weeks 1–3

- **20 structured interviews** with bakery chain owners and operations managers across at least three districts. Target mixed chain sizes (3–5 outlets, 6–15, 15+) to find where the pain threshold sits.
- Fixed question set covering: how indents are collected today; daily unsold and expired quantity; frequency of transport damage disputes; what software they currently run; what they currently pay for it; who decides on a purchase.
- **Specific test for Stage 1:** describe outlet-side control *without* mentioning production planning, and record whether it alone generates interest. This single answer validates or invalidates the entire two-stage sequencing.

### 11.2 Commercial validation — Weeks 2–4

- Establish current software spend per chain and per outlet.
- Test the per-outlet pricing structure directly, at three price points.
- Identify the actual decision-maker and the actual blocker. The working assumption is owner-decides, accountant-blocks; confirm it.

### 11.3 Cluster sizing — Weeks 1–4

- Count addressable chains: multi-outlet bakeries with a central kitchen, by district. Sources: FSSAI licence registers, bakery and food business associations, distributor networks.
- This produces the first defensible market-size number in the project.

### 11.4 Competitive check — Week 3

- Confirm no regional vendor already serves this cluster with a bakery-specific product. If one exists, understand its pricing, penetration and weaknesses before proceeding.

### 11.5 Technical validation — Week 4

- Spend a full operating cycle inside one central kitchen, from 6 PM indent to 7 AM gate receipt.
- Confirm the 24-hour loop as documented matches reality, and capture where it diverges. The indent cut-off in particular is likely to be softer in practice than the model assumes, and the product must handle the exception path.

**Gate:** §11 completes before R1 and R2 are committed. Engineering continues on Phase 1 horizontal scope throughout, since nothing in Phase 1 is wasted under any outcome of this validation.

---

## 12. Document Control

| | |
|---|---|
| **Supersedes** | Nothing; first issue |
| **Companions** | *Niyanthra ERP — Implementation Roadmap & Team Playbook v1.0* (Part 1); *Functional Module & Data Architecture Specification v1.0* (Part 2) |
| **Status** | Draft for decision |
| **Decision owner** | Founders |
| **Decision deadline** | End of Playbook Month 1 |
| **Change control** | Any change to the beachhead vertical requires a revision of this document and a corresponding review of Part 1 §Critical Path and Part 2 Appendix C |
| **Review cadence** | Re-score §6.2 after §11 completes; review at each phase boundary |
| **Open items** | (1) All of §11; (2) approval of R1–R7; (3) confirmation that P1-S tables in §8.2 can be migrated within Phase 1 without schedule impact; (4) decision on whether Tally sync is Phase 2 or accelerated |
