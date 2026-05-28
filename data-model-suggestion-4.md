# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Email Collaboration Platform · Created: 2026-05-20

## Philosophy

This model combines relational tables for operational CRUD with a property graph layer for relationship-heavy queries. The core insight is that an email collaboration platform is fundamentally a relationship network: contacts email each other, agents collaborate on conversations, conversations reference other conversations, contacts belong to companies that have relationships with other companies, and email threads form reply chains via In-Reply-To and References headers.

Traditional relational models handle one-to-many and simple many-to-many relationships well, but struggle with queries like "find all conversations involving contacts at companies we've had escalations with in the last 90 days" or "show me the full reply chain including forwarded branches." These multi-hop traversal queries require recursive CTEs in SQL that are complex and slow. A graph layer handles them natively.

The implementation uses PostgreSQL's `ltree` extension for hierarchical paths (conversation threading, tag hierarchies, team hierarchies) and a generic `graph_edge` table for arbitrary typed relationships between any two entities. This keeps the entire system in PostgreSQL — no separate graph database needed — while enabling graph-style queries via ltree operators and recursive CTEs on the edge table.

**Best for:** Platforms where relationship discovery, network analysis, and multi-hop queries are important — especially for B2B customer success (account hierarchies), conflict-of-interest detection, and intelligent routing based on relationship history.

**Trade-offs:**
- (+) Multi-hop relationship queries are fast and natural
- (+) Email threading via ltree captures branching reply chains accurately
- (+) Contact/company relationship networks enable intelligent routing and context enrichment
- (+) Conversation linking (related, merged, forwarded) is a first-class operation
- (+) All in PostgreSQL — no separate graph database infrastructure
- (-) Graph edge table adds complexity — developers must understand the edge model
- (-) ltree paths must be maintained on INSERT/UPDATE — adds write overhead
- (-) Generic edge table is harder to reason about than explicit foreign keys
- (-) Graph queries can be expensive without careful indexing
- (-) Less common pattern — harder to hire for than standard relational

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JMAP RFC 8621 | Thread entity modeled as ltree hierarchy — JMAP thread grouping maps to ltree ancestor queries |
| RFC 5322 | Message-ID, In-Reply-To, References headers define the email reply graph — stored as graph edges |
| RFC 8621 (keywords) | JMAP keywords (seen, flagged, answered) stored as properties on message nodes |
| ITIL v4 | SLA policies linked to conversations via graph edges — enables SLA inheritance through account hierarchies |
| ISO 3166-1 | Contact and company location modeled as graph properties for jurisdiction-aware queries |
| GDPR Article 17 | Graph edges to anonymised contacts are preserved (for structural integrity) but PII is removed from node properties |
| SCIM 2.0 | User-to-team and user-to-role relationships modeled as graph edges for flexible hierarchy queries |

---

## Graph Infrastructure

```sql
-- Enable ltree extension for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;

-- Entity registry: every first-class object in the system is a node
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    node_type       TEXT NOT NULL,                          -- conversation, message, contact, company, user, team, inbox, tag
    label           TEXT,                                    -- human-readable label for display
    properties      JSONB NOT NULL DEFAULT '{}',           -- type-specific properties
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_node_org_type ON graph_node (organisation_id, node_type);
CREATE INDEX idx_node_properties ON graph_node USING GIN (properties jsonb_path_ops);

-- Typed, directed relationships between any two nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    edge_type       TEXT NOT NULL,                          -- see edge type taxonomy below
    properties      JSONB NOT NULL DEFAULT '{}',           -- edge-specific metadata
    weight          NUMERIC(6,3) DEFAULT 1.0,              -- for ranking/scoring queries
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                            -- NULL = current; set to end temporal edges
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_edge_source ON graph_edge (source_id, edge_type);
CREATE INDEX idx_edge_target ON graph_edge (target_id, edge_type);
CREATE INDEX idx_edge_type ON graph_edge (organisation_id, edge_type);
CREATE INDEX idx_edge_temporal ON graph_edge (valid_from, valid_to) WHERE valid_to IS NOT NULL;
```

### Edge Type Taxonomy

```
-- Conversation relationships
REPLIED_TO          -- message -> message (email reply chain)
FORWARDED_FROM      -- message -> message (forwarded email)
BELONGS_TO          -- message -> conversation
IN_INBOX            -- conversation -> inbox
TAGGED_WITH         -- conversation -> tag
ASSIGNED_TO         -- conversation -> user
ESCALATED_TO        -- conversation -> user (with properties: reason, timestamp)
RELATED_TO          -- conversation -> conversation (linked conversations)
MERGED_INTO         -- conversation -> conversation (merge target)
HAS_SLA             -- conversation -> sla_policy

-- Contact/company relationships
WORKS_AT            -- contact -> company
MANAGES             -- contact -> contact (reporting hierarchy)
CONTACTED_BY        -- contact -> conversation (participated in)
SENT                -- contact -> message
RECEIVED            -- message -> contact

-- User/team relationships
MEMBER_OF           -- user -> team
HAS_ROLE            -- user -> role
MANAGES_TEAM        -- user -> team

-- Company relationships
SUBSIDIARY_OF       -- company -> company
PARTNER_OF          -- company -> company
CUSTOMER_OF         -- company -> organisation (our tenant)

-- Knowledge base
REFERENCES_ARTICLE  -- conversation -> kb_article (suggested/linked)
```

---

## Operational Tables (Relational Core)

```sql
-- Organisations (tenants)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),        -- link to graph node
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

-- Inboxes
CREATE TABLE inbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    email_address   TEXT NOT NULL,
    provider_config JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Conversations
CREATE TABLE conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    subject         TEXT NOT NULL DEFAULT '',
    status          TEXT NOT NULL DEFAULT 'open',
    priority        TEXT NOT NULL DEFAULT 'normal',
    channel         TEXT NOT NULL DEFAULT 'email',
    assignee_id     UUID REFERENCES app_user(id),
    -- Thread path for hierarchical queries (ltree)
    thread_path     ltree,
    -- Denormalised for list views
    last_message_at  TIMESTAMPTZ,
    message_count   INTEGER NOT NULL DEFAULT 0,
    -- SLA summary
    sla_status      TEXT,
    sla_next_due    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conv_org_status ON conversation (organisation_id, status, last_message_at DESC);
CREATE INDEX idx_conv_thread ON conversation USING GIST (thread_path);
CREATE INDEX idx_conv_assignee ON conversation (assignee_id) WHERE assignee_id IS NOT NULL;

-- Messages
CREATE TABLE message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    conversation_id UUID NOT NULL REFERENCES conversation(id),
    author_type     TEXT NOT NULL,
    author_id       UUID,
    message_type    TEXT NOT NULL DEFAULT 'reply',
    direction       TEXT NOT NULL DEFAULT 'inbound',
    -- RFC 5322 fields
    rfc_message_id  TEXT,
    in_reply_to     TEXT,
    reference_ids   TEXT[],
    from_address    TEXT,
    to_addresses    TEXT[],
    cc_addresses    TEXT[],
    subject         TEXT,
    body_plain      TEXT,
    body_html       TEXT,
    -- Thread position (ltree path within conversation)
    reply_path      ltree,
    -- Metadata
    headers         JSONB NOT NULL DEFAULT '{}',
    attachments     JSONB NOT NULL DEFAULT '[]',
    ai              JSONB NOT NULL DEFAULT '{}',
    is_draft        BOOLEAN NOT NULL DEFAULT FALSE,
    sent_at         TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_msg_conv ON message (conversation_id, created_at);
CREATE INDEX idx_msg_reply_path ON message USING GIST (reply_path);
CREATE INDEX idx_msg_rfc_id ON message (rfc_message_id) WHERE rfc_message_id IS NOT NULL;

-- Contacts
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    display_name    TEXT,
    primary_email   TEXT,
    handles         JSONB NOT NULL DEFAULT '[]',
    company_node_id UUID REFERENCES graph_node(id),        -- link to company graph node
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    is_anonymised   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contact_org ON contact (organisation_id);
CREATE INDEX idx_contact_email ON contact (primary_email);

-- Companies (separate from contacts — B2B account-level entity)
CREATE TABLE company (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    domain          TEXT,
    industry        TEXT,
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_company_org ON company (organisation_id);
CREATE INDEX idx_company_domain ON company (domain) WHERE domain IS NOT NULL;
```

---

## SLA, Tags, and Supporting Tables

```sql
CREATE TABLE tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    colour          TEXT,
    -- Hierarchical tags via ltree
    tag_path        ltree,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, name)
);
CREATE INDEX idx_tag_path ON tag USING GIST (tag_path);

CREATE TABLE sla_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    priority        TEXT NOT NULL,
    targets         JSONB NOT NULL,
    conditions      JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE automation_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    rule            JSONB NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE canned_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    body_html       TEXT NOT NULL,
    body_plain      TEXT,
    category        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE kb_article (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID REFERENCES graph_node(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    title           TEXT NOT NULL,
    slug            TEXT NOT NULL,
    body_html       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',
    search_vector   TSVECTOR,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kb_search ON kb_article USING GIN (search_vector);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    actor_id        UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID NOT NULL,
    changes         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org_time ON audit_log (organisation_id, created_at DESC);

CREATE TABLE notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    type            TEXT NOT NULL,
    title           TEXT NOT NULL,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Graph Query Examples

### Email Reply Chain (Branching Thread Tree)

```sql
-- Build the reply chain for a message using graph edges
-- Find all messages that this message is a reply to (ancestors)
WITH RECURSIVE reply_chain AS (
    -- Start with the current message
    SELECT m.id, m.subject, m.from_address, m.created_at, 0 AS depth
    FROM message m
    WHERE m.id = 'message-uuid'

    UNION ALL

    -- Walk up the REPLIED_TO edges
    SELECT m.id, m.subject, m.from_address, m.created_at, rc.depth + 1
    FROM reply_chain rc
    JOIN graph_edge e ON e.source_id = (SELECT node_id FROM message WHERE id = rc.id)
        AND e.edge_type = 'REPLIED_TO'
    JOIN graph_node gn ON gn.id = e.target_id
    JOIN message m ON m.node_id = gn.id
    WHERE rc.depth < 50  -- safety limit
)
SELECT * FROM reply_chain ORDER BY depth DESC;

-- Alternative using ltree (faster, pre-computed):
-- Find all messages in the same thread subtree
SELECT m.*
FROM message m
WHERE m.reply_path <@ (SELECT reply_path FROM message WHERE id = 'message-uuid')
ORDER BY m.reply_path;
```

### Contact Network: Find All Contacts at Related Companies

```sql
-- "Show me all contacts at companies that are subsidiaries of Acme Corp"
WITH RECURSIVE company_tree AS (
    SELECT gn.id AS node_id, c.name, 0 AS depth
    FROM company c
    JOIN graph_node gn ON gn.id = c.node_id
    WHERE c.name = 'Acme Corp' AND c.organisation_id = 'org-uuid'

    UNION ALL

    SELECT e.source_id, c2.name, ct.depth + 1
    FROM company_tree ct
    JOIN graph_edge e ON e.target_id = ct.node_id AND e.edge_type = 'SUBSIDIARY_OF'
    JOIN graph_node gn ON gn.id = e.source_id
    JOIN company c2 ON c2.node_id = gn.id
    WHERE ct.depth < 10
)
SELECT ct.name AS company_name, con.display_name, con.primary_email
FROM company_tree ct
JOIN graph_edge e ON e.target_id = ct.node_id AND e.edge_type = 'WORKS_AT'
JOIN graph_node gn ON gn.id = e.source_id
JOIN contact con ON con.node_id = gn.id
ORDER BY ct.depth, con.display_name;
```

### Intelligent Routing: Find Best Agent Based on Relationship History

```sql
-- "Which agents have successfully resolved conversations for contacts at this company?"
SELECT
    u.display_name AS agent,
    COUNT(*) AS resolved_count,
    AVG(EXTRACT(EPOCH FROM (c.updated_at - c.created_at))) AS avg_resolution_seconds
FROM graph_edge company_contact
JOIN graph_edge contact_conv ON contact_conv.source_id = company_contact.source_id
    AND contact_conv.edge_type = 'CONTACTED_BY'
JOIN conversation c ON c.node_id = contact_conv.target_id
    AND c.status = 'closed'
JOIN graph_edge conv_agent ON conv_agent.source_id = contact_conv.target_id
    AND conv_agent.edge_type = 'ASSIGNED_TO'
JOIN app_user u ON u.node_id = conv_agent.target_id
WHERE company_contact.target_id = (SELECT node_id FROM company WHERE domain = 'acme.com' AND organisation_id = 'org-uuid')
    AND company_contact.edge_type = 'WORKS_AT'
GROUP BY u.display_name
ORDER BY resolved_count DESC;
```

### Conversation Linking and Merging

```sql
-- Find all conversations related to a given conversation (including merged ones)
WITH RECURSIVE related AS (
    SELECT c.id, c.subject, c.status, 'origin' AS relation, 0 AS depth
    FROM conversation c WHERE c.id = 'conv-uuid'

    UNION ALL

    SELECT c2.id, c2.subject, c2.status,
           e.edge_type AS relation, r.depth + 1
    FROM related r
    JOIN graph_node gn ON gn.id = (SELECT node_id FROM conversation WHERE id = r.id)
    JOIN graph_edge e ON (e.source_id = gn.id OR e.target_id = gn.id)
        AND e.edge_type IN ('RELATED_TO', 'MERGED_INTO')
    JOIN graph_node gn2 ON gn2.id = CASE WHEN e.source_id = gn.id THEN e.target_id ELSE e.source_id END
    JOIN conversation c2 ON c2.node_id = gn2.id
    WHERE r.depth < 5
)
SELECT DISTINCT ON (id) * FROM related ORDER BY id, depth;
```

### Hierarchical Tags via ltree

```sql
-- Find all conversations tagged with "billing" or any sub-tag (billing.refunds, billing.invoices, etc.)
SELECT c.*
FROM conversation c
JOIN graph_edge e ON e.source_id = c.node_id AND e.edge_type = 'TAGGED_WITH'
JOIN tag t ON t.node_id = e.target_id
WHERE t.tag_path <@ 'billing'
  AND c.organisation_id = 'org-uuid';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | graph_node, graph_edge |
| Identity & Multi-Tenancy | 1 | organisation (users, teams via graph) |
| Operational Entities | 6 | app_user, inbox, conversation, message, contact, company |
| Tags & SLA | 2 | tag, sla_policy |
| Automation | 2 | automation_rule, canned_response |
| Knowledge Base | 1 | kb_article |
| Audit & Notifications | 3 | audit_log, notification, webhook |
| **Total** | **17** | Plus graph_node/graph_edge carry most relationship data |

---

## Key Design Decisions

1. **Dual representation: relational + graph** — Every major entity has both a relational table (for fast CRUD, strong typing, and indexed queries) and a `graph_node` reference (for relationship traversal). The relational table is the source of truth for entity properties; the graph layer is the source of truth for relationships.

2. **Generic edge table with typed relationships** — The `graph_edge` table uses an `edge_type` column to distinguish between REPLIED_TO, WORKS_AT, ASSIGNED_TO, etc. This enables relationship queries that span types: "find agents who have handled conversations from contacts at this company" without knowing the specific relationship types in advance.

3. **ltree for hierarchical data** — Email threading (reply chains), tag hierarchies, and company hierarchies use PostgreSQL's ltree extension. This enables ancestor/descendant queries with `<@` and `@>` operators that are dramatically faster than recursive CTEs for deep hierarchies.

4. **Temporal edges** — The `valid_from`/`valid_to` columns on graph_edge enable temporal relationship queries: "who was assigned to this conversation on Tuesday?" or "which contacts worked at Acme Corp in Q1 2026?" This is essential for an email platform where assignments, team memberships, and contact relationships change frequently.

5. **Company as a first-class entity** — Unlike the other models which embed company info in the contact, this model elevates Company to its own entity with a graph node. This enables B2B account hierarchies (SUBSIDIARY_OF), partnership networks (PARTNER_OF), and company-level SLA tracking that the account-based organisation features demand.

6. **Weight on edges for ranking** — The `weight` column on graph_edge enables PageRank-style queries: "which contacts are most important based on conversation volume and recency?" Weights can be updated by a background process that analyses interaction patterns.

7. **Conversation linking via graph edges** — Instead of a `merged_with` column or a separate linking table, conversation relationships (related, merged, forwarded from) are all graph edges. This enables transitive queries: "show me all conversations connected to this one, including those connected through intermediaries."

8. **Graph-powered intelligent routing** — The graph structure enables AI routing decisions based on relationship context: assign conversations from contacts at Company X to agents who have the best resolution track record with that company's contacts. This is a significant differentiator over rule-based assignment.
