# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Email Collaboration Platform · Created: 2026-05-20

## Philosophy

This model uses a traditional relational backbone for core entities (conversations, messages, users, inboxes) but leverages PostgreSQL JSONB columns extensively for variable, extensible, and domain-specific data. The key insight is that an email collaboration platform must handle enormous variability: different email providers return different header sets, different organisations need different custom fields on conversations, different channels (email vs. SMS vs. chat) have different metadata structures, and AI enrichment produces schema-evolving outputs.

Rather than creating dozens of narrow tables or constantly running ALTER TABLE migrations, the hybrid approach places stable, frequently-queried fields in typed columns and puts everything else in indexed JSONB. This mirrors how modern platforms like Front and Help Scout structure their APIs — a fixed set of top-level fields with an extensible `custom_fields` or `metadata` object.

The approach draws from the "structured flexibility" pattern used by Stripe (where every object has a `metadata` hash), Shopify (metafields), and Salesforce (custom fields stored as generic key-value with type metadata). PostgreSQL's JSONB operators, GIN indexes, and jsonpath queries make this practical at scale without sacrificing query performance for the fields that matter most.

**Best for:** Rapid MVP development, multi-tenant platforms where tenants need custom fields, and teams that want to iterate on the schema without downtime migrations.

**Trade-offs:**
- (+) Extremely flexible — new fields require no schema migration, just a JSON key
- (+) Faster to build — fewer tables, fewer migrations, fewer junction tables
- (+) Custom fields per tenant are trivial — just different keys in the JSONB column
- (+) Channel-agnostic — email, SMS, chat metadata all fit in the same message table via JSONB
- (+) AI enrichment data evolves without schema changes
- (-) JSONB fields lack foreign key constraints — referential integrity must be enforced in application code
- (-) Complex JSONB queries can be slower than indexed relational columns
- (-) Reporting/analytics on JSONB fields requires careful GIN indexing or extraction
- (-) Schema validation must be done at the application layer (no CHECK constraints on JSON structure)
- (-) Harder for new developers to understand the full schema — some structure is implicit in JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JMAP RFC 8621 | Core message/mailbox/thread fields are relational; JMAP-specific metadata (keywords, mailbox roles) stored in JSONB |
| RFC 5322 | Common headers (From, To, Subject) are relational columns; extended headers stored in `headers` JSONB |
| RFC 2045-2049 (MIME) | Attachment metadata stored as JSONB array on the message — no separate attachment table needed for basic use |
| ITIL v4 | SLA policy targets are relational; per-tenant SLA customisations stored in JSONB config |
| OpenAPI 3.1 | API responses map cleanly — fixed fields from columns, extensible fields from JSONB |
| GDPR Article 17 | PII fields identified in a `pii_fields` JSONB metadata key for automated erasure |
| SCIM 2.0 | User provisioning attributes stored in `scim_attributes` JSONB on user records |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    -- All org-level settings in one JSONB column
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "data_residency": "EU",
    --   "timezone": "Europe/London",
    --   "business_hours": {"mon": {"start": "09:00", "end": "17:00"}, ...},
    --   "branding": {"logo_url": "...", "primary_colour": "#1a73e8"},
    --   "features": {"ai_suggestions": true, "sentiment_analysis": true},
    --   "custom_conversation_fields": [
    --     {"key": "account_tier", "label": "Account Tier", "type": "select", "options": ["free","pro","enterprise"]},
    --     {"key": "region", "label": "Region", "type": "text"}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    role            TEXT NOT NULL DEFAULT 'agent',          -- admin, agent, viewer
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    -- Extensible user profile
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "scim_external_id": "okta-123",
    --   "signature_html": "<p>Best regards...</p>",
    --   "notification_preferences": {"email": true, "push": true, "slack": false},
    --   "availability": "online"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_app_user_org ON app_user (organisation_id);

CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    member_ids      UUID[] NOT NULL DEFAULT '{}',          -- denormalised for fast lookup
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_team_org ON team (organisation_id);
```

---

## Inboxes & Connected Accounts

```sql
CREATE TABLE inbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    email_address   TEXT NOT NULL,
    channel_type    TEXT NOT NULL DEFAULT 'email',          -- email, sms, chat, social
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- Provider-specific configuration in JSONB
    provider_config JSONB NOT NULL DEFAULT '{}',
    -- Example for Gmail:
    -- {
    --   "provider": "gmail",
    --   "oauth_token": "encrypted-token",
    --   "oauth_refresh": "encrypted-refresh",
    --   "oauth_expires_at": "2026-06-01T00:00:00Z",
    --   "sync_state": "synced",
    --   "last_sync_at": "2026-05-20T10:00:00Z",
    --   "history_id": "12345"
    -- }
    -- Example for IMAP:
    -- {
    --   "provider": "imap",
    --   "imap_host": "imap.example.com",
    --   "imap_port": 993,
    --   "smtp_host": "smtp.example.com",
    --   "smtp_port": 587,
    --   "username": "support@example.com"
    -- }
    -- Example for JMAP:
    -- {
    --   "provider": "jmap",
    --   "session_url": "https://jmap.example.com/.well-known/jmap",
    --   "account_id": "abc123"
    -- }
    access_user_ids UUID[] NOT NULL DEFAULT '{}',
    access_team_ids UUID[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inbox_org ON inbox (organisation_id);
```

---

## Conversations

```sql
CREATE TABLE conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    inbox_id        UUID NOT NULL REFERENCES inbox(id),
    subject         TEXT NOT NULL DEFAULT '',
    status          TEXT NOT NULL DEFAULT 'open',
    priority        TEXT NOT NULL DEFAULT 'normal',
    channel         TEXT NOT NULL DEFAULT 'email',
    assignee_id     UUID REFERENCES app_user(id),
    team_id         UUID REFERENCES team(id),
    -- Denormalised for fast list queries
    last_message_at  TIMESTAMPTZ,
    last_message_preview TEXT,                              -- first 200 chars of last message
    message_count   INTEGER NOT NULL DEFAULT 0,
    -- Multi-inbox: array of inbox IDs this conversation appears in
    inbox_ids       UUID[] NOT NULL DEFAULT '{}',
    -- Tags as array (no junction table needed)
    tag_ids         UUID[] NOT NULL DEFAULT '{}',
    -- Participant info denormalised
    participants    JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"type": "contact", "id": "uuid", "email": "customer@example.com", "name": "Jane Doe", "role": "from"},
    --   {"type": "user", "id": "uuid", "email": "agent@company.com", "name": "John Agent", "role": "assignee"}
    -- ]
    -- Custom fields per org (defined in organisation.settings.custom_conversation_fields)
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"account_tier": "enterprise", "region": "EMEA", "contract_value": 50000}
    -- SLA tracking denormalised
    sla             JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "policy_id": "uuid",
    --   "policy_name": "Enterprise SLA",
    --   "first_response_due": "2026-05-20T11:00:00Z",
    --   "first_response_at": null,
    --   "resolution_due": "2026-05-21T17:00:00Z",
    --   "status": "active",
    --   "breached": false
    -- }
    -- AI enrichment
    ai              JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "intent": "order_status_inquiry",
    --   "intent_confidence": 0.94,
    --   "sentiment": "frustrated",
    --   "sentiment_score": -0.72,
    --   "summary": "Customer asking about delayed order #1234",
    --   "suggested_priority": "high",
    --   "suggested_tags": ["shipping", "escalation"]
    -- }
    -- Snooze
    snooze_until    TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Primary listing query: org + status + recency
CREATE INDEX idx_conv_org_status_last ON conversation (organisation_id, status, last_message_at DESC);
-- Assignee workload
CREATE INDEX idx_conv_assignee ON conversation (assignee_id) WHERE assignee_id IS NOT NULL AND status = 'open';
-- SLA breach risk queries
CREATE INDEX idx_conv_sla_due ON conversation
    USING GIN (sla jsonb_path_ops);
-- Tag filtering
CREATE INDEX idx_conv_tags ON conversation USING GIN (tag_ids);
-- Custom field queries (GIN index on the JSONB)
CREATE INDEX idx_conv_custom ON conversation USING GIN (custom_fields jsonb_path_ops);
-- Inbox filtering
CREATE INDEX idx_conv_inboxes ON conversation USING GIN (inbox_ids);
```

---

## Messages

```sql
CREATE TABLE message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    -- Core fields (relational for fast queries)
    author_type     TEXT NOT NULL,                          -- agent, contact, system, ai
    author_id       UUID,                                   -- user or contact UUID
    message_type    TEXT NOT NULL DEFAULT 'reply',         -- reply, note, system, draft
    direction       TEXT NOT NULL DEFAULT 'inbound',
    body_plain      TEXT,
    body_html       TEXT,
    -- Email-specific headers (JSONB for flexibility across channels)
    headers         JSONB NOT NULL DEFAULT '{}',
    -- Example for email:
    -- {
    --   "message_id": "<abc@example.com>",
    --   "in_reply_to": "<def@example.com>",
    --   "references": ["<ghi@example.com>", "<def@example.com>"],
    --   "from": {"name": "Jane Doe", "email": "jane@example.com"},
    --   "to": [{"name": "Support", "email": "support@company.com"}],
    --   "cc": [],
    --   "bcc": [],
    --   "subject": "Re: Order #1234"
    -- }
    -- Example for SMS:
    -- {
    --   "from_number": "+14155551234",
    --   "to_number": "+14155555678",
    --   "provider_message_id": "twilio-sid-123"
    -- }
    -- Attachments as JSONB array (avoids separate table for simple cases)
    attachments     JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"id": "uuid", "filename": "receipt.pdf", "content_type": "application/pdf",
    --    "size_bytes": 45000, "storage_key": "s3://bucket/key", "is_inline": false},
    --   {"id": "uuid", "filename": "screenshot.png", "content_type": "image/png",
    --    "size_bytes": 120000, "storage_key": "s3://bucket/key2", "content_id": "cid:img1", "is_inline": true}
    -- ]
    -- AI analysis per message
    ai              JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "sentiment": "negative",
    --   "sentiment_score": -0.65,
    --   "language": "en",
    --   "suggested_reply": "I apologize for the delay...",
    --   "suggested_reply_confidence": 0.87
    -- }
    -- Channel-specific metadata
    channel_meta    JSONB NOT NULL DEFAULT '{}',
    is_draft        BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_message_conv ON message (conversation_id, created_at);
CREATE INDEX idx_message_headers_msgid ON message ((headers->>'message_id'))
    WHERE headers->>'message_id' IS NOT NULL;
```

---

## Contacts

```sql
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    display_name    TEXT,
    primary_email   TEXT,
    -- All handles in one JSONB array (no junction table)
    handles         JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"type": "email", "value": "jane@example.com", "is_primary": true},
    --   {"type": "email", "value": "jane.doe@company.com", "is_primary": false},
    --   {"type": "phone", "value": "+14155551234", "is_primary": false},
    --   {"type": "twitter", "value": "@janedoe", "is_primary": false}
    -- ]
    -- Company and role info
    company         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"name": "Acme Corp", "domain": "acme.com", "industry": "SaaS", "size": "50-200"}
    -- CRM sync data
    crm             JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "salesforce": {"id": "003xxx", "last_synced": "2026-05-20T10:00:00Z"},
    --   "hubspot": {"id": "12345", "last_synced": "2026-05-20T10:00:00Z", "deal_value": 50000}
    -- }
    -- Custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Conversation stats (denormalised)
    conversation_count INTEGER NOT NULL DEFAULT 0,
    last_contacted_at TIMESTAMPTZ,
    -- GDPR
    is_anonymised   BOOLEAN NOT NULL DEFAULT FALSE,
    pii_fields      TEXT[] NOT NULL DEFAULT '{display_name,primary_email,handles}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contact_org ON contact (organisation_id);
CREATE INDEX idx_contact_email ON contact (primary_email) WHERE primary_email IS NOT NULL;
-- GIN index for handle lookups across all handle types
CREATE INDEX idx_contact_handles ON contact USING GIN (handles jsonb_path_ops);
```

### Contact Handle Lookup Query Example

```sql
-- Find a contact by any email handle
SELECT * FROM contact
WHERE organisation_id = 'org-uuid'
  AND handles @> '[{"type": "email", "value": "jane@example.com"}]';

-- Find contacts at a specific company domain
SELECT * FROM contact
WHERE organisation_id = 'org-uuid'
  AND company->>'domain' = 'acme.com';
```

---

## Tags

```sql
CREATE TABLE tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    colour          TEXT,
    parent_id       UUID REFERENCES tag(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, name)
);
CREATE INDEX idx_tag_org ON tag (organisation_id);
```

---

## SLA Policies

```sql
CREATE TABLE sla_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    description     TEXT,
    priority        TEXT NOT NULL,
    -- Targets in JSONB for flexibility (different targets per channel, etc.)
    targets         JSONB NOT NULL,
    -- Example:
    -- {
    --   "first_response": {"seconds": 3600, "business_hours": true},
    --   "next_response": {"seconds": 7200, "business_hours": true},
    --   "resolution": {"seconds": 86400, "business_hours": true}
    -- }
    -- Conditions for matching
    conditions      JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"field": "priority", "op": "eq", "value": "urgent"},
    --   {"field": "inbox_id", "op": "in", "value": ["uuid1", "uuid2"]}
    -- ]
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Automation Rules

```sql
CREATE TABLE automation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    description     TEXT,
    -- Entire rule definition in JSONB
    rule            JSONB NOT NULL,
    -- Example:
    -- {
    --   "trigger": "message.received",
    --   "conditions": [
    --     {"field": "message.body_plain", "op": "contains", "value": "urgent"},
    --     {"field": "conversation.priority", "op": "eq", "value": "normal"}
    --   ],
    --   "actions": [
    --     {"type": "set_priority", "value": "high"},
    --     {"type": "assign_team", "team_id": "uuid"},
    --     {"type": "add_tag", "tag_id": "uuid"},
    --     {"type": "send_notification", "channel": "slack", "message": "Urgent email received"}
    --   ]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    run_count       INTEGER NOT NULL DEFAULT 0,
    last_run_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Canned Responses

```sql
CREATE TABLE canned_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    subject         TEXT,
    body_html       TEXT NOT NULL,
    body_plain      TEXT,
    category        TEXT,
    variables       TEXT[] NOT NULL DEFAULT '{}',           -- e.g. {'customer_name','order_id'}
    created_by      UUID REFERENCES app_user(id),
    usage_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Knowledge Base

```sql
CREATE TABLE kb_article (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    collection      TEXT NOT NULL DEFAULT 'general',
    title           TEXT NOT NULL,
    slug            TEXT NOT NULL,
    body_html       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    -- Search and stats
    search_vector   TSVECTOR,
    view_count      INTEGER NOT NULL DEFAULT 0,
    helpful_count   INTEGER NOT NULL DEFAULT 0,
    -- Metadata
    meta            JSONB NOT NULL DEFAULT '{}',
    -- Example: {"author_id": "uuid", "tags": ["billing","returns"], "seo_description": "..."}
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);
CREATE INDEX idx_kb_search ON kb_article USING GIN (search_vector);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'user',
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    -- Before/after snapshots in JSONB
    changes         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"assignee_id": {"old": null, "new": "uuid-of-agent"}}
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"ip": "192.168.1.1", "user_agent": "Mozilla/5.0...", "automation_rule_id": "uuid"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org_time ON audit_log (organisation_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);
```

---

## Notifications & Integrations

```sql
CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    type            TEXT NOT NULL,
    title           TEXT NOT NULL,
    body            TEXT,
    resource_type   TEXT,
    resource_id     UUID,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notif_user ON notification (user_id, is_read, created_at DESC);

CREATE TABLE webhook (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret          TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    meta            JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisation, app_user, team |
| Inboxes | 1 | inbox (provider config in JSONB) |
| Conversations | 1 | conversation (participants, SLA, AI, custom fields all in JSONB) |
| Messages | 1 | message (headers, attachments, AI all in JSONB) |
| Contacts | 1 | contact (handles, CRM, company all in JSONB) |
| Tags | 1 | tag |
| SLA Policies | 1 | sla_policy (targets and conditions in JSONB) |
| Automation | 2 | automation_rule, canned_response |
| Knowledge Base | 1 | kb_article |
| Audit & Notifications | 2 | audit_log, notification |
| Integrations | 1 | webhook |
| **Total** | **15** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **JSONB for channel-agnostic messages** — The `headers` JSONB on the message table stores email headers, SMS metadata, or chat metadata in the same column. This means adding a new channel (e.g., WhatsApp) requires zero schema changes — just new keys in the JSONB.

2. **Attachments as JSONB array, not a separate table** — For most operations (displaying a message with its attachments), embedding attachments in the message row eliminates a JOIN. For platforms with heavy attachment-specific queries (search by filename across all messages), a separate table or a GIN index on the JSONB array would be added.

3. **Tags as UUID arrays, not junction tables** — The `tag_ids UUID[]` column on conversations with a GIN index replaces a `conversation_tag` junction table. Array containment queries (`tag_ids @> ARRAY['uuid']::uuid[]`) are fast with GIN indexes and avoid the JOIN overhead of junction tables.

4. **SLA tracking denormalised into conversation** — The `sla` JSONB column on conversation stores the current SLA state inline. This eliminates a JOIN for the most common query (list conversations with SLA status). A background worker updates this JSONB when SLA events occur.

5. **Custom fields per organisation** — The `custom_conversation_fields` definition in `organisation.settings` defines what fields are available; the actual values are stored in `conversation.custom_fields`. This gives each tenant their own "schema" without ALTER TABLE.

6. **Contact handles in JSONB array** — Rather than a separate `contact_handle` table, handles are embedded in the contact row. The GIN index with `jsonb_path_ops` enables efficient lookups like "find contact with email X" using the `@>` containment operator.

7. **AI enrichment co-located with its entity** — Sentiment, intent, and suggestions are stored as JSONB on the conversation and message rows where they are displayed. This avoids JOINs for the read path and allows the AI schema to evolve (new models, new fields) without migrations.

8. **Organisation settings as JSONB configuration** — Business hours, branding, feature flags, and custom field definitions all live in `organisation.settings`. This makes the organisation table a self-contained configuration document that the application loads once and caches.
