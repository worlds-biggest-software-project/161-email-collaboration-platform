# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Email Collaboration Platform · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The event log is the single source of truth; all queryable state is derived from it via materialised read models (CQRS — Command Query Responsibility Segregation). When a conversation is assigned, an SLA is breached, or a message is sent, the system writes an event, not an UPDATE statement. Read models are projections that can be rebuilt from the event stream at any time.

This architecture is inspired by event-sourced systems used in financial services, compliance-heavy SaaS platforms, and platforms like EventStoreDB. For an email collaboration platform where audit trails, SLA tracking, and temporal queries ("who was assigned to this conversation at 3pm last Tuesday?") are core requirements, event sourcing provides these capabilities natively rather than bolting them on as an afterthought.

The key insight is that email collaboration is inherently event-driven: a message arrives, an agent assigns it, a reply is drafted, an SLA timer starts, a customer escalates. Modeling these as events rather than mutable rows captures the full history of every conversation and enables powerful analytics, replay, and debugging.

**Best for:** Platforms where complete audit trails, temporal queries, and regulatory compliance are paramount — especially for enterprise, healthcare, and government deployments.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — every state change is an event
- (+) Temporal queries are trivial — replay events to any point in time
- (+) Enables powerful analytics — event streams feed real-time dashboards, AI training, and SLA prediction models
- (+) Read models can be optimised independently — different projections for different query patterns
- (+) Natural fit for email (inherently event-driven domain)
- (-) Higher write amplification — every change is an INSERT + projection update
- (-) Increased complexity — developers must understand event sourcing, projections, and eventual consistency
- (-) Read model rebuilds can be slow for large event stores
- (-) Eventual consistency between write (events) and read (projections) may surprise users
- (-) More infrastructure — requires event bus/queue (e.g., NATS, Kafka, or pg_notify)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JMAP RFC 8621 | Read model projections align with JMAP Mailbox/Email/Thread data types |
| RFC 5322 | Email header fields stored as event payload properties |
| ITIL v4 | SLA lifecycle events (started, paused, breached, met) model ITIL incident lifecycle |
| OCSF (Open Cybersecurity Schema Framework) | Event schema structure inspired by OCSF for structured, typed event logging |
| GDPR Article 17 | Crypto-shredding pattern — per-contact encryption keys allow GDPR deletion without mutating events |
| ISO/IEC 27001 | Immutable event store satisfies audit logging requirements (A.8.15 Logging) |
| OAuth 2.0 | Token lifecycle events (granted, refreshed, revoked) tracked in event stream |

---

## Event Store (Source of Truth)

```sql
-- The immutable event store — all state changes are appended here
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                          -- aggregate root ID (conversation, contact, etc.)
    stream_type     TEXT NOT NULL,                          -- conversation, contact, inbox, sla, user
    event_type      TEXT NOT NULL,                          -- e.g. conversation.created, message.received, sla.breached
    event_version   BIGINT NOT NULL,                        -- monotonically increasing per stream
    payload         JSONB NOT NULL,                         -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',           -- actor, ip, correlation_id, causation_id
    organisation_id UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Optimised for reading a stream in order
CREATE INDEX idx_event_stream ON event_store (stream_id, event_version);
-- Optimised for replaying all events of a type (projection rebuilds)
CREATE INDEX idx_event_type ON event_store (event_type, created_at);
-- Optimised for org-scoped event queries
CREATE INDEX idx_event_org ON event_store (organisation_id, created_at);
-- Partitioned by month for large-scale deployments
-- In production, partition by created_at range
```

### Event Type Taxonomy

```
-- Conversation lifecycle
conversation.created
conversation.assigned
conversation.unassigned
conversation.status_changed      -- open, snoozed, closed, archived
conversation.priority_changed
conversation.tagged
conversation.untagged
conversation.merged
conversation.moved               -- between inboxes

-- Message lifecycle
message.received                 -- inbound email/sms/chat
message.sent                     -- outbound reply
message.draft_created
message.draft_updated
message.draft_discarded
message.note_added               -- internal note

-- SLA lifecycle
sla.started
sla.paused                       -- e.g. waiting on customer
sla.resumed
sla.first_response_met
sla.first_response_breached
sla.resolution_met
sla.resolution_breached

-- Contact lifecycle
contact.created
contact.updated
contact.merged
contact.anonymised               -- GDPR erasure

-- AI events
ai.suggestion_generated
ai.suggestion_accepted
ai.suggestion_rejected
ai.sentiment_analysed
ai.intent_classified

-- Collaboration events
collaboration.draft_started      -- collision detection
collaboration.draft_ended
collaboration.mention_created
```

### Example Event Payloads

```sql
-- conversation.assigned event
-- payload: {
--   "assignee_id": "uuid-of-agent",
--   "previous_assignee_id": null,
--   "assignment_method": "manual"    -- manual, automation, round_robin, ai
-- }
-- metadata: {
--   "actor_id": "uuid-of-manager",
--   "actor_type": "user",
--   "ip_address": "192.168.1.1",
--   "correlation_id": "uuid-linking-related-events",
--   "causation_id": "uuid-of-triggering-event"
-- }

-- message.received event
-- payload: {
--   "message_id": "uuid",
--   "rfc_message_id": "<abc@example.com>",
--   "from_address": "customer@example.com",
--   "to_addresses": ["support@company.com"],
--   "subject": "Order #1234 not delivered",
--   "body_plain": "...",
--   "body_html": "...",
--   "attachments": [{"filename": "receipt.pdf", "size": 45000, "storage_key": "s3://..."}],
--   "channel": "email",
--   "contact_id": "uuid"
-- }

-- sla.first_response_breached event
-- payload: {
--   "sla_policy_id": "uuid",
--   "target_seconds": 3600,
--   "actual_seconds": 4521,
--   "breach_severity": "minor"       -- minor (<2x), major (2-5x), critical (>5x)
-- }
```

---

## Read Model Projections

These materialised views are rebuilt from the event store. They are the query layer.

### Conversation Projection

```sql
-- Materialised read model: current conversation state
CREATE TABLE rm_conversation (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    subject         TEXT NOT NULL DEFAULT '',
    status          TEXT NOT NULL DEFAULT 'open',
    priority        TEXT NOT NULL DEFAULT 'normal',
    channel         TEXT NOT NULL DEFAULT 'email',
    assignee_id     UUID,
    team_id         UUID,
    snooze_until    TIMESTAMPTZ,
    first_message_at TIMESTAMPTZ,
    last_message_at  TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    message_count   INTEGER NOT NULL DEFAULT 0,
    inbox_ids       UUID[] NOT NULL DEFAULT '{}',
    tag_ids         UUID[] NOT NULL DEFAULT '{}',
    -- Denormalised contact info for fast display
    primary_contact_id   UUID,
    primary_contact_name TEXT,
    primary_contact_email TEXT,
    -- SLA status denormalised
    sla_status      TEXT,                                   -- on_track, at_risk, breached
    sla_next_due    TIMESTAMPTZ,
    -- Event tracking
    last_event_id   UUID NOT NULL,
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_conv_org_status ON rm_conversation (organisation_id, status);
CREATE INDEX idx_rm_conv_assignee ON rm_conversation (assignee_id) WHERE assignee_id IS NOT NULL;
CREATE INDEX idx_rm_conv_sla ON rm_conversation (organisation_id, sla_status, sla_next_due)
    WHERE sla_status IN ('at_risk', 'breached');
```

### Message Projection

```sql
CREATE TABLE rm_message (
    id              UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    author_type     TEXT NOT NULL,
    author_id       UUID,
    author_name     TEXT,
    message_type    TEXT NOT NULL,
    direction       TEXT NOT NULL,
    from_address    TEXT,
    to_addresses    TEXT[],
    cc_addresses    TEXT[],
    subject         TEXT,
    body_plain      TEXT,
    body_html       TEXT,
    is_draft        BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    -- AI enrichment
    sentiment       TEXT,
    sentiment_score NUMERIC(4,3),
    intent          TEXT,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_message_conv ON rm_message (conversation_id, created_at);
```

### SLA Projection

```sql
CREATE TABLE rm_sla_instance (
    id              UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    sla_policy_id   UUID NOT NULL,
    sla_policy_name TEXT NOT NULL,
    priority        TEXT NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    first_response_due TIMESTAMPTZ NOT NULL,
    first_response_at  TIMESTAMPTZ,
    resolution_due  TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    status          TEXT NOT NULL,                          -- active, met, breached, paused
    breach_count    INTEGER NOT NULL DEFAULT 0,
    pause_duration_seconds INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_sla_due ON rm_sla_instance (first_response_due) WHERE status = 'active';
CREATE INDEX idx_rm_sla_conv ON rm_sla_instance (conversation_id);
```

### Contact Projection

```sql
CREATE TABLE rm_contact (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    display_name    TEXT,
    company_name    TEXT,
    primary_email   TEXT,
    all_handles     JSONB NOT NULL DEFAULT '[]',           -- [{type, handle}]
    conversation_count INTEGER NOT NULL DEFAULT 0,
    last_seen_at    TIMESTAMPTZ,
    is_anonymised   BOOLEAN NOT NULL DEFAULT FALSE,
    -- Encryption key reference for crypto-shredding (GDPR)
    encryption_key_id UUID,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_contact_org ON rm_contact (organisation_id);
CREATE INDEX idx_rm_contact_email ON rm_contact (primary_email);
```

---

## Supporting Tables (Mutable — Not Event-Sourced)

Some reference data is mutable and does not benefit from event sourcing.

```sql
-- Organisation (tenant) configuration
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    data_residency  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

-- Roles
CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    team_id UUID NOT NULL REFERENCES team(id),
    user_id UUID NOT NULL REFERENCES app_user(id),
    PRIMARY KEY (team_id, user_id)
);

-- Inboxes
CREATE TABLE inbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    email_address   TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- SLA policy definitions (reference data)
CREATE TABLE sla_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    priority        TEXT NOT NULL,
    sla_type        TEXT NOT NULL DEFAULT 'incident',
    first_response_target INTEGER NOT NULL,
    resolution_target INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tags (reference data)
CREATE TABLE tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    colour          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Connected accounts
CREATE TABLE connected_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    provider        TEXT NOT NULL,
    email_address   TEXT NOT NULL,
    oauth_token     TEXT,
    oauth_refresh   TEXT,
    sync_state      TEXT NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Automation rules
CREATE TABLE automation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    trigger_event   TEXT NOT NULL,
    conditions      JSONB NOT NULL DEFAULT '[]',
    actions         JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Knowledge base
CREATE TABLE kb_article (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    title           TEXT NOT NULL,
    body_html       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Encryption keys for crypto-shredding (GDPR)
CREATE TABLE encryption_key (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL,
    key_material    BYTEA NOT NULL,                        -- encrypted with master key
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    shredded_at     TIMESTAMPTZ,                           -- NULL until GDPR deletion
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Projection Rebuild Process

```sql
-- Example: rebuild the conversation projection from events
-- This would typically run in application code, but the SQL pattern is:

-- Step 1: Truncate the projection
TRUNCATE rm_conversation;

-- Step 2: Replay events in order
-- (pseudocode — in practice this runs in application code)
--
-- FOR each event IN (SELECT * FROM event_store WHERE stream_type = 'conversation' ORDER BY created_at, event_version):
--   CASE event.event_type:
--     'conversation.created' -> INSERT INTO rm_conversation
--     'conversation.assigned' -> UPDATE rm_conversation SET assignee_id = payload->>'assignee_id'
--     'conversation.status_changed' -> UPDATE rm_conversation SET status = payload->>'new_status'
--     'message.received' -> UPDATE rm_conversation SET message_count = message_count + 1, last_message_at = event.created_at
--     ...
```

## Temporal Query Examples

```sql
-- "What was the state of conversation X at 3pm last Tuesday?"
-- Replay events up to that point:
SELECT *
FROM event_store
WHERE stream_id = 'conversation-uuid'
  AND created_at <= '2026-05-13 15:00:00+00'
ORDER BY event_version;

-- "How many SLA breaches occurred in April 2026?"
SELECT COUNT(*)
FROM event_store
WHERE event_type = 'sla.first_response_breached'
  AND organisation_id = 'org-uuid'
  AND created_at BETWEEN '2026-04-01' AND '2026-05-01';

-- "Who has handled the most conversations this week?"
SELECT payload->>'assignee_id' AS assignee_id, COUNT(*) AS assignments
FROM event_store
WHERE event_type = 'conversation.assigned'
  AND organisation_id = 'org-uuid'
  AND created_at >= now() - INTERVAL '7 days'
GROUP BY payload->>'assignee_id'
ORDER BY assignments DESC;

-- "Show me the full timeline for conversation X"
SELECT event_type, payload, metadata, created_at
FROM event_store
WHERE stream_id = 'conversation-uuid'
ORDER BY event_version;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | event_store (the single source of truth) |
| Read Model Projections | 4 | rm_conversation, rm_message, rm_sla_instance, rm_contact |
| Identity & Multi-Tenancy | 6 | organisation, app_user, role, user_role, team, team_member |
| Reference Data | 6 | inbox, sla_policy, tag, connected_account, automation_rule, kb_article |
| GDPR Support | 1 | encryption_key (crypto-shredding) |
| **Total** | **18** | Plus event store partitions in production |

---

## Key Design Decisions

1. **Single event store table** — All domain events go to one table, partitioned by `created_at` for time-range queries. The `stream_type` column enables selective replay (e.g., rebuild only conversation projections). This is simpler than separate event tables per aggregate.

2. **CQRS separation** — Write operations append to `event_store`; read operations query `rm_*` projection tables. This allows independent optimization: the event store is append-optimised, while projections have rich indexes for list/filter/sort queries.

3. **Crypto-shredding for GDPR** — Rather than mutating or deleting events (which breaks the immutability guarantee), each contact's PII in event payloads is encrypted with a per-contact key. To comply with GDPR Article 17 (right to erasure), the encryption key is destroyed ("shredded"), making the PII in events unrecoverable while preserving the event stream structure.

4. **Correlation and causation IDs** — Every event's metadata includes `correlation_id` (links all events triggered by the same user action) and `causation_id` (the specific event that caused this one). This enables full causal chain analysis: "the automation rule fired because a message was received, which triggered an SLA start."

5. **Reference data is mutable** — Not everything benefits from event sourcing. Organisation settings, SLA policy definitions, tag lists, and automation rules are mutable reference data stored in standard relational tables. Only domain state changes (conversations, messages, SLA tracking, contacts) are event-sourced.

6. **Projections are disposable** — Any `rm_*` table can be truncated and rebuilt from the event store. This means schema changes to read models are zero-downtime: deploy new projection code, rebuild the table, swap traffic.

7. **Event type taxonomy follows domain language** — Event types use `aggregate.verb_past_tense` format (e.g., `conversation.assigned`, `sla.breached`). This makes the event stream human-readable and directly maps to webhook event types for external integrations.

8. **Native analytics on the event stream** — SLA breach analysis, team performance metrics, and AI training data can all be derived directly from the event store without additional ETL pipelines. The event stream is both the operational log and the analytics source.
