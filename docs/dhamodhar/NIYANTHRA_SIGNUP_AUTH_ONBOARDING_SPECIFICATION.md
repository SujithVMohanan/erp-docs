# 🚀 Niyanthra — Sign-Up & Onboarding UX Specification

**Version:** 1.0
**Date:** September 2026
**Document Series:** Part 3 — companion to *Market Strategy & Vertical Selection* (Part 0), *Implementation Roadmap & Team Playbook* (Part 1), *Functional Module & Data Architecture Specification* (Part 2)
**Author role:** Principal PM / UX Architect
**Objective:** Replace a form-heavy setup wizard with an instant-provisioning, product-led growth (PLG) flow. Target: **signed-up and inside a working tenant in under 60 seconds**, everything else deferred to in-app setup.

---

## 0. Design Principle

> **Authenticate → Provision → Explore.** Never **Authenticate → Configure → Explore.**

Every field that isn't strictly required to create a legally-addressable tenant record is moved from the sign-up funnel into the in-app checklist. This is a direct application of Part 0 §3.2 and §3.4: Kerala SME buyers abandon anything that smells like an implementation project, and time-to-value has to be measured in minutes, not weeks. GSTIN, branches, team members, module selection — none of it blocks Step 3.

---

## 1. The Sign-Up Funnel (3 Steps Max)

### Step 1 — Authentication

**Fields:** Email *or* phone, password — **or** one-tap Google SSO.

| Rule | Why |
|---|---|
| Phone accepted as a first-class identifier, not just a recovery field | Trader/distributor segment is phone-primary; email is often a shared or rarely-checked address |
| Google SSO pre-fills name + email, skips password creation entirely | Removes the highest-drop-off field in any form: password creation with strength rules |
| OTP (SMS or WhatsApp) instead of email verification when phone is chosen | Email verification click-through in this segment is unreliable; OTP closes in seconds |
| No CAPTCHA on this step | Push bot mitigation to rate-limiting and device fingerprinting server-side, not a UX tax on every real user |
| Password fields (if used): 1 field, real-time strength meter, no "confirm password" repeat | Confirm-password fields measurably increase abandonment for negligible error-catching value |

**Exit condition:** a verified `user` row exists. No `company` exists yet.

### Step 2 — Instant Workspace Creation

**Fields, and only these:**

1. **Company Name** (free text)
2. **State** (dropdown, pre-filtered to the 36 Indian states/UTs, Kerala pinned to the top given launch geography per Part 0)

That's it. No industry picker, no employee count, no GSTIN, no address, no phone-again. Industry vertical (Part 0 §8.3's `vertical_template`) is inferred later from in-app behaviour and the setup checklist — never asked here, because a dropdown of business types is exactly the kind of "which box am I" friction that stalls a first-time user.

| What's deliberately **not** here and why |
|---|
| GSTIN — validated later; many sign-ups are pre-registration or the person signing up isn't the one who has it memorized |
| Business address — derivable later from the first branch/GST registration; asking twice is the anti-pattern to avoid |
| Company size / turnover — used for nothing at provisioning time; asking unused questions signals a form, not a product |
| "How did you hear about us" — send to a post-activation survey, never the funnel |

**Exit condition:** the moment the user submits this step, provisioning runs synchronously in the background (§2) and the user lands **directly** in a live workspace — not a "setting up your account, please wait" interstitial longer than a couple of seconds. If provisioning genuinely can't complete in under ~2 seconds, show a skeleton dashboard immediately and let modules populate asynchronously rather than blocking on a spinner.

### Step 3 — Dashboard Access + In-App Setup Checklist

The user is now inside a real, working tenant — not a demo, not a sandbox. A persistent, dismissible **Setup Checklist** panel (sidebar or a docked card) replaces every step that used to be pre-entry:

| Checklist item | Trigger | Blocking? |
|---|---|---|
| ✅ Workspace created | Auto-complete on entry | — |
| Add GSTIN / GST registration | Opens `company_gst_registration` form (Part 2 §7.2) inline | **Not blocking.** Billing screens work with GST fields empty until the first tax-bearing invoice is submitted, at which point the system prompts contextually |
| Toggle modules you need | Simple on/off switches — POS, Purchase, HR, etc. — mapped to `module_entitlement` (Part 0 §8.2) | Not blocking; sensible defaults are pre-enabled (SD, INV, FI, POS) based on the P1 launch scope |
| Invite your team | Opens the invite panel (§3) | Not blocking |
| Import opening stock / customers | Bulk import (Part 2 FI-18, flagged there as onboarding-critical) | Not blocking at sign-up, but surfaced prominently — this is the one checklist item most correlated with activation |
| Connect Tally (if applicable) | Optional, per Part 0 R3 | Not blocking |

**Rule:** nothing in Step 3 ever *prevents* the user from clicking around, creating a test invoice, or exploring a module. The checklist informs; it never gates. The only true gate in the entire product is GST validation *at invoice submission*, which already exists in Part 2 (§ "Validation is deliberately enforced at invoice submit, not only at item creation").

---

## 2. Tenant & State Initialization (Behind Step 2)

When the user submits Company Name + State, the following happens synchronously in a single transaction, mapped directly onto Part 2's existing multi-tenant architecture:

```
POST /api/v1/onboarding/provision
{
  "company_name": "Kerala Traders & Sons",
  "state_code": "KL"
}
```

**1. Tenant record creation**
- A new `company` row is created (Part 2 §2: *"Company — a tenant... `company_id` is the tenancy boundary"*). This is a **row**, not a new database or schema — Niyanthra is shared-schema multi-tenant with RLS, so "instant" is genuinely instant: no infra provisioning step exists to wait on.
- A default `branch` row is created as the company's primary location, `state_code` set from Step 2. This becomes the seed for the first `company_gst_registration` once GSTIN is added later — the state is captured now specifically so that placeholder is never a blank, unresolvable field.

**2. Default master data seeding**
- Chart of accounts: the platform default template (`company_id IS NULL` rows) is copied in per Part 2 §"A company inherits the platform default template at creation". This is already the existing mechanism — onboarding just triggers it earlier and silently, with no "set up your accounting" screen shown to the user.
- Tax rules: nothing is copied per-tenant. `compliance_parameter` (GST rates, HSN, state codes, Professional Tax slabs) lives in the read-only Master DB per Part 2 §6, versioned by `effective_from`/`effective_to`. The new tenant simply reads from it — there is no seeding step, which is precisely why this scales to instant provisioning.
- State-linked defaults: the `state_code` from Step 2 resolves `place_of_supply` defaults and Professional Tax jurisdiction (Part 2 §6.8, §"State-specific slabs") the moment they're needed, with no separate configuration screen.

**3. RBAC bootstrap**
- The signing-up user is stamped as the tenant's sole member with an **Owner/Admin** role (§3) on the same `company_id`. RLS policies (Part 2 §4) are active from the first write — there is no "trusted setup window" where isolation is off.

**4. What is explicitly deferred, not defaulted**
- GSTIN and `company_gst_registration`: absent until the user adds one. Invoicing works in a GST-pending state; only *tax-bearing* actions require it.
- Module entitlements beyond the P1 default set: off until toggled.
- Any second branch, warehouse, or user: created on demand from inside the app.

**Target: this entire sequence completes in under 2 seconds**, because it is row inserts against an existing schema, not schema creation. This is a direct dividend of Part 2's shared-schema-with-RLS decision over a schema-per-tenant or database-per-tenant model — worth stating explicitly, because it's the architectural fact that makes "instant workspace" a true claim rather than a marketing one.

---

## 3. User Provisioning & Role Management (Post-Sign-Up)

All of this happens from **Settings → Team**, never from the sign-up funnel.

### 3.1 Inviting users

| Step | Detail |
|---|---|
| Admin clicks "Invite team member" | Enters phone or email + selects a role from a fixed list (below) |
| Invite sent via WhatsApp or SMS link (email optional, not primary) | Consistent with Part 0 R4 — this segment's staff are phone/WhatsApp-first, and an email invite to a cashier is likely to sit unread |
| Invitee taps link → sets a password (or uses Google SSO) → lands **directly** inside the tenant, no company-creation step | They join an existing `company_id`; Step 2 of §1 never runs for them |
| No seat limit blocks this during trial | Charging by outlet/seat (Part 0 §9) is a billing-time concern, not an invite-time gate — gating invites during a trial kills the "get your team in and try it together" motion that drives PLG conversion |

### 3.2 Role model at launch

A simple, fixed role set ships first — not a permissions matrix builder. Granular, per-document permissioning (the 5-layer permission model referenced in Part 2 §PM-03, e.g. approval-threshold routing) sits *underneath* these roles as the system matures, but the user never has to configure it directly.

| Role | Typical user | Access |
|---|---|---|
| **Owner** | Business owner (the sign-up user) | Full access, all modules, billing, can delete the workspace |
| **Admin** | Ops manager | Full access to enabled modules, cannot manage billing or delete the workspace |
| **Accountant** | In-house or outsourced accountant | Full FI module (ledgers, GST returns, reports); read-only on INV/SD; no POS access |
| **Manager** | Branch/outlet manager | Full access to SD, INV, POS *for their assigned branch(es) only*; no FI |
| **Cashier / Counter Staff** | Billing counter | POS only: create sale, apply pre-configured discounts, view own session — no rate override (`sd.override_rate` stays admin-only per Part 2), no reports, no other branches |
| **Staff (view-only)** | Warehouse/floor staff needing visibility, not entry | Read-only on assigned module(s); used for the barcode/scan-based data entry flows from Part 0 §3.3, where the *action* (scan a crate) is permitted but broader navigation isn't |

### 3.3 How RBAC is enforced

- **Row-level**, not just menu-level. A Manager scoped to "Outlet A" doesn't just have Outlet A's menu items hidden elsewhere — their queries are filtered by `branch_id` server-side, the same defense-in-depth pattern Part 2 §4 uses for `company_id` isolation. A UI toggle is not the security boundary; the query filter is.
- **Role assignment is per-branch where relevant.** A Manager or Cashier is invited *to* a branch, not just to the company — this matters immediately for the multi-outlet bakery/retail vertical in Part 0, where a Kochi outlet manager should never see Thrissur's stock or sales.
- **Admin can change or revoke roles at any time** from the same Team panel; a revoked user loses access on next request (session-checked, not just login-time), consistent with the RLS-enforced, non-cacheable-trust posture Part 2 establishes elsewhere.
- **No self-service role escalation.** A Cashier cannot invite an Accountant. Only Owner/Admin issue invites, which keeps the trial-phase "invite freely" policy in §3.1 safe.

---

## 4. Conversion & Instrumentation Notes

- **Funnel target:** Step 1 → live dashboard in under 60 seconds for the median user, under 2 seconds of that being backend provisioning (§2).
- **Track drop-off per step, not just overall conversion.** Step 1 abandonment signals an auth-friction problem; Step 2 abandonment (unlikely, given only two fields) signals trust/legitimacy concerns worth a separate look; a high bounce *inside* the app post-Step-3 signals the checklist itself, not the funnel, is the problem.
- **The real activation metric isn't sign-up completion — it's first invoice or first stock entry.** Instrument the checklist items in §1 individually; Part 0 §11 already flags opening-balance import as the item most correlated with whether a trial converts, and that pattern should hold for the funnel metrics too.
- **A/B surface for later, not launch:** whether Step 2 asks for State at all, versus inferring it later from the first GST registration. Keeping it in Step 1 is recommended for now because it's the one field that meaningfully personalizes Step 3 (Professional Tax jurisdiction, Kerala-pinned dropdown), but it's a legitimate two-field-vs-one-field test once there's traffic to test it on.

---

## 5. Document Control

| | |
|---|---|
| **Companions** | Part 0 (Market Strategy & Vertical Selection), Part 1 (Implementation Roadmap & Team Playbook), Part 2 (Functional Module & Data Architecture Spec) |
| **Depends on** | Part 2 §2 (tenancy model), §4 (RLS/composite FK pattern), §6 (compliance_parameter, GST registration); Part 0 §8.2–8.3 (module_entitlement, vertical_template), R3/R4 (Tally sync, WhatsApp layer) |
| **Status** | Draft for review |
| **Open items** | (1) Confirm 2-second provisioning target is achievable under real DB load, not just idle-dev conditions; (2) decide whether WhatsApp invite requires Business API approval lead time before launch; (3) finalize whether "Staff (view-only)" is P1 or deferred — check against Part 2 Appendix C phasing |
