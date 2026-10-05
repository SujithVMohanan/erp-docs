# Audit Trail

> **Simplify. Manage. Grow.**

Every consequential action is attributable to a user (or an AI action acting **as** a user). Audit is not optional logging in controllers; document services write it.

Related: [Document Engine](document-engine.md) · [AI Architecture](ai-architecture.md)

---

## 1. Header stamps

On the document (see document engine):

```text
created_by / created_at
updated_by / updated_at
submitted_by / submitted_at
approved_by / approved_at
rejected_by / rejected_at
cancelled_by / cancelled_at
posted_by / posted_at
created_by_type = HUMAN | AI
ai_assistance
ai_action_id
```

---

## 2. Field-level history

### `audit_field_changes`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `tenant_id` | BIGINT | |
| `company_id` | BIGINT | |
| `document_type` | VARCHAR(50) | |
| `document_id` | BIGINT | |
| `field_name` | VARCHAR(80) | |
| `old_value` | TEXT NULL | JSON-serialized |
| `new_value` | TEXT NULL | |
| `reason` | TEXT NULL | |
| `changed_by` | BIGINT | |
| `changed_at` | TIMESTAMP | |
| `ai_action_id` | UUID NULL | |

Capture amount, party, qty, warehouse, status, and tax fields. Skip noisy updated_at-only writes.

---

## 3. Action log

### `audit_actions`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | BIGINT PK | |
| `company_id` | BIGINT | |
| `document_type` | VARCHAR(50) NULL | |
| `document_id` | BIGINT NULL | |
| `action` | VARCHAR(40) | `create` · `update` · `submit` · `approve` · `post` · `void` · … |
| `actor_user_id` | BIGINT | Always a real user (AI runs as the requesting user) |
| `created_by_type` | VARCHAR(10) | `HUMAN` · `AI` |
| `ai_action_id` | UUID NULL | |
| `payload` | JSONB NULL | Non-sensitive summary |
| `created_at` | TIMESTAMP | |

Permission denials may be logged separately at security level (not this table’s primary job).

---

## 4. AI correlation

### `ai_actions`

| Field | Type | Notes |
| ----- | ---- | ----- |
| `id` | UUID PK | = `ai_action_id` |
| `company_id` | BIGINT | |
| `user_id` | BIGINT | Requester |
| `intent` | VARCHAR(80) | |
| `tool_name` | VARCHAR(80) | |
| `prompt_hash` | VARCHAR(64) | Do not store raw secrets |
| `result_summary` | TEXT | |
| `created_document_type` | VARCHAR(50) NULL | |
| `created_document_id` | BIGINT NULL | Draft id |
| `created_at` | TIMESTAMP | |

If AI cannot identify a product/customer, no document is created; the action still logs the clarification.

---

## 5. Retention and access

Audit rows are append-only. Permission `platform.audit.view`. No update/delete APIs for business users.

---

<p align="center"><sub>Niyanthra ERP — Internal Architecture Documentation</sub></p>
