# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Email Collaboration Platform · Created: 2026-05-20

## Philosophy

This model follows a traditional normalized relational design where every domain concept gets its own table with well-defined foreign keys and constraints. The schema is directly inspired by the JMAP (RFC 8620/8621) data model, which defines four primary mail objects — Mailbox, Email, Thread, and Identity — as first-class entities. By aligning the internal schema with the JMAP specification, the platform gains a natural mapping between its database and the modern email protocol it exposes.

Every relationship is explicit. Junction tables handle many-to-many associations (emails belonging to multiple mailboxes, conversations having multiple participants, contacts having multiple handles). Reference data tables enforce controlled vocabularies for SLA tiers, conversation statuses, and channel types. This makes the schema self-documenting and straightforward to query with standard SQL joins.

The trade-off is table count: a fully normalized model for this domain requires 40+ tables. But the benefit is data integrity — no orphaned records, no ambiguous nulls, no duplicated data. For a platform where SLA compliance and audit trails are business-critical, referential integrity is not optional.

**Best for:** Teams prioritising data integrity, regulatory compliance, and long-term maintainability over rapid prototyping speed.

**Trade-offs:**
- (+) Strong referential integrity — impossible to have orphaned messages or broken SLA chains
- (+) Natural alignment with JMAP RFC 8621 data model — simplifies protocol implementation
- (+) Standard SQL queries — no JSONB operators or graph traversals needed for most operations
- (+) Easy to reason about for new developers — every concept has a table
- (-) High table count (~45 tables) increases migration complexity
- (-) Schema changes require ALTER TABLE + data migration for new fields
- (-) Many-to-many junction tables add JOIN overhead for common queries
- (-) Less flexible for jurisdiction-specific or customer-specific custom fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JMAP RFC 8621 | Mailbox, Email, Thread, Identity entities map directly to JMAP data types |
| JMAP RFC 8620 | Core JMAP session/account model informs the account and mailbox_account tables |
| RFC 5322 | Email message headers (From, To, Subject, Date, Message-ID, In-Reply-To, References) stored as explicit columns |
| RFC 2045-2049 (MIME) | Attachment and body part tables model MIME multipart structure |
| ITIL v4 | SLA policy tiers (Incident, Service Request, Change) modeled as reference data |
| ISO 3166-1 | Country codes for contact addresses and data residency tracking |
| OAuth 2.0 (RFC 6749) | Connected account credentials table stores OAuth tokens per provider |
| SCIM 2.0 (RFC 7643) | User provisioning fields align with SCIM schema for enterprise SSO sync |
| GDPR Article 17 | Soft-delete and anonymisation columns on contact and message tables |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisations (tenants)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',          -- free, starter, professional, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    data_residency  TEXT,                                   -- ISO 3166-1 alpha-2 country code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisation_slug ON organisation (slug);

-- Users (team members)
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    status          TEXT NOT NULL DEFAULT 'active',        -- active, deactivated, invited
    scim_external_id TEXT,                                  -- SCIM 2.0 external ID for SSO provisioning
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_app_user_org ON app_user (organisation_id);

-- Roles (RBAC)
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                          -- admin, agent, viewer, custom
    permissions     TEXT[] NOT NULL DEFAULT '{}',           -- e.g. {'inbox.read','inbox.write','sla.manage'}
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_role_org_name ON role (organisation_id, name);

-- User-role assignments
CREATE TABLE user_role (
    user_id UUID NOT NULL REFERENCES app_user(id),
    role_id UUID NOT NULL REFERENCES role(id),
    PRIMARY KEY (user_id, role_id)
);

-- Teams
CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    team_id UUID NOT NULL REFERENCES team(id),
    user_id UUID NOT NULL REFERENCES app_user(id),
    PRIMARY KEY (team_id, user_id)
);
```

---

## Connected Email Accounts

```sql
-- Connected mailbox accounts (Gmail, Outlook, IMAP)
CREATE TABLE connected_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    user_id         UUID REFERENCES app_user(id),          -- NULL if org-level shared account
    provider        TEXT NOT NULL,                          -- gmail, outlook, imap, jmap
    email_address   TEXT NOT NULL,
    display_name    TEXT,
    oauth_token     TEXT,                                   -- encrypted OAuth 2.0 access token
    oauth_refresh   TEXT,                                   -- encrypted refresh token
    oauth_expires_at TIMESTAMPTZ,
    imap_host       TEXT,
    imap_port       INTEGER,
    smtp_host       TEXT,
    smtp_port       INTEGER,
    jmap_session_url TEXT,                                  -- JMAP session resource URL (RFC 8620)
    sync_state      TEXT NOT NULL DEFAULT 'pending',       -- pending, syncing, synced, error
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_connected_account_org ON connected_account (organisation_id);
```

---

## Inbox & Mailbox (JMAP-aligned)

```sql
-- Shared inboxes (maps to JMAP Mailbox)
CREATE TABLE inbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                          -- e.g. 'Support', 'Billing', 'Sales'
    email_address   TEXT NOT NULL,                          -- support@company.com
    connected_account_id UUID REFERENCES connected_account(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inbox_org ON inbox (organisation_id);

-- Inbox access (which users/teams can see which inboxes)
CREATE TABLE inbox_access (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inbox_id    UUID NOT NULL REFERENCES inbox(id),
    user_id     UUID REFERENCES app_user(id),
    team_id     UUID REFERENCES team(id),
    access_level TEXT NOT NULL DEFAULT 'read_write',       -- read_only, read_write, admin
    CHECK (user_id IS NOT NULL OR team_id IS NOT NULL)
);
CREATE INDEX idx_inbox_access_inbox ON inbox_access (inbox_id);
```

---

## Conversations & Threads (JMAP-aligned)

```sql
-- Conversations (the central entity — maps to JMAP Thread)
CREATE TABLE conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    subject         TEXT NOT NULL DEFAULT '',
    status          TEXT NOT NULL DEFAULT 'open',          -- open, snoozed, closed, archived, spam, trash
    priority        TEXT NOT NULL DEFAULT 'normal',        -- urgent, high, normal, low
    channel         TEXT NOT NULL DEFAULT 'email',         -- email, sms, chat, social
    assignee_id     UUID REFERENCES app_user(id),
    team_id         UUID REFERENCES team(id),
    snooze_until    TIMESTAMPTZ,
    first_message_at TIMESTAMPTZ,
    last_message_at  TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    message_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conversation_org_status ON conversation (organisation_id, status);
CREATE INDEX idx_conversation_assignee ON conversation (assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_conversation_last_msg ON conversation (organisation_id, last_message_at DESC);

-- Conversation-inbox junction (a conversation can appear in multiple inboxes)
CREATE TABLE conversation_inbox (
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    inbox_id        UUID NOT NULL REFERENCES inbox(id),
    PRIMARY KEY (conversation_id, inbox_id)
);

-- Conversation participants
CREATE TABLE conversation_participant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    contact_id      UUID REFERENCES contact(id),
    user_id         UUID REFERENCES app_user(id),
    role            TEXT NOT NULL DEFAULT 'participant',   -- from, to, cc, bcc, participant
    CHECK (contact_id IS NOT NULL OR user_id IS NOT NULL)
);
CREATE INDEX idx_conv_participant_conv ON conversation_participant (conversation_id);
```

---

## Messages (JMAP Email-aligned)

```sql
-- Messages (maps to JMAP Email)
CREATE TABLE message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    author_type     TEXT NOT NULL,                          -- agent, contact, system, ai
    author_user_id  UUID REFERENCES app_user(id),
    author_contact_id UUID REFERENCES contact(id),
    message_type    TEXT NOT NULL DEFAULT 'email',         -- email, note, sms, chat, system
    direction       TEXT NOT NULL DEFAULT 'inbound',       -- inbound, outbound
    -- RFC 5322 headers
    rfc_message_id  TEXT,                                   -- Message-ID header
    in_reply_to     TEXT,                                   -- In-Reply-To header
    references      TEXT[],                                 -- References header chain
    from_address    TEXT,
    to_addresses    TEXT[],
    cc_addresses    TEXT[],
    bcc_addresses   TEXT[],
    subject         TEXT,
    body_plain      TEXT,
    body_html       TEXT,
    -- Status
    is_draft        BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_message_conversation ON message (conversation_id, created_at);
CREATE INDEX idx_message_rfc_id ON message (rfc_message_id) WHERE rfc_message_id IS NOT NULL;

-- Message attachments (MIME parts)
CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES message(id),
    filename        TEXT NOT NULL,
    content_type    TEXT NOT NULL,                          -- MIME type per RFC 2045
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,                          -- S3/object storage key
    content_id      TEXT,                                   -- Content-ID for inline attachments
    is_inline       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_attachment_message ON attachment (message_id);
```

---

## Contacts & CRM

```sql
-- Contacts (external people the org communicates with)
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    display_name    TEXT,
    company_name    TEXT,
    avatar_url      TEXT,
    notes           TEXT,
    is_anonymised   BOOLEAN NOT NULL DEFAULT FALSE,        -- GDPR Article 17 compliance
    anonymised_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contact_org ON contact (organisation_id);

-- Contact handles (email addresses, phone numbers, social accounts)
CREATE TABLE contact_handle (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id  UUID NOT NULL REFERENCES contact(id),
    handle_type TEXT NOT NULL,                              -- email, phone, twitter, facebook, whatsapp
    handle      TEXT NOT NULL,                              -- the actual address/number
    is_primary  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contact_handle_lookup ON contact_handle (handle_type, handle);
CREATE INDEX idx_contact_handle_contact ON contact_handle (contact_id);

-- CRM sync records
CREATE TABLE crm_sync (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    provider        TEXT NOT NULL,                          -- salesforce, hubspot
    external_id     TEXT NOT NULL,
    last_synced_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    sync_data       JSONB,
    UNIQUE (contact_id, provider)
);
```

---

## Tags & Labels

```sql
CREATE TABLE tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    colour          TEXT,                                    -- hex colour code
    parent_tag_id   UUID REFERENCES tag(id),                -- hierarchical tags
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_tag_org_name ON tag (organisation_id, name);

CREATE TABLE conversation_tag (
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    tag_id          UUID NOT NULL REFERENCES tag(id),
    tagged_by       UUID REFERENCES app_user(id),
    tagged_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (conversation_id, tag_id)
);
```

---

## SLA Management (ITIL v4-aligned)

```sql
-- SLA policies
CREATE TABLE sla_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    description     TEXT,
    priority        TEXT NOT NULL,                          -- urgent, high, normal, low
    -- ITIL v4 categories
    sla_type        TEXT NOT NULL DEFAULT 'incident',      -- incident, service_request, change
    -- Targets in seconds
    first_response_target   INTEGER NOT NULL,               -- e.g. 3600 = 1 hour
    resolution_target       INTEGER,                        -- e.g. 86400 = 24 hours
    next_response_target    INTEGER,                        -- ongoing reply target
    -- Business hours
    business_hours_id UUID REFERENCES business_hours(id),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Business hours definitions
CREATE TABLE business_hours (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    timezone        TEXT NOT NULL,                          -- IANA timezone
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE business_hours_schedule (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_hours_id UUID NOT NULL REFERENCES business_hours(id),
    day_of_week       SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),  -- 0=Sunday
    start_time        TIME NOT NULL,
    end_time          TIME NOT NULL
);

-- SLA policy conditions (which conversations get this SLA)
CREATE TABLE sla_policy_condition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sla_policy_id   UUID NOT NULL REFERENCES sla_policy(id),
    field           TEXT NOT NULL,                          -- inbox_id, tag, priority, channel
    operator        TEXT NOT NULL,                          -- eq, neq, in, not_in
    value           TEXT NOT NULL
);

-- SLA instances (tracking per conversation)
CREATE TABLE sla_instance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    sla_policy_id   UUID NOT NULL REFERENCES sla_policy(id),
    -- Timestamps
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    first_response_at TIMESTAMPTZ,
    first_response_due TIMESTAMPTZ NOT NULL,
    resolution_due  TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    -- Status
    first_response_breached BOOLEAN NOT NULL DEFAULT FALSE,
    resolution_breached     BOOLEAN NOT NULL DEFAULT FALSE,
    status          TEXT NOT NULL DEFAULT 'active',        -- active, met, breached, paused
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sla_instance_conv ON sla_instance (conversation_id);
CREATE INDEX idx_sla_instance_due ON sla_instance (first_response_due) WHERE status = 'active';
```

---

## Automation & Workflow Rules

```sql
CREATE TABLE automation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    description     TEXT,
    trigger_event   TEXT NOT NULL,                          -- conversation.created, message.received, sla.breaching
    conditions      JSONB NOT NULL DEFAULT '[]',           -- [{field, operator, value}]
    actions         JSONB NOT NULL DEFAULT '[]',           -- [{type, params}]
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_automation_rule_org ON automation_rule (organisation_id, is_active);

-- Canned responses / templates
CREATE TABLE canned_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    subject         TEXT,
    body_plain      TEXT,
    body_html       TEXT,
    category        TEXT,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Knowledge Base

```sql
CREATE TABLE kb_collection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    is_public       BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE TABLE kb_article (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES kb_collection(id),
    title           TEXT NOT NULL,
    slug            TEXT NOT NULL,
    body_html       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',         -- draft, published, archived
    author_id       UUID REFERENCES app_user(id),
    published_at    TIMESTAMPTZ,
    view_count      INTEGER NOT NULL DEFAULT 0,
    helpful_count   INTEGER NOT NULL DEFAULT 0,
    not_helpful_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kb_article_collection ON kb_article (collection_id, status);
```

---

## AI Features

```sql
-- AI-generated suggestions per conversation
CREATE TABLE ai_suggestion (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    suggestion_type TEXT NOT NULL,                          -- reply_draft, intent_classification, sentiment, priority, summary
    content         TEXT NOT NULL,
    confidence      NUMERIC(4,3),                           -- 0.000 to 1.000
    model_version   TEXT,
    accepted        BOOLEAN,
    accepted_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_suggestion_conv ON ai_suggestion (conversation_id);

-- Sentiment analysis per message
CREATE TABLE message_sentiment (
    message_id      UUID PRIMARY KEY REFERENCES message(id),
    sentiment       TEXT NOT NULL,                          -- positive, neutral, negative, frustrated
    score           NUMERIC(4,3) NOT NULL,
    analysed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    actor_id        UUID REFERENCES app_user(id),
    actor_type      TEXT NOT NULL DEFAULT 'user',          -- user, system, automation, ai
    action          TEXT NOT NULL,                          -- conversation.assigned, message.sent, sla.breached, etc.
    resource_type   TEXT NOT NULL,                          -- conversation, message, contact, sla_policy, etc.
    resource_id     UUID NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_org_time ON audit_log (organisation_id, created_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log (resource_type, resource_id);
```

---

## Notifications & Collision Detection

```sql
CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    notification_type TEXT NOT NULL,                        -- assignment, mention, sla_warning, new_message
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    title           TEXT NOT NULL,
    body            TEXT,
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notification_user ON notification (user_id, is_read, created_at DESC);

-- Active draft tracking for collision detection
CREATE TABLE active_draft (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_heartbeat  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (conversation_id, user_id)
);
CREATE INDEX idx_active_draft_conv ON active_draft (conversation_id);
```

---

## Integrations & Webhooks

```sql
CREATE TABLE integration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        TEXT NOT NULL,                          -- slack, salesforce, hubspot, zapier, custom
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,                        -- e.g. {'conversation.created','message.received'}
    secret          TEXT NOT NULL,                          -- HMAC signing secret
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 6 | organisation, app_user, role, user_role, team, team_member |
| Connected Accounts | 1 | connected_account |
| Inbox Management | 2 | inbox, inbox_access |
| Conversations & Threading | 3 | conversation, conversation_inbox, conversation_participant |
| Messages & Attachments | 2 | message, attachment |
| Contacts & CRM | 3 | contact, contact_handle, crm_sync |
| Tags | 2 | tag, conversation_tag |
| SLA Management | 5 | sla_policy, business_hours, business_hours_schedule, sla_policy_condition, sla_instance |
| Automation | 2 | automation_rule, canned_response |
| Knowledge Base | 2 | kb_collection, kb_article |
| AI Features | 2 | ai_suggestion, message_sentiment |
| Audit & Notifications | 3 | audit_log, notification, active_draft |
| Integrations | 2 | integration, webhook |
| **Total** | **33** | |

---

## Key Design Decisions

1. **JMAP-aligned entity model** — Conversation maps to JMAP Thread, Message maps to JMAP Email, Inbox maps to JMAP Mailbox. This means the JMAP protocol layer is a thin translation, not a deep transformation.

2. **Conversations can span multiple inboxes** — The `conversation_inbox` junction table models the real-world case where an email sent to support@ with billing@ CC'd appears in both inboxes. This mirrors JMAP's design where an Email can belong to multiple Mailboxes.

3. **Contact handles are separated from contacts** — A single contact may have multiple email addresses, phone numbers, and social handles. The `contact_handle` table allows cross-channel conversation stitching by looking up any handle to find the same contact.

4. **SLA policies use condition matching, not inbox binding** — Rather than hard-coding "inbox X gets SLA Y", the `sla_policy_condition` table allows flexible matching on any combination of inbox, tag, priority, and channel. This mirrors Zendesk's sophisticated SLA engine.

5. **Business hours are first-class entities** — SLA calculations must account for business hours. Separating business hours into their own tables (with per-day schedules) allows different SLAs to use different hour definitions (e.g., 24/7 for urgent, 9-5 for normal).

6. **Audit log uses a generic resource_type/resource_id pattern** — Rather than separate audit tables per entity, a single polymorphic audit table captures all actions. The `metadata` JSONB column stores action-specific details (e.g., old and new assignee for an assignment change).

7. **Active draft table for collision detection** — Instead of relying solely on WebSocket presence, the `active_draft` table provides persistent state for detecting when multiple agents are drafting replies to the same conversation. A heartbeat mechanism handles stale entries.

8. **GDPR compliance built into contact table** — The `is_anonymised` and `anonymised_at` columns on the contact table support the right to erasure (Article 17) without deleting the conversation history. Anonymised contacts retain their conversation associations but lose PII.
