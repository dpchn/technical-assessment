# Unified PostgreSQL Schema Proposal

## Design Principles

1. **Service-agnostic core** -- A single `services` table replaces all `dtv_*` prefixed fields. New visa types, compliance products, or geographies are rows, not schema changes.
2. **Append-only audit** -- Every mutation that matters (staff review, AI verdict, payment event, assignment change) lands in a dedicated audit or history table with actor + timestamp.
3. **Hard-delete GDPR** -- `accounts.deleted_at` soft-deletes for grace period; a scheduled job physically purges the row and all FK-cascaded children after confirmation.
4. **Multi-region ready** -- `region` column on `services` + service-type registry keyed by geography. No hard-coded country logic in the schema.
5. **Unlinked leads** -- Conversations can exist without an account FK. The link is optional and made explicit when known.


---

## Table Definitions

### 1. `accounts`

The canonical user record. One row per client.

```
accounts
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
auth_provider_id    TEXT  NOT NULL UNIQUE       -- Firebase UID or future auth provider ID
email               TEXT                         -- primary email
first_name          TEXT
last_name           TEXT
nationality         TEXT                         -- ISO 3166-1 alpha-2
referral_code       TEXT  UNIQUE                 -- this user's referral code
referred_by         UUID  FK(accounts.id)        -- who referred them
client_source       TEXT                         -- e.g. 'client_referral', 'instagram'
expo_token          TEXT                         -- push notification token
is_ghost            BOOLEAN NOT NULL DEFAULT false
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
deleted_at          TIMESTAMPTZ                  -- soft-delete for GDPR grace period
```

**Indexes:** `auth_provider_id` (unique), `email`, `nationality`, `referral_code` (unique), `deleted_at` (partial, WHERE deleted_at IS NOT NULL for purge jobs).

**Rationale:** All DTV-specific fields (`dtv_purpose`, `dtv_submission_country`, etc.) move to the `services` table. The account is identity-only. `user_id` redundancy is eliminated. `rejected_timestamp` empty-string hack is replaced by proper NULLable timestamps on services.

---

### 2. `account_channels`

Communication channels per account. One row per channel type.

```
account_channels
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
account_id          UUID  NOT NULL FK(accounts.id) ON DELETE CASCADE
channel_type        TEXT  NOT NULL               -- 'email', 'whatsapp', 'line', 'personal_email'
channel_value       TEXT  NOT NULL               -- the identifier/address
is_preferred        BOOLEAN NOT NULL DEFAULT false
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
UNIQUE(account_id, channel_type, channel_value)
```

**Rationale:** Replaces the flat `channels` object. Extensible to new channel types without schema changes. `is_preferred` replaces `channel_preference`.

---

### 3. `account_platforms`

Tracks which client-facing platforms the user has authenticated on.

```
account_platforms
─────────────────────────────────────────────────
account_id          UUID  NOT NULL FK(accounts.id) ON DELETE CASCADE
platform            TEXT  NOT NULL               -- 'web', 'ios', 'android'
first_seen_at       TIMESTAMPTZ NOT NULL DEFAULT now()
last_seen_at        TIMESTAMPTZ NOT NULL DEFAULT now()
PRIMARY KEY (account_id, platform)
```

---

### 4. `service_types` (Registry)

Configuration-driven catalog of available services. Adding a new visa type or compliance product = inserting a row, not a migration.

```
service_types
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()                     
display_name        TEXT  NOT NULL              -- e.g. 'dtv', '90day_report', 'work_permit', 'marriage_permit' (ENUM)
description         TEXT
category            TEXT  NOT NULL               -- 'visa', 'compliance', 'tax', 'permit'
region              TEXT                         -- ISO country code or NULL for global
is_active           BOOLEAN NOT NULL DEFAULT true
config              JSONB NOT NULL DEFAULT '{}'  -- validation rules, SLA thresholds, etc.
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Rationale:** `config` JSONB holds service-specific settings (SLA days, required fields, etc.) that vary per type but don't warrant their own columns. The schema doesn't need to know what a "DTV" is.

---

### 5. `document_type_requirements` (Registry)

Defines which documents are required for each service type.

```
document_type_requirements
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
service_type_id     TEXT  NOT NULL FK(service_types.id)
doc_type_key        TEXT  NOT NULL               -- e.g. 'passport', 'financial_assets', 'accommodation'  (ENUM)
display_name        TEXT  NOT NULL
is_required         BOOLEAN NOT NULL DEFAULT true
sort_order          INT  NOT NULL DEFAULT 0
validation_rules    JSONB                        -- AI validation config
UNIQUE(service_type_id, doc_type_key)
```

**Rationale:** Decouples document requirements from the schema. When ISSA launches work permits in a new country, the team inserts rows here -- no migration needed. The old `dtv_passport` key becomes service_type=`dtv`, doc_type_key=`passport`.

---

### 6. `step_definitions` (Registry)

Defines the workflow steps for each service type.

```
step_definitions
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
service_type_id     TEXT  NOT NULL FK(service_types.id)
step_key            TEXT  NOT NULL               -- e.g. 'begin', 'upload', 'submit', 'approved'  (ENUM)
display_name        TEXT  NOT NULL
sort_order          INT  NOT NULL DEFAULT 0
UNIQUE(service_type_id, step_key)
```

---

### 7. `services`

A single user application/engagement. One user can have many concurrent services (two DTVs, a 90-day report, etc.).

```
services
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
account_id          UUID  NOT NULL FK(accounts.id) ON DELETE CASCADE
service_type_id     TEXT  NOT NULL FK(service_types.id)
tracking_code       TEXT  UNIQUE                 -- client-facing short code
status              TEXT  NOT NULL DEFAULT 'draft'  -- 'draft','active','waiting_for_client','submitted','approved','rejected','cancelled'
region              TEXT                         -- ISO country code where this service applies
metadata            JSONB NOT NULL DEFAULT '{}'  -- service-specific fields (purpose, embassy, applicant details, etc.)
assigned_to         UUID  FK(staff_members.id)   -- primary staff assignee
reviewed_by         UUID  FK(staff_members.id)
readiness_status    TEXT
rejected_at         TIMESTAMPTZ                  -- NULL = not rejected (fixes empty string hack)
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Indexes:** `account_id`, `service_type_id`, `status`, `assigned_to`, `(status, service_type_id)` composite for dashboard filtering, `region`.

**`metadata` JSONB examples:**
```json
-- DTV service (for self)
{
  "applicant_name": "John Smith",
  "relationship": "self",
  "purpose": "selfemploy",
  "submission_country": "VN",
  "submission_embassy": "Hanoi",
  "exit_date": "2026-03-07",
  "applied_before": false,
  "need_course": false
}

-- DTV service (for spouse, same account)
{
  "applicant_name": "Jane Smith",
  "relationship": "spouse",
  "purpose": "selfemploy",
  "submission_country": "VN",
  "submission_embassy": "Hanoi",
  "exit_date": "2026-03-07"
}

-- 90-day reporting service
{
  "last_arrival": "2026-01-15",
  "last_report": "2026-01-20",
  "in_thailand": true
}
```

**Rationale:** This is the core design decision. Instead of dozens of `dtv_*` columns on the account, all service-specific data lives in `metadata` JSONB on a per-service row. Common queryable fields (`status`, `assigned_to`, `region`) are promoted to real columns for indexing. Service-specific fields that are only read by the application layer stay in JSONB. This allows multiple concurrent services per user, each with its own lifecycle, documents, and workflow steps.

**Family applications:** A parent applying for DTV for themselves + spouse + 2 children simply creates 4 service rows under one `account_id`. Each service's `metadata` holds the applicant's identity (`applicant_name`, `relationship`). Each has its own independent documents, reviews, steps, and payment. No separate dependents table needed -- the service row is the unit of work.

**Tradeoff:** JSONB `metadata` trades strict column-level constraints for extensibility. Validation of service-specific fields must happen at the application layer or via CHECK constraints on known paths. For the legal dashboard's cross-account queries on common fields (status, assignee, region), real columns are used. For service-specific filters (e.g., "all DTVs submitted to Vietnam"), GIN indexes on `metadata` with `jsonb_path_ops` provide performant querying.

---

### 8. `documents`

A specific document submission within a service.

```
documents
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
service_id          UUID  NOT NULL FK(services.id) ON DELETE CASCADE
doc_type_key        TEXT  NOT NULL               -- matches document_type_requirements.doc_type_key
status              TEXT  NOT NULL DEFAULT 'pending'  -- 'pending', 'approved', 'need_changes'
feedback            TEXT                         -- staff feedback shown to client
changes_requested   TEXT                         -- change request message
title_override      TEXT                         -- custom display name if staff overrides
is_skipped          BOOLEAN NOT NULL DEFAULT false  -- waived requirement
is_custom           BOOLEAN NOT NULL DEFAULT false  -- ad-hoc doc added by staff
uploaded_at         TIMESTAMPTZ                  -- most recent upload time
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
UNIQUE(service_id, doc_type_key)
```

**Indexes:** `service_id`, `status`, `(service_id, status)`.

---

### 9. `document_files`

Individual files within a document submission.

```
document_files
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
document_id         UUID  NOT NULL FK(documents.id) ON DELETE CASCADE
storage_path        TEXT  NOT NULL               -- path in storage bucket
original_name       TEXT  NOT NULL
uploaded_at         TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Index:** `document_id`.

---

### 10. `document_reviews` (Staff Review History)

Append-only log of every staff review action. Replaces `docs[*].history`.

```
document_reviews
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
document_id         UUID  NOT NULL FK(documents.id) ON DELETE CASCADE
action              TEXT  NOT NULL               -- 'approved', 'need_changes'
reviewer_id         UUID  NOT NULL FK(staff_members.id)
review_data         JSONB                        -- supplementary data (change request text, etc.)
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Indexes:** `document_id`, `reviewer_id`, `created_at`.

**Rationale:** Proper relational audit trail with FK to staff. Replaces the embedded `history` array. The JSON-encoded `data` string in the old schema becomes structured `review_data` JSONB.

---

### 11. `ai_document_reviews`

Append-only log of AI review runs. Preserves the intentional separation between AI verdicts and authoritative account state.

```
ai_document_reviews
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
document_id         UUID  NOT NULL FK(documents.id) ON DELETE CASCADE
result              TEXT  NOT NULL               -- 'approved', 'rejected', 'unsure'
feedback            TEXT  NOT NULL               -- AI explanation
files_reviewed      TEXT[] NOT NULL              -- storage paths reviewed in this run
model_version       TEXT                         -- track which AI model produced this
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Indexes:** `document_id`, `created_at`.

**Rationale:** Decoupled from staff reviews by design. AI produces verdicts; humans decide when to promote them. `model_version` is added for auditability as AI capabilities expand.

---

### 12. `service_steps`

Tracks completion of workflow milestones per service.

```
service_steps
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
service_id          UUID  NOT NULL FK(services.id) ON DELETE CASCADE
step_key            TEXT  NOT NULL               -- matches step_definitions.step_key
completed_at        TIMESTAMPTZ NOT NULL DEFAULT now()
UNIQUE(service_id, step_key)
```

**Index:** `service_id`.

---

### 13. `payments`

One payment per service. Accommodates multiple providers.

```
payments
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
service_id          UUID  NOT NULL FK(services.id) ON DELETE CASCADE UNIQUE
provider            TEXT  NOT NULL               -- 'stripe', 'wise', 'bank_transfer', etc.
provider_reference  TEXT                         -- Stripe payment ID, Wise transfer ID, etc.
amount              NUMERIC(12,2) NOT NULL
currency            TEXT  NOT NULL DEFAULT 'THB'
status              TEXT  NOT NULL DEFAULT 'pending'  -- 'pending', 'completed', 'refunded', 'failed'
payment_url         TEXT                         -- checkout URL if applicable
eligible_for_refund BOOLEAN NOT NULL DEFAULT false
paid_at             TIMESTAMPTZ
refunded_at         TIMESTAMPTZ
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Indexes:** `service_id` (unique), `provider`, `status`.

**Rationale:** `UNIQUE(service_id)` enforces one payment per service per the clarified requirement. The `provider` column accommodates Stripe, Wise, bank transfers, etc. Referral commission tracking (`paid_referrer_date`) moves to a separate concern if needed, but can be tracked in the audit log.

---

### 14. `staff_members`

Internal users (agents, legal staff, admins, AI service accounts).

```
staff_members
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
email               TEXT  NOT NULL UNIQUE
display_name        TEXT  NOT NULL
role                TEXT  NOT NULL               -- 'admin', 'legal_staff', 'agent', 'ai_service'
is_active           BOOLEAN NOT NULL DEFAULT true
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

---

### 15. `conversations`

Customer support threads from social/messaging channels. Can exist without an account link (unlinked leads).

```
conversations
─────────────────────────────────────────────────
id                  SERIAL  PK
external_id         INTEGER UNIQUE               -- Chatwoot conversation ID
contact_id          INTEGER NOT NULL              -- Chatwoot contact ID
contact_name        TEXT
contact_phone       TEXT
contact_email       TEXT
contact_language    TEXT
contact_profile_pic TEXT
contact_country_code TEXT
contact_status      TEXT
channel_id          INTEGER NOT NULL
channel_name        TEXT  NOT NULL
channel_source      TEXT  NOT NULL                -- 'instagram', 'tiktok_business', 'whatsapp', etc.
channel_meta        JSONB NOT NULL DEFAULT '{}'
assignee_id         UUID  FK(staff_members.id)
ai_active           BOOLEAN NOT NULL DEFAULT false
is_handed_off       BOOLEAN NOT NULL DEFAULT false
is_waiting_for_legal BOOLEAN NOT NULL DEFAULT false
blocked             BOOLEAN NOT NULL DEFAULT false
lifecycle           TEXT  NOT NULL DEFAULT 'New Lead'
lifecycle_automation_disabled BOOLEAN NOT NULL DEFAULT false
opened_at           TIMESTAMPTZ
closed_at           TIMESTAMPTZ
closed_by           UUID  FK(staff_members.id)
first_response_time_seconds NUMERIC
resolution_time_seconds     NUMERIC
incoming_message_count INTEGER NOT NULL DEFAULT 0
outgoing_message_count INTEGER NOT NULL DEFAULT 0
category            TEXT
summary             TEXT
notes               TEXT
locale              TEXT
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Indexes:** `contact_id`, `assignee_id`, `channel_source`, `lifecycle`, `created_at`, `(channel_source, lifecycle)` composite.

---

### 16. `conversation_tags`

```
conversation_tags
─────────────────────────────────────────────────
conversation_id     INTEGER NOT NULL FK(conversations.id) ON DELETE CASCADE
tag                 TEXT NOT NULL
PRIMARY KEY (conversation_id, tag)
```

**Index:** `tag` for filtering by tag across conversations.

---

### 17. `conversation_lead_qualifications`

CRM/lead enrichment data for a conversation. Separated to keep the conversations table focused on messaging state.

```
conversation_lead_qualifications
─────────────────────────────────────────────────
conversation_id     INTEGER PK FK(conversations.id) ON DELETE CASCADE
client_has_account  BOOLEAN
client_name         TEXT
client_nationality  TEXT
client_location     TEXT
client_urgency      TEXT
client_interested_in TEXT
client_topics       TEXT
client_buying_segment TEXT
client_buying_intent TEXT
current_visa        TEXT
service_interest    JSONB                        -- replaces dtv_* fields; e.g. {"type":"dtv","purpose":"selfemploy","package":null}
current_step        TEXT
submission_country  TEXT
applied_before      BOOLEAN
is_emergency        BOOLEAN
is_qualified        BOOLEAN
source              TEXT
updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Rationale:** Lead qualification fields are only relevant for the CRM/sales workflow. Separating them keeps the core conversation table lean and avoids NULL-heavy rows for conversations that are never qualified.

---

### 18. `account_conversation_links`

Links accounts to conversations across multiple channels. An account can be linked to many conversations; a conversation can optionally be linked to one account.

```
account_conversation_links
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
account_id          UUID  NOT NULL FK(accounts.id) ON DELETE CASCADE
conversation_id     INTEGER NOT NULL FK(conversations.id) ON DELETE CASCADE
linked_via          TEXT  NOT NULL               -- 'chatwoot_contact_id', 'email_match', 'manual', etc.
linked_at           TIMESTAMPTZ NOT NULL DEFAULT now()
UNIQUE(account_id, conversation_id)
```

**Indexes:** `account_id`, `conversation_id`.

**Rationale:** Replaces the one-directional `issa_ai_id` lookup. Supports linking multiple conversation channels to one account. `linked_via` records how the match was made for auditability.

---

### 19. `audit_log`

System-wide append-only audit trail for compliance.

```
audit_log
─────────────────────────────────────────────────
id                  BIGSERIAL PK
actor_type          TEXT  NOT NULL               -- 'staff', 'client', 'ai_service', 'system'
actor_id            TEXT  NOT NULL               -- UUID of staff/account or service name
action              TEXT  NOT NULL               -- 'document.reviewed', 'service.status_changed', 'account.deleted', etc.
target_type         TEXT  NOT NULL               -- 'document', 'service', 'account', 'conversation'
target_id           TEXT  NOT NULL               -- UUID/ID of the affected entity
details             JSONB                        -- action-specific payload (old value, new value, etc.)
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

**Indexes:** `(target_type, target_id)`, `actor_id`, `created_at`, `action`.

**Partitioning:** Range-partition by `created_at` (monthly). Audit logs grow unboundedly; partitioning enables efficient retention policies and time-range queries for compliance.

---

### 20. `notifications`

Dispatched notifications to clients.

```
notifications
─────────────────────────────────────────────────
id                  UUID  PK  DEFAULT gen_random_uuid()
account_id          UUID  NOT NULL FK(accounts.id) ON DELETE CASCADE
service_id          UUID  FK(services.id) ON DELETE SET NULL
channel             TEXT  NOT NULL               -- 'push', 'email', 'in_app'
title               TEXT
body                TEXT  NOT NULL
status              TEXT  NOT NULL DEFAULT 'pending' -- 'pending', 'sent', 'failed'
sent_at             TIMESTAMPTZ
created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
```

---

## Registry/Lookup Tables
- service_types
- document_type_requirements
- step_definitions



## Indexing & Partitioning Strategy

### Primary Indexes (Performance-Critical)

| Table | Index | Purpose |
|---|---|---|
| `services` | `(account_id)` | Client app: fetch all my services |
| `services` | `(status, service_type_id)` | Legal dashboard: filter by status + type |
| `services` | `(assigned_to) WHERE status NOT IN ('approved','rejected','cancelled')` | Dashboard: my active cases |
| `services` | `(region, service_type_id)` | Multi-region queries |
| `services.metadata` | GIN with `jsonb_path_ops` | Service-specific filters (embassy, purpose) |
| `documents` | `(service_id, status)` | Fetch pending documents per service |
| `conversations` | `(contact_id)` | Account linking lookup |
| `conversations` | `(assignee_id, lifecycle)` | Agent dashboard: my conversations |
| `conversations` | `(channel_source, created_at)` | Channel analytics |
| `audit_log` | `(target_type, target_id, created_at)` | Entity audit history |

### Partitioning

| Table | Strategy | Rationale |
|---|---|---|
| `audit_log` | Range by `created_at` (monthly) | Unbounded growth; enables retention policies and fast time-range scans |
| `conversations` | Range by `created_at` (quarterly) | High volume from social channels; older conversations are rarely queried |
| `ai_document_reviews` | Range by `created_at` (quarterly) | Append-only, grows with every upload event |

### Scale Considerations (50K users/month, 1M concurrent)

- **Read-heavy workload:** Most clients read their own data. The `account_id` FK index on `services`, `documents`, etc. ensures single-user queries are index-only lookups.
- **Connection pooling:** PgBouncer or equivalent is assumed at this scale.
- **Read replicas:** Legal dashboard and scheduled jobs should read from replicas to avoid contention with client-facing writes.
- **Write spikes:** Document upload events and staff review sessions create burst writes. The append-only tables (`document_reviews`, `ai_document_reviews`, `audit_log`) benefit from partitioning to avoid index bloat.

---


### Row-Level Security (RLS) Boundaries

| Consumer | Access Pattern |
|---|---|
| **Client app** | Can only read/write rows where `account_id = auth.uid()`. Enforced via RLS policy on every client-facing table. |
| **Legal dashboard** | Read access to all accounts/services/documents. Write access scoped to review actions. All writes logged to `audit_log`. |
| **Comms dashboard** | Full access to `conversations` and linked tables. Read-only on account data via the link table. |
| **AI service** | Read access to documents and service metadata. Write access only to `ai_document_reviews` and conversation AI state fields. |
| **Scheduled jobs** | Service account with targeted read/write. Idempotent operations only. |

### GDPR Hard Delete Flow

1. Client requests deletion → `accounts.deleted_at` set to `now()`
2. Grace period (e.g. 30 days) — account is soft-deleted, invisible to all consumers
3. Scheduled job runs: `DELETE FROM accounts WHERE deleted_at < now() - interval '30 days'`
4. `ON DELETE CASCADE` propagates to: `services` → `documents` → `document_files`, `document_reviews`, `ai_document_reviews`, `service_steps`, `payments`, `account_channels`, `account_platforms`, `account_conversation_links`, `notifications`
5. `audit_log` entries are anonymized (actor_id/target_id replaced with `'[deleted]'`) but retained for compliance
6. Conversation records are preserved (unlinked leads remain valid) — the link row is deleted but the conversation itself stays

---

---

## Tradeoffs

### Tradeoffs Made

1. **JSONB `metadata` on services** -- Trades column-level DB constraints for extensibility. Service-specific validation must live in the application layer. The alternative (an EAV pattern or one table per service type) was rejected: EAV is hard to query and audit; table-per-type doesn't scale to 150+ countries/service combinations.

2. **Separate `conversation_lead_qualifications` table** -- Adds a JOIN for lead-qualified conversations but keeps the high-volume conversations table lean. At 1M+ conversations, fewer NULLable columns = better compression and cache efficiency.

3. **`audit_log` with TEXT ids** -- Using TEXT for `actor_id` and `target_id` (rather than typed FKs) allows logging across entity types without polymorphic FK complexity. The tradeoff is no referential integrity on audit rows — acceptable for an append-only log.

4. **Single `payments` row per service** -- Simplifies the model per requirements. If installment or multi-payment support is needed later, this becomes a 1:N relationship (remove the UNIQUE constraint on `service_id`).
