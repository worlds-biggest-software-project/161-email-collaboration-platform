# Email Collaboration Platform — Phased Development Plan

> Project: 161-email-collaboration-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22+) | Full-stack unified language; strong async I/O for email sync workers; excellent type safety for complex domain models; best ecosystem for real-time WebSocket collaboration (collision detection, live presence) |
| API framework | Fastify 5 with @fastify/swagger | High-performance HTTP framework; native OpenAPI 3.1 schema generation; plugin architecture suits multi-tenant middleware; first-class TypeScript support |
| Frontend | Next.js 15 (App Router) with React 19 | Server-side rendering for fast inbox loads; React Server Components reduce client bundle; App Router enables streaming for large conversation lists; shared TypeScript types with backend |
| Database | PostgreSQL 16 | JMAP-aligned relational model requires strong referential integrity; JSONB for flexible metadata (provider configs, AI enrichment, custom fields); GIN indexes for tag/handle lookups; ltree extension for thread hierarchies; row-level security for multi-tenancy |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead; native PostgreSQL JSONB and array support; migration tooling; avoids the abstraction penalty of Prisma for complex queries |
| Task queue | BullMQ (Redis-backed) | Email sync, SLA breach checks, AI enrichment, and webhook delivery are all async workloads; BullMQ provides delayed jobs (SLA timers), rate limiting (API quotas), and retry with backoff; Redis also serves as pub/sub for real-time events |
| Real-time | Socket.IO 4 | Collision detection, live typing indicators, and inbox updates require bidirectional real-time; Socket.IO handles reconnection, rooms (per-conversation, per-inbox), and namespaces (per-organisation) |
| Cache | Redis 7 (shared with BullMQ) | Session store, conversation list cache, contact lookup cache, rate limiting counters |
| Object storage | S3-compatible (MinIO for self-hosted) | Email attachments, knowledge base images; presigned URLs for client-side upload/download |
| Search | PostgreSQL full-text search (tsvector) | Avoids Elasticsearch operational burden for MVP; pg_trgm for fuzzy matching; upgrade path to Meilisearch or Typesense in later phases if needed |
| AI / LLM | OpenAI-compatible API (gpt-4o default) via Vercel AI SDK | AI SDK provides streaming, tool use, and provider abstraction; supports OpenAI, Anthropic, local models; used for reply drafting, sentiment analysis, intent classification, SLA prediction |
| Email protocol | IMAP/SMTP via imapflow + nodemailer; Gmail API via googleapis; Microsoft Graph via @microsoft/microsoft-graph-client | imapflow is the best Node.js IMAP library (modern, stream-based); nodemailer for SMTP sending; native API clients for Gmail and Outlook provide push notifications and richer metadata |
| Authentication | Lucia Auth with Arctic (OAuth providers) | Lightweight, session-based auth with adapter for PostgreSQL; Arctic handles OAuth 2.0 flows for Google, Microsoft, SAML via passport-saml |
| Containerisation | Docker + docker-compose | Self-hosted deployment target; compose orchestrates app, PostgreSQL, Redis, MinIO; multi-stage builds for production images |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest is fastest TypeScript test runner; native ESM, mocking, coverage; Playwright for browser-based E2E testing of the inbox UI |
| Code quality | ESLint 9 (flat config) + Prettier + tsc --noEmit | Standard TypeScript toolchain; strict mode enabled; Prettier for consistent formatting |
| Package manager | pnpm 9 | Workspace support for monorepo (shared types between frontend and backend); faster installs; strict dependency resolution |
| CI | GitHub Actions | Standard for open-source; matrix testing across Node versions; Docker image build and push |

### Data Model Selection

Data Model Suggestion 3 (Hybrid Relational + JSONB) is adopted as the primary schema design. Rationale:

1. **Fewest tables (15)** reduces migration complexity and speeds MVP delivery.
2. **JSONB columns** for provider configs, AI enrichment, SLA state, and custom fields enable rapid iteration without schema migrations.
3. **GIN indexes on JSONB** maintain query performance for the fields that matter (tag filtering, handle lookups, SLA breach queries).
4. **Channel-agnostic message model** via JSONB headers naturally supports future SMS/chat channels without schema changes.
5. **Custom fields per tenant** are trivial with JSONB, critical for a multi-tenant SaaS.

Select elements from Data Model 1 (normalized SLA management tables) are incorporated for the SLA subsystem where referential integrity matters most. The audit log follows Data Model 2's event-style pattern (append-only with JSONB changes column) to satisfy compliance requirements without full event sourcing complexity.

### Project Structure

```
email-collaboration-platform/
├── pnpm-workspace.yaml
├── package.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── docker-publish.yml
├── packages/
│   └── shared/                          # Shared types, constants, validation
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── types/
│           │   ├── conversation.ts
│           │   ├── message.ts
│           │   ├── contact.ts
│           │   ├── inbox.ts
│           │   ├── sla.ts
│           │   ├── user.ts
│           │   └── index.ts
│           ├── constants/
│           │   ├── sla.ts
│           │   ├── channels.ts
│           │   └── permissions.ts
│           └── validation/
│               ├── conversation.ts
│               └── message.ts
├── apps/
│   ├── api/                             # Fastify backend
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── drizzle.config.ts
│   │   └── src/
│   │       ├── index.ts                 # Server entry point
│   │       ├── config.ts                # Environment config with defaults
│   │       ├── db/
│   │       │   ├── schema.ts            # Drizzle schema definitions
│   │       │   ├── migrate.ts
│   │       │   └── migrations/
│   │       ├── plugins/
│   │       │   ├── auth.ts              # Authentication plugin
│   │       │   ├── multi-tenant.ts      # Org scoping plugin
│   │       │   ├── rate-limit.ts
│   │       │   └── websocket.ts         # Socket.IO setup
│   │       ├── routes/
│   │       │   ├── auth/
│   │       │   ├── conversations/
│   │       │   ├── messages/
│   │       │   ├── contacts/
│   │       │   ├── inboxes/
│   │       │   ├── sla/
│   │       │   ├── tags/
│   │       │   ├── automation/
│   │       │   ├── knowledge-base/
│   │       │   ├── ai/
│   │       │   ├── webhooks/
│   │       │   └── admin/
│   │       ├── services/
│   │       │   ├── conversation.service.ts
│   │       │   ├── message.service.ts
│   │       │   ├── contact.service.ts
│   │       │   ├── sla.service.ts
│   │       │   ├── ai.service.ts
│   │       │   ├── automation.service.ts
│   │       │   └── notification.service.ts
│   │       ├── workers/
│   │       │   ├── email-sync.worker.ts
│   │       │   ├── sla-check.worker.ts
│   │       │   ├── ai-enrichment.worker.ts
│   │       │   ├── webhook-delivery.worker.ts
│   │       │   └── email-send.worker.ts
│   │       ├── integrations/
│   │       │   ├── gmail/
│   │       │   ├── outlook/
│   │       │   ├── imap/
│   │       │   ├── smtp/
│   │       │   ├── slack/
│   │       │   └── crm/
│   │       └── lib/
│   │           ├── email-parser.ts      # MIME parsing utilities
│   │           ├── sla-calculator.ts    # Business hours SLA math
│   │           ├── thread-builder.ts    # RFC 5322 threading
│   │           └── crypto.ts            # Token encryption
│   └── web/                             # Next.js frontend
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       └── src/
│           ├── app/
│           │   ├── layout.tsx
│           │   ├── page.tsx
│           │   ├── (auth)/
│           │   │   ├── login/
│           │   │   └── signup/
│           │   └── (dashboard)/
│           │       ├── layout.tsx
│           │       ├── inbox/
│           │       │   ├── [inboxId]/
│           │       │   │   └── page.tsx
│           │       │   └── page.tsx
│           │       ├── conversation/
│           │       │   └── [conversationId]/
│           │       │       └── page.tsx
│           │       ├── contacts/
│           │       ├── analytics/
│           │       ├── settings/
│           │       │   ├── inboxes/
│           │       │   ├── sla/
│           │       │   ├── automation/
│           │       │   ├── team/
│           │       │   └── integrations/
│           │       └── knowledge-base/
│           ├── components/
│           │   ├── inbox/
│           │   ├── conversation/
│           │   ├── message/
│           │   ├── contact/
│           │   ├── sla/
│           │   ├── ai/
│           │   └── ui/                  # Shared UI primitives
│           ├── hooks/
│           ├── lib/
│           │   ├── api-client.ts
│           │   └── socket.ts
│           └── stores/
│               ├── inbox.store.ts
│               └── conversation.store.ts
└── tests/
    ├── fixtures/
    │   ├── emails/                      # Sample .eml files
    │   ├── mime/                         # MIME multipart samples
    │   └── seed.sql                     # Test data seeding
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation — Project Scaffolding, Database, and Authentication

### Purpose

Establish the monorepo structure, database schema, authentication system, and core multi-tenancy primitives. After this phase, a developer can sign up, create an organisation, invite team members, and authenticate via email/password and OAuth. The database schema is deployed and migrations are working. Docker compose brings up the full stack locally.

### Tasks

#### 1.1 — Monorepo Scaffolding and Build System

**What**: Set up the pnpm workspace with api, web, and shared packages; configure TypeScript, ESLint, Prettier, and Vitest across all packages.

**Design**:

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

```jsonc
// Root package.json scripts
{
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck",
    "db:migrate": "pnpm --filter api db:migrate",
    "db:generate": "pnpm --filter api db:generate",
    "docker:up": "docker compose up -d",
    "docker:down": "docker compose down"
  }
}
```

```typescript
// packages/shared/src/types/user.ts
export interface User {
  id: string;
  organisationId: string;
  email: string;
  displayName: string;
  avatarUrl: string | null;
  status: 'active' | 'deactivated' | 'invited';
  role: 'admin' | 'agent' | 'viewer';
  permissions: string[];
  profile: Record<string, unknown>;
  createdAt: Date;
  updatedAt: Date;
}

export interface Organisation {
  id: string;
  name: string;
  slug: string;
  plan: 'free' | 'starter' | 'professional' | 'enterprise';
  settings: OrganisationSettings;
  createdAt: Date;
  updatedAt: Date;
}

export interface OrganisationSettings {
  dataResidency?: string;
  timezone: string;
  businessHours: Record<string, { start: string; end: string }>;
  branding: { logoUrl?: string; primaryColour?: string };
  features: { aiSuggestions: boolean; sentimentAnalysis: boolean };
  customConversationFields: CustomFieldDefinition[];
}

export interface CustomFieldDefinition {
  key: string;
  label: string;
  type: 'text' | 'number' | 'select' | 'date' | 'boolean';
  options?: string[];
  required?: boolean;
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: email_collab
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD:-localdev}
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  api:
    build:
      context: .
      target: api
    depends_on: [postgres, redis, minio]
    ports: ["3001:3001"]
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD:-localdev}@postgres:5432/email_collab
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000

  web:
    build:
      context: .
      target: web
    depends_on: [api]
    ports: ["3000:3000"]
    environment:
      NEXT_PUBLIC_API_URL: http://api:3001

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Unit: shared types compile with strict TypeScript — no errors`
- `Unit: pnpm build — all three packages build successfully`
- `Integration: docker compose up — all services healthy within 30s`
- `Integration: pnpm lint — zero warnings across all packages`
- `Integration: pnpm typecheck — zero errors across all packages`

---

#### 1.2 — Database Schema and Migrations

**What**: Implement the PostgreSQL schema using Drizzle ORM with all core tables from the hybrid relational + JSONB data model.

**Design**:

```typescript
// apps/api/src/db/schema.ts
import { pgTable, uuid, text, boolean, timestamp, integer,
         jsonb, index, uniqueIndex } from 'drizzle-orm/pg-core';

export const organisation = pgTable('organisation', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  plan: text('plan').notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const appUser = pgTable('app_user', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  email: text('email').notNull(),
  displayName: text('display_name').notNull(),
  avatarUrl: text('avatar_url'),
  status: text('status').notNull().default('active'),
  role: text('role').notNull().default('agent'),
  permissions: text('permissions').array().notNull().default([]),
  passwordHash: text('password_hash'),
  profile: jsonb('profile').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_app_user_org_email').on(table.organisationId, table.email),
  index('idx_app_user_org').on(table.organisationId),
]);

export const team = pgTable('team', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  memberIds: uuid('member_ids').array().notNull().default([]),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_team_org').on(table.organisationId),
]);

export const inbox = pgTable('inbox', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  emailAddress: text('email_address').notNull(),
  channelType: text('channel_type').notNull().default('email'),
  isActive: boolean('is_active').notNull().default(true),
  providerConfig: jsonb('provider_config').notNull().default({}),
  accessUserIds: uuid('access_user_ids').array().notNull().default([]),
  accessTeamIds: uuid('access_team_ids').array().notNull().default([]),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_inbox_org').on(table.organisationId),
]);

export const conversation = pgTable('conversation', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  inboxId: uuid('inbox_id').notNull().references(() => inbox.id),
  subject: text('subject').notNull().default(''),
  status: text('status').notNull().default('open'),
  priority: text('priority').notNull().default('normal'),
  channel: text('channel').notNull().default('email'),
  assigneeId: uuid('assignee_id').references(() => appUser.id),
  teamId: uuid('team_id').references(() => team.id),
  lastMessageAt: timestamp('last_message_at', { withTimezone: true }),
  lastMessagePreview: text('last_message_preview'),
  messageCount: integer('message_count').notNull().default(0),
  inboxIds: uuid('inbox_ids').array().notNull().default([]),
  tagIds: uuid('tag_ids').array().notNull().default([]),
  participants: jsonb('participants').notNull().default([]),
  customFields: jsonb('custom_fields').notNull().default({}),
  sla: jsonb('sla').notNull().default({}),
  ai: jsonb('ai').notNull().default({}),
  snoozeUntil: timestamp('snooze_until', { withTimezone: true }),
  closedAt: timestamp('closed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_conv_org_status_last').on(table.organisationId, table.status, table.lastMessageAt),
  index('idx_conv_assignee').on(table.assigneeId),
]);

export const message = pgTable('message', {
  id: uuid('id').primaryKey().defaultRandom(),
  conversationId: uuid('conversation_id').notNull().references(() => conversation.id),
  authorType: text('author_type').notNull(),
  authorId: uuid('author_id'),
  messageType: text('message_type').notNull().default('reply'),
  direction: text('direction').notNull().default('inbound'),
  bodyPlain: text('body_plain'),
  bodyHtml: text('body_html'),
  headers: jsonb('headers').notNull().default({}),
  attachments: jsonb('attachments').notNull().default([]),
  ai: jsonb('ai').notNull().default({}),
  channelMeta: jsonb('channel_meta').notNull().default({}),
  isDraft: boolean('is_draft').notNull().default(false),
  sentAt: timestamp('sent_at', { withTimezone: true }),
  receivedAt: timestamp('received_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_message_conv').on(table.conversationId, table.createdAt),
]);

export const contact = pgTable('contact', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  displayName: text('display_name'),
  primaryEmail: text('primary_email'),
  handles: jsonb('handles').notNull().default([]),
  company: jsonb('company').notNull().default({}),
  crm: jsonb('crm').notNull().default({}),
  customFields: jsonb('custom_fields').notNull().default({}),
  conversationCount: integer('conversation_count').notNull().default(0),
  lastContactedAt: timestamp('last_contacted_at', { withTimezone: true }),
  isAnonymised: boolean('is_anonymised').notNull().default(false),
  piiFields: text('pii_fields').array().notNull().default(['display_name', 'primary_email', 'handles']),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_contact_org').on(table.organisationId),
  index('idx_contact_email').on(table.primaryEmail),
]);

export const tag = pgTable('tag', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  colour: text('colour'),
  parentId: uuid('parent_id'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_tag_org_name').on(table.organisationId, table.name),
]);

export const slaPolicy = pgTable('sla_policy', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  description: text('description'),
  priority: text('priority').notNull(),
  targets: jsonb('targets').notNull(),
  conditions: jsonb('conditions').notNull().default([]),
  sortOrder: integer('sort_order').notNull().default(0),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const automationRule = pgTable('automation_rule', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  description: text('description'),
  rule: jsonb('rule').notNull(),
  isActive: boolean('is_active').notNull().default(true),
  runCount: integer('run_count').notNull().default(0),
  lastRunAt: timestamp('last_run_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const cannedResponse = pgTable('canned_response', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  subject: text('subject'),
  bodyHtml: text('body_html').notNull(),
  bodyPlain: text('body_plain'),
  category: text('category'),
  variables: text('variables').array().notNull().default([]),
  createdBy: uuid('created_by').references(() => appUser.id),
  usageCount: integer('usage_count').notNull().default(0),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const kbArticle = pgTable('kb_article', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  collection: text('collection').notNull().default('general'),
  title: text('title').notNull(),
  slug: text('slug').notNull(),
  bodyHtml: text('body_html').notNull(),
  status: text('status').notNull().default('draft'),
  viewCount: integer('view_count').notNull().default(0),
  helpfulCount: integer('helpful_count').notNull().default(0),
  meta: jsonb('meta').notNull().default({}),
  publishedAt: timestamp('published_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_kb_org_slug').on(table.organisationId, table.slug),
]);

export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull(),
  actorId: uuid('actor_id'),
  actorType: text('actor_type').notNull().default('user'),
  action: text('action').notNull(),
  resourceType: text('resource_type').notNull(),
  resourceId: uuid('resource_id').notNull(),
  changes: jsonb('changes').notNull().default({}),
  context: jsonb('context').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_audit_org_time').on(table.organisationId, table.createdAt),
  index('idx_audit_resource').on(table.resourceType, table.resourceId),
]);

export const notification = pgTable('notification', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => appUser.id),
  type: text('type').notNull(),
  title: text('title').notNull(),
  body: text('body'),
  resourceType: text('resource_type'),
  resourceId: uuid('resource_id'),
  isRead: boolean('is_read').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_notif_user').on(table.userId, table.isRead, table.createdAt),
]);

export const webhook = pgTable('webhook', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  url: text('url').notNull(),
  events: text('events').array().notNull(),
  secret: text('secret').notNull(),
  isActive: boolean('is_active').notNull().default(true),
  meta: jsonb('meta').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// Session table for Lucia Auth
export const session = pgTable('session', {
  id: text('id').primaryKey(),
  userId: uuid('user_id').notNull().references(() => appUser.id),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
});
```

**Testing**:
- `Unit: drizzle-kit generate — produces valid SQL migration files without errors`
- `Integration: drizzle-kit migrate — applies migrations to empty PostgreSQL database successfully`
- `Integration: INSERT into organisation — row created with correct defaults (plan='free', settings='{}')`
- `Integration: INSERT into app_user with duplicate (org_id, email) — unique constraint violation raised`
- `Integration: JSONB GIN index on conversation.sla — containment query uses index (EXPLAIN shows idx_conv_sla_due)`
- `Unit: rollback migration — drops all tables cleanly`

---

#### 1.3 — Authentication and Session Management

**What**: Implement email/password signup and login, OAuth 2.0 login with Google and Microsoft, session management, and password reset flow.

**Design**:

```typescript
// apps/api/src/plugins/auth.ts
import { Lucia } from 'lucia';
import { DrizzlePostgreSQLAdapter } from '@lucia-auth/adapter-drizzle';

export function buildLucia(db: DrizzleDatabase) {
  const adapter = new DrizzlePostgreSQLAdapter(db, session, appUser);
  return new Lucia(adapter, {
    sessionCookie: {
      attributes: { secure: process.env.NODE_ENV === 'production' },
    },
    getUserAttributes: (attributes) => ({
      email: attributes.email,
      displayName: attributes.displayName,
      organisationId: attributes.organisationId,
      role: attributes.role,
      permissions: attributes.permissions,
    }),
  });
}
```

API endpoints:

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/api/auth/signup` | `{ email, password, displayName, orgName }` | `{ user, organisation, session }` |
| POST | `/api/auth/login` | `{ email, password }` | `{ user, session }` |
| POST | `/api/auth/logout` | (session cookie) | `204 No Content` |
| GET | `/api/auth/oauth/google` | (redirect) | Redirect to Google consent |
| GET | `/api/auth/oauth/google/callback` | `{ code, state }` | Set session cookie, redirect |
| GET | `/api/auth/oauth/microsoft` | (redirect) | Redirect to Azure AD consent |
| GET | `/api/auth/oauth/microsoft/callback` | `{ code, state }` | Set session cookie, redirect |
| POST | `/api/auth/forgot-password` | `{ email }` | `202 Accepted` |
| POST | `/api/auth/reset-password` | `{ token, newPassword }` | `{ user, session }` |
| GET | `/api/auth/me` | (session cookie) | `{ user, organisation }` |

```typescript
// apps/api/src/routes/auth/signup.ts
import { z } from 'zod';

export const signupSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(128),
  displayName: z.string().min(1).max(100),
  orgName: z.string().min(1).max(100),
});

export type SignupRequest = z.infer<typeof signupSchema>;

// Handler creates organisation, then creates user with role='admin', then creates session
```

**Testing**:
- `Unit: signupSchema — valid payload passes validation`
- `Unit: signupSchema — password < 8 chars rejected with clear error`
- `Unit: signupSchema — missing orgName rejected`
- `Integration: POST /api/auth/signup — creates organisation + user + session, returns 201`
- `Integration: POST /api/auth/signup with duplicate email — returns 409 Conflict`
- `Integration: POST /api/auth/login with valid credentials — returns session cookie and user`
- `Integration: POST /api/auth/login with wrong password — returns 401`
- `Integration: GET /api/auth/me with valid session — returns user and organisation`
- `Integration: GET /api/auth/me with expired session — returns 401`
- `Integration (mocked OAuth): Google callback with valid code — creates user and session`
- `Integration (mocked OAuth): Microsoft callback with valid code — creates user and session`
- `Integration: POST /api/auth/logout — invalidates session, subsequent /me returns 401`

---

#### 1.4 — Multi-Tenancy Middleware and RBAC

**What**: Implement organisation-scoped data access, role-based permission checking, and team management endpoints.

**Design**:

```typescript
// apps/api/src/plugins/multi-tenant.ts
// Fastify plugin that:
// 1. Extracts session from cookie
// 2. Validates session with Lucia
// 3. Sets request.user (with organisationId, role, permissions)
// 4. All subsequent DB queries are scoped to request.user.organisationId

export const PERMISSIONS = {
  INBOX_READ: 'inbox.read',
  INBOX_WRITE: 'inbox.write',
  INBOX_ADMIN: 'inbox.admin',
  CONVERSATION_READ: 'conversation.read',
  CONVERSATION_WRITE: 'conversation.write',
  CONVERSATION_ASSIGN: 'conversation.assign',
  CONTACT_READ: 'contact.read',
  CONTACT_WRITE: 'contact.write',
  SLA_MANAGE: 'sla.manage',
  AUTOMATION_MANAGE: 'automation.manage',
  TEAM_MANAGE: 'team.manage',
  ORG_ADMIN: 'org.admin',
  ANALYTICS_VIEW: 'analytics.view',
  KB_MANAGE: 'kb.manage',
} as const;

export type Permission = typeof PERMISSIONS[keyof typeof PERMISSIONS];

export const ROLE_PERMISSIONS: Record<string, Permission[]> = {
  admin: Object.values(PERMISSIONS),
  agent: [
    PERMISSIONS.INBOX_READ, PERMISSIONS.INBOX_WRITE,
    PERMISSIONS.CONVERSATION_READ, PERMISSIONS.CONVERSATION_WRITE,
    PERMISSIONS.CONVERSATION_ASSIGN,
    PERMISSIONS.CONTACT_READ, PERMISSIONS.CONTACT_WRITE,
    PERMISSIONS.ANALYTICS_VIEW,
  ],
  viewer: [
    PERMISSIONS.INBOX_READ,
    PERMISSIONS.CONVERSATION_READ,
    PERMISSIONS.CONTACT_READ,
  ],
};

// Route-level guard decorator
export function requirePermission(permission: Permission) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const userPerms = [
      ...ROLE_PERMISSIONS[request.user.role],
      ...request.user.permissions,
    ];
    if (!userPerms.includes(permission)) {
      return reply.status(403).send({ error: 'Forbidden', required: permission });
    }
  };
}
```

Team management endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/teams` | List teams in organisation |
| POST | `/api/teams` | Create team (requires team.manage) |
| PATCH | `/api/teams/:id` | Update team name/settings |
| POST | `/api/teams/:id/members` | Add member to team |
| DELETE | `/api/teams/:id/members/:userId` | Remove member from team |
| GET | `/api/users` | List users in organisation |
| POST | `/api/users/invite` | Invite user via email (requires org.admin) |
| PATCH | `/api/users/:id/role` | Change user role (requires org.admin) |

**Testing**:
- `Unit: ROLE_PERMISSIONS — admin has all permissions`
- `Unit: ROLE_PERMISSIONS — viewer lacks conversation.write`
- `Unit: requirePermission — user with correct permission passes`
- `Unit: requirePermission — user without permission gets 403`
- `Integration: GET /api/teams — returns only teams for authenticated user's organisation`
- `Integration: POST /api/teams as agent — returns 403 (requires team.manage)`
- `Integration: POST /api/teams as admin — creates team, returns 201`
- `Integration: GET /api/users — org A cannot see org B's users`
- `Integration: POST /api/users/invite — creates user with status='invited', sends invite email (mocked)`

---

## Phase 2: Email Integration — Connect, Sync, and Send

### Purpose

Connect the platform to real email accounts (Gmail, Outlook, generic IMAP/SMTP), sync existing messages into conversations, and send outbound replies. After this phase, a user can connect their support@company.com inbox and see incoming emails threaded into conversations in the database. Outbound replies are sent through the connected account.

### Tasks

#### 2.1 — Email Provider Abstraction Layer

**What**: Build a provider-agnostic interface for email operations (list messages, fetch message, send message, sync changes) with concrete implementations for Gmail API, Microsoft Graph, and IMAP/SMTP.

**Design**:

```typescript
// apps/api/src/integrations/email-provider.ts

export interface EmailEnvelope {
  messageId: string;          // RFC 5322 Message-ID
  inReplyTo?: string;         // In-Reply-To header
  references: string[];       // References header chain
  from: EmailAddress;
  to: EmailAddress[];
  cc: EmailAddress[];
  bcc: EmailAddress[];
  subject: string;
  date: Date;
  bodyPlain?: string;
  bodyHtml?: string;
  attachments: EmailAttachment[];
  rawHeaders: Record<string, string>;
}

export interface EmailAddress {
  name?: string;
  email: string;
}

export interface EmailAttachment {
  filename: string;
  contentType: string;
  size: number;
  contentId?: string;        // For inline attachments (RFC 2392)
  content?: Buffer;          // Populated on fetch, null on list
}

export interface SyncResult {
  newMessages: EmailEnvelope[];
  updatedMessageIds: string[];
  deletedMessageIds: string[];
  nextSyncToken: string;
}

export interface EmailProvider {
  readonly providerName: string;

  // OAuth flow
  getAuthUrl(redirectUri: string, state: string): string;
  exchangeCode(code: string, redirectUri: string): Promise<OAuthTokens>;
  refreshToken(refreshToken: string): Promise<OAuthTokens>;

  // Sync
  initialSync(credentials: ProviderCredentials, since?: Date): Promise<SyncResult>;
  incrementalSync(credentials: ProviderCredentials, syncToken: string): Promise<SyncResult>;

  // Operations
  fetchMessage(credentials: ProviderCredentials, messageId: string): Promise<EmailEnvelope>;
  sendMessage(credentials: ProviderCredentials, draft: OutboundEmail): Promise<{ messageId: string }>;
  markRead(credentials: ProviderCredentials, messageIds: string[]): Promise<void>;
}

export interface OAuthTokens {
  accessToken: string;
  refreshToken: string;
  expiresAt: Date;
}

export interface ProviderCredentials {
  accessToken: string;
  refreshToken: string;
  expiresAt: Date;
  // IMAP-specific
  imapHost?: string;
  imapPort?: number;
  smtpHost?: string;
  smtpPort?: number;
}

export interface OutboundEmail {
  inReplyTo?: string;
  references?: string[];
  to: EmailAddress[];
  cc?: EmailAddress[];
  bcc?: EmailAddress[];
  subject: string;
  bodyPlain: string;
  bodyHtml: string;
  attachments?: { filename: string; content: Buffer; contentType: string }[];
}
```

```typescript
// apps/api/src/integrations/gmail/gmail-provider.ts
export class GmailProvider implements EmailProvider {
  readonly providerName = 'gmail';

  // Uses googleapis npm package
  // initialSync: Gmail API users.messages.list with q="after:YYYY/MM/DD", then batch fetch
  // incrementalSync: Gmail API users.history.list with startHistoryId
  // sendMessage: Gmail API users.messages.send with RFC 2822 raw message
  // Push notifications: Gmail API users.watch with Pub/Sub topic
}

// apps/api/src/integrations/outlook/outlook-provider.ts
export class OutlookProvider implements EmailProvider {
  readonly providerName = 'outlook';

  // Uses @microsoft/microsoft-graph-client
  // initialSync: GET /me/mailFolders/inbox/messages?$orderby=receivedDateTime desc
  // incrementalSync: GET /me/mailFolders/inbox/messages/delta
  // sendMessage: POST /me/sendMail
  // Push notifications: Microsoft Graph subscriptions (webhooks)
}

// apps/api/src/integrations/imap/imap-provider.ts
export class ImapProvider implements EmailProvider {
  readonly providerName = 'imap';

  // Uses imapflow for IMAP (RFC 9051)
  // Uses nodemailer for SMTP (RFC 5321)
  // initialSync: IMAP FETCH with SINCE criteria
  // incrementalSync: IMAP IDLE + periodic SEARCH SINCE
  // sendMessage: nodemailer.sendMail
  // No push: polling via IDLE + scheduled sync
}
```

**Testing**:
- `Unit: GmailProvider.getAuthUrl — returns valid Google OAuth URL with correct scopes`
- `Unit: OutlookProvider.getAuthUrl — returns valid Azure AD OAuth URL with mail.read scope`
- `Integration (mocked API): GmailProvider.initialSync — returns EmailEnvelope[] from mock Gmail response`
- `Integration (mocked API): GmailProvider.sendMessage — calls Gmail API with correct RFC 2822 body`
- `Integration (mocked API): OutlookProvider.incrementalSync — returns delta changes correctly`
- `Integration (mocked API): ImapProvider.initialSync — IMAP FETCH returns parsed MIME messages`
- `Unit: EmailEnvelope parsing — multipart/mixed email with 2 attachments parsed correctly`
- `Unit: EmailEnvelope parsing — email with Content-ID inline image sets contentId field`
- `Fixture-based: parse 10 sample .eml files from tests/fixtures/emails/ — all produce valid EmailEnvelopes`

---

#### 2.2 — Inbox Connection and OAuth Flow

**What**: Implement the UI and API flow for connecting a Gmail, Outlook, or IMAP inbox to the organisation, storing encrypted credentials, and triggering initial sync.

**Design**:

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/inboxes` | List organisation's inboxes |
| POST | `/api/inboxes/connect/gmail` | Start Gmail OAuth flow |
| GET | `/api/inboxes/connect/gmail/callback` | Handle Gmail OAuth callback |
| POST | `/api/inboxes/connect/outlook` | Start Outlook OAuth flow |
| GET | `/api/inboxes/connect/outlook/callback` | Handle Outlook OAuth callback |
| POST | `/api/inboxes/connect/imap` | Connect via IMAP credentials |
| DELETE | `/api/inboxes/:id` | Disconnect inbox |
| POST | `/api/inboxes/:id/sync` | Trigger manual sync |
| GET | `/api/inboxes/:id/status` | Get sync status |

```typescript
// apps/api/src/lib/crypto.ts
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGORITHM = 'aes-256-gcm';

export function encryptToken(plaintext: string, masterKey: Buffer): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv(ALGORITHM, masterKey, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  return Buffer.concat([iv, tag, encrypted]).toString('base64');
}

export function decryptToken(ciphertext: string, masterKey: Buffer): string {
  const buf = Buffer.from(ciphertext, 'base64');
  const iv = buf.subarray(0, 16);
  const tag = buf.subarray(16, 32);
  const encrypted = buf.subarray(32);
  const decipher = createDecipheriv(ALGORITHM, masterKey, iv);
  decipher.setAuthTag(tag);
  return decipher.update(encrypted) + decipher.final('utf8');
}
```

**Testing**:
- `Unit: encryptToken/decryptToken roundtrip — decrypted value matches original`
- `Unit: decryptToken with wrong key — throws authentication error`
- `Integration: POST /api/inboxes/connect/gmail — returns redirect URL to Google`
- `Integration (mocked OAuth): Gmail callback — creates inbox with encrypted tokens in providerConfig`
- `Integration: GET /api/inboxes — returns inbox list with sync_state (tokens not exposed)`
- `Integration: DELETE /api/inboxes/:id — soft-deletes inbox, stops sync worker`
- `Integration: POST /api/inboxes/connect/imap — validates IMAP connection before saving`
- `Integration: POST /api/inboxes/connect/imap with bad host — returns 422 with connection error`

---

#### 2.3 — Email Sync Worker and Thread Builder

**What**: Background worker that performs initial and incremental email syncs, creates/updates conversations using RFC 5322 threading (Message-ID, In-Reply-To, References), and creates contacts from sender addresses.

**Design**:

```typescript
// apps/api/src/workers/email-sync.worker.ts
// BullMQ worker processing jobs from the 'email-sync' queue

// Job types:
// 1. initial-sync: Full sync for newly connected inbox (paginated, up to 30 days history)
// 2. incremental-sync: Delta sync using provider sync tokens (runs every 60s)
// 3. webhook-sync: Triggered by Gmail Pub/Sub or Graph webhook notification

// For each synced message:
// 1. Parse EmailEnvelope
// 2. Find or create Contact from sender address
// 3. Thread into existing Conversation using thread-builder
// 4. Create Message record with headers JSONB
// 5. Upload attachments to S3
// 6. Update Conversation denormalized fields (lastMessageAt, messageCount, preview)
// 7. Emit 'message.received' event via Redis pub/sub (for real-time UI updates)
```

```typescript
// apps/api/src/lib/thread-builder.ts

/**
 * RFC 5322 threading algorithm:
 * 1. Check In-Reply-To header — find message with matching Message-ID
 * 2. If not found, check References header (last entry first)
 * 3. If not found, subject-based matching (Re: stripped, same inbox, within 7 days)
 * 4. If no match, create new conversation
 */
export async function findOrCreateConversation(
  db: DrizzleDatabase,
  envelope: EmailEnvelope,
  inboxId: string,
  organisationId: string,
): Promise<{ conversationId: string; isNew: boolean }> {
  // Step 1: In-Reply-To lookup
  if (envelope.inReplyTo) {
    const existing = await db.select()
      .from(message)
      .where(sql`${message.headers}->>'message_id' = ${envelope.inReplyTo}`)
      .limit(1);
    if (existing.length > 0) {
      return { conversationId: existing[0].conversationId, isNew: false };
    }
  }

  // Step 2: References lookup (reverse order)
  for (const ref of [...envelope.references].reverse()) {
    const existing = await db.select()
      .from(message)
      .where(sql`${message.headers}->>'message_id' = ${ref}`)
      .limit(1);
    if (existing.length > 0) {
      return { conversationId: existing[0].conversationId, isNew: false };
    }
  }

  // Step 3: Subject-based fallback
  const normalizedSubject = envelope.subject.replace(/^(Re|Fwd|Fw):\s*/gi, '').trim();
  if (normalizedSubject) {
    const subjectMatch = await db.select()
      .from(conversation)
      .where(and(
        eq(conversation.organisationId, organisationId),
        eq(conversation.subject, normalizedSubject),
        gt(conversation.lastMessageAt, sql`now() - interval '7 days'`),
        arrayContains(conversation.inboxIds, [inboxId]),
      ))
      .orderBy(desc(conversation.lastMessageAt))
      .limit(1);
    if (subjectMatch.length > 0) {
      return { conversationId: subjectMatch[0].id, isNew: false };
    }
  }

  // Step 4: New conversation
  const [newConv] = await db.insert(conversation).values({
    organisationId,
    inboxId,
    subject: normalizedSubject || '(no subject)',
    inboxIds: [inboxId],
    channel: 'email',
  }).returning();

  return { conversationId: newConv.id, isNew: true };
}
```

**Testing**:
- `Unit: thread-builder — message with In-Reply-To matching existing message joins that conversation`
- `Unit: thread-builder — message with References[2] matching existing message joins that conversation`
- `Unit: thread-builder — message with subject "Re: Order #1234" matches conversation "Order #1234"`
- `Unit: thread-builder — message with no matching headers or subject creates new conversation`
- `Unit: thread-builder — subject match older than 7 days is ignored (new conversation created)`
- `Unit: subject normalization — "Re: Re: Fwd: Hello" normalizes to "Hello"`
- `Integration: email-sync worker processes initial-sync job — creates conversations and messages in DB`
- `Integration: email-sync worker processes incremental-sync — new messages added to existing conversations`
- `Integration: email-sync worker — creates Contact record from new sender address`
- `Integration: email-sync worker — existing Contact found by email handle, not duplicated`
- `Fixture-based: sync 20 sample .eml files — correct threading (3 conversations with 5, 8, 7 messages)`

---

#### 2.4 — Outbound Email Sending

**What**: Send replies and new emails through connected inbox accounts, with attachment support, proper threading headers, and send tracking.

**Design**:

```typescript
// apps/api/src/routes/messages/send.ts

export const sendMessageSchema = z.object({
  conversationId: z.string().uuid(),
  bodyHtml: z.string().min(1),
  bodyPlain: z.string().optional(),
  to: z.array(z.object({ name: z.string().optional(), email: z.string().email() })),
  cc: z.array(z.object({ name: z.string().optional(), email: z.string().email() })).optional(),
  bcc: z.array(z.object({ name: z.string().optional(), email: z.string().email() })).optional(),
  attachmentIds: z.array(z.string().uuid()).optional(), // Pre-uploaded attachment IDs
});

// POST /api/conversations/:id/reply
// 1. Validate user has conversation.write permission
// 2. Load conversation and inbox
// 3. Build OutboundEmail with correct In-Reply-To and References from conversation history
// 4. Enqueue to 'email-send' BullMQ queue
// 5. Create Message record with direction='outbound', isDraft=false
// 6. Update Conversation denormalized fields
// 7. Return message object

// The email-send worker:
// 1. Decrypt provider credentials
// 2. Call provider.sendMessage()
// 3. Update message.sentAt
// 4. Log to audit_log
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/conversations/:id/reply` | Send reply to conversation |
| POST | `/api/conversations/:id/note` | Add internal note (not sent) |
| POST | `/api/messages/draft` | Save draft |
| PATCH | `/api/messages/:id/draft` | Update draft |
| DELETE | `/api/messages/:id/draft` | Discard draft |
| POST | `/api/attachments/upload` | Upload attachment (returns presigned URL) |

**Testing**:
- `Unit: sendMessageSchema — valid reply payload passes`
- `Unit: sendMessageSchema — empty bodyHtml rejected`
- `Integration: POST /api/conversations/:id/reply — creates outbound message, enqueues send job`
- `Integration: email-send worker — calls provider.sendMessage with correct In-Reply-To header`
- `Integration: POST /api/conversations/:id/note — creates message with messageType='note', not sent externally`
- `Integration: POST /api/messages/draft — creates message with isDraft=true`
- `Integration: PATCH /api/messages/:id/draft — updates draft body`
- `Integration: DELETE /api/messages/:id/draft — removes draft message`
- `Integration (mocked SMTP): send reply — nodemailer called with correct MIME structure and attachments`
- `E2E: connect inbox, receive test email, reply — original sender receives reply with correct threading`

---

## Phase 3: Core Inbox UI — Conversation List, Detail, and Composition

### Purpose

Build the web frontend for the shared inbox experience: conversation list with filtering/sorting, conversation detail view with message thread, reply composition, and internal notes. After this phase, agents can log in, see their assigned conversations, read full email threads, and send replies through the browser.

### Tasks

#### 3.1 — Inbox Layout and Conversation List

**What**: Three-panel inbox layout (sidebar, conversation list, conversation detail) with filtering by status, assignee, tag, and inbox; sorting by recency and priority.

**Design**:

```typescript
// apps/web/src/components/inbox/InboxLayout.tsx
// Three-panel responsive layout:
// - Left sidebar (240px): inbox list, quick filters (Open, Assigned to Me, Unassigned, Snoozed, Closed)
// - Center panel (350px): conversation list with infinite scroll
// - Right panel (remaining): conversation detail

// apps/web/src/lib/api-client.ts
export interface ConversationListParams {
  inboxId?: string;
  status?: 'open' | 'snoozed' | 'closed' | 'archived';
  assigneeId?: string | 'unassigned';
  tagIds?: string[];
  priority?: string;
  search?: string;
  sortBy?: 'last_message_at' | 'created_at' | 'priority';
  sortDir?: 'asc' | 'desc';
  cursor?: string;     // cursor-based pagination
  limit?: number;      // default 25
}

export interface ConversationListItem {
  id: string;
  subject: string;
  status: string;
  priority: string;
  assignee: { id: string; displayName: string; avatarUrl: string | null } | null;
  lastMessageAt: string;
  lastMessagePreview: string;
  messageCount: number;
  participants: Array<{ type: string; name: string; email: string }>;
  tagIds: string[];
  sla: { status: string; nextDue: string | null } | null;
  isRead: boolean;
}
```

Backend endpoint:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/conversations` | Paginated conversation list with filters |

```typescript
// GET /api/conversations response
{
  conversations: ConversationListItem[];
  nextCursor: string | null;
  totalCount: number;
}
```

**Testing**:
- `Unit: ConversationList component — renders conversation items with subject, preview, assignee avatar`
- `Unit: ConversationList — clicking a conversation calls onSelect with conversation ID`
- `Unit: ConversationList — empty state shows "No conversations" message`
- `Integration: GET /api/conversations?status=open — returns only open conversations for the org`
- `Integration: GET /api/conversations?assigneeId=unassigned — returns unassigned conversations`
- `Integration: GET /api/conversations?tagIds=uuid1,uuid2 — returns conversations with any matching tag`
- `Integration: cursor pagination — page 1 returns 25 items + nextCursor, page 2 returns next batch`
- `E2E: login, navigate to inbox — conversation list loads with real data`
- `E2E: click filter "Assigned to me" — list updates to show only agent's conversations`

---

#### 3.2 — Conversation Detail and Message Thread

**What**: Full conversation view showing the complete message thread (email replies, internal notes, system events), participant list, conversation metadata, and action buttons (assign, tag, close, snooze).

**Design**:

```typescript
// apps/web/src/components/conversation/ConversationDetail.tsx

export interface ConversationDetailData {
  id: string;
  subject: string;
  status: string;
  priority: string;
  channel: string;
  assignee: { id: string; displayName: string; avatarUrl: string | null } | null;
  team: { id: string; name: string } | null;
  participants: ConversationParticipant[];
  tags: { id: string; name: string; colour: string }[];
  sla: SlaState | null;
  customFields: Record<string, unknown>;
  ai: ConversationAiState;
  createdAt: string;
  closedAt: string | null;
}

export interface MessageData {
  id: string;
  authorType: 'agent' | 'contact' | 'system' | 'ai';
  author: { id: string; name: string; email: string; avatarUrl: string | null };
  messageType: 'reply' | 'note' | 'system' | 'draft';
  direction: 'inbound' | 'outbound';
  bodyHtml: string;
  bodyPlain: string;
  attachments: AttachmentData[];
  sentAt: string | null;
  receivedAt: string | null;
  createdAt: string;
  ai: { sentiment?: string; sentimentScore?: number };
}

// Messages are rendered in chronological order
// Internal notes have a distinct yellow background
// System messages (assignment changes, status changes) are compact timeline entries
// Collapsed quoted text detection: hide "On <date> <sender> wrote:" blocks by default
```

Backend endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/conversations/:id` | Full conversation detail |
| GET | `/api/conversations/:id/messages` | Paginated message list |
| PATCH | `/api/conversations/:id` | Update status, priority, assignee, tags |
| POST | `/api/conversations/:id/snooze` | Snooze until timestamp |
| POST | `/api/conversations/:id/assign` | Assign to user or team |

**Testing**:
- `Unit: MessageThread component — renders email messages with HTML body safely (XSS sanitized)`
- `Unit: MessageThread — internal notes rendered with yellow background`
- `Unit: MessageThread — system messages rendered as compact timeline`
- `Unit: MessageThread — quoted text collapsed by default, expandable on click`
- `Unit: ConversationActions — assign button opens user picker`
- `Integration: GET /api/conversations/:id — returns full conversation with participants and SLA`
- `Integration: PATCH /api/conversations/:id {status: "closed"} — sets closedAt, emits event`
- `Integration: POST /api/conversations/:id/assign {userId} — updates assigneeId, creates audit_log entry`
- `E2E: open conversation — message thread loads with all messages in order`
- `E2E: assign conversation to self — assignee updates in real time`
- `E2E: close conversation — status changes, conversation moves to closed filter`

---

#### 3.3 — Reply Composer and Internal Notes

**What**: Rich text reply composer with formatting toolbar, canned response insertion, attachment upload, CC/BCC fields, and a toggle for internal notes vs. external replies.

**Design**:

```typescript
// apps/web/src/components/conversation/ReplyComposer.tsx

// Uses TipTap editor for rich text (HTML output)
// Toolbar: Bold, Italic, Underline, Link, Bullet List, Ordered List, Code Block, Image
// Bottom bar: Send button, Note toggle, CC/BCC toggle, Attachment button, Canned Responses dropdown

export interface ReplyComposerProps {
  conversationId: string;
  defaultTo: EmailAddress[];
  onSent: (message: MessageData) => void;
  onDraftSaved: (message: MessageData) => void;
}

// State machine for composer:
// idle → composing → sending → sent
// idle → composing → saving_draft → draft_saved → composing
// composing → discarding → idle

// Keyboard shortcuts:
// Cmd+Enter: Send
// Cmd+Shift+Enter: Send as Note
// Cmd+S: Save Draft
// Escape: Discard (with confirmation if content exists)
```

**Testing**:
- `Unit: ReplyComposer — renders with default To addresses from conversation`
- `Unit: ReplyComposer — toggling Note mode changes send button text and removes To/CC fields`
- `Unit: ReplyComposer — Cmd+Enter triggers send`
- `Unit: ReplyComposer — Cmd+S saves draft`
- `Unit: ReplyComposer — canned response insertion replaces variables ({customer_name}) with contact data`
- `Integration: type reply and send — POST /api/conversations/:id/reply called, message appears in thread`
- `Integration: upload attachment — presigned URL obtained, file uploaded, attachment shown in composer`
- `E2E: compose and send reply — reply appears in thread, sent via email provider`
- `E2E: compose and send note — note appears in thread with yellow background, not sent externally`

---

#### 3.4 — Collision Detection and Real-Time Updates

**What**: Show which agents are currently viewing or replying to a conversation (presence indicators), prevent duplicate replies via real-time collision detection, and push new messages to open conversations.

**Design**:

```typescript
// apps/api/src/plugins/websocket.ts
// Socket.IO rooms:
// - org:{orgId} — org-wide events (new conversation, assignment changes)
// - inbox:{inboxId} — inbox-level events (conversation list updates)
// - conversation:{convId} — conversation-level events (new messages, typing, presence)

// Events emitted by server:
interface ServerEvents {
  'conversation:new': (data: ConversationListItem) => void;
  'conversation:updated': (data: Partial<ConversationDetailData> & { id: string }) => void;
  'message:new': (data: MessageData) => void;
  'presence:update': (data: { conversationId: string; users: PresenceUser[] }) => void;
  'typing:update': (data: { conversationId: string; userId: string; isTyping: boolean }) => void;
}

// Events emitted by client:
interface ClientEvents {
  'conversation:join': (conversationId: string) => void;
  'conversation:leave': (conversationId: string) => void;
  'typing:start': (conversationId: string) => void;
  'typing:stop': (conversationId: string) => void;
}

interface PresenceUser {
  id: string;
  displayName: string;
  avatarUrl: string | null;
  activity: 'viewing' | 'replying';
  since: string;
}

// Collision detection:
// When a user starts composing, they emit typing:start
// Server broadcasts presence:update to all users in that conversation room
// UI shows "Agent X is replying..." banner
// If another agent sends a reply, the composing agent sees a warning: "Agent X just replied"
```

**Testing**:
- `Unit: PresenceIndicator component — shows avatar stack of viewing agents`
- `Unit: PresenceIndicator — shows "Agent X is replying..." when another agent types`
- `Integration: two Socket.IO clients join same conversation room — both receive presence updates`
- `Integration: client A emits typing:start — client B receives typing:update with isTyping=true`
- `Integration: client A disconnects — client B receives presence:update without client A`
- `Integration: new message synced by worker — emitted to conversation room via Socket.IO`
- `E2E: two browser sessions on same conversation — both see each other's presence`
- `E2E: agent A starts typing — agent B sees "Agent A is replying..." banner`

---

## Phase 4: SLA Tracking and Business Hours

### Purpose

Implement the SLA engine: policy definition, automatic policy matching to conversations, business-hours-aware deadline calculation, breach detection, escalation alerts, and SLA dashboard. After this phase, admins can configure SLA policies (e.g., "urgent emails get 1-hour first response during business hours"), and the system automatically tracks and alerts on breaches.

### Tasks

#### 4.1 — SLA Policy Management

**What**: CRUD endpoints for SLA policies with ITIL v4 tier support (incident, service request, change), condition-based matching, and business hours configuration.

**Design**:

```typescript
// packages/shared/src/types/sla.ts

export interface SlaPolicy {
  id: string;
  organisationId: string;
  name: string;
  description?: string;
  priority: 'urgent' | 'high' | 'normal' | 'low';
  targets: SlaTargets;
  conditions: SlaCondition[];
  sortOrder: number;
  isActive: boolean;
}

export interface SlaTargets {
  firstResponse: { seconds: number; businessHours: boolean };
  nextResponse?: { seconds: number; businessHours: boolean };
  resolution?: { seconds: number; businessHours: boolean };
}

export interface SlaCondition {
  field: 'priority' | 'inbox_id' | 'channel' | 'tag_id' | 'custom_field';
  op: 'eq' | 'neq' | 'in' | 'not_in' | 'contains';
  value: string | string[];
  customFieldKey?: string;  // When field = 'custom_field'
}

// Business hours stored in organisation.settings.businessHours:
// {
//   "mon": { "start": "09:00", "end": "17:00" },
//   "tue": { "start": "09:00", "end": "17:00" },
//   ...
//   "sat": null,  // closed
//   "sun": null   // closed
// }
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/sla-policies` | List SLA policies |
| POST | `/api/sla-policies` | Create SLA policy |
| PATCH | `/api/sla-policies/:id` | Update SLA policy |
| DELETE | `/api/sla-policies/:id` | Delete SLA policy |
| PUT | `/api/sla-policies/reorder` | Reorder policies (evaluation priority) |
| PATCH | `/api/settings/business-hours` | Update business hours |

**Testing**:
- `Unit: SLA policy validation — targets.firstResponse.seconds must be > 0`
- `Unit: SLA policy validation — conditions with unknown field rejected`
- `Integration: POST /api/sla-policies — creates policy, returns 201`
- `Integration: PUT /api/sla-policies/reorder — updates sortOrder for all policies`
- `Integration: DELETE /api/sla-policies/:id — soft-deletes, existing SLA instances unaffected`

---

#### 4.2 — SLA Calculator with Business Hours

**What**: Core calculation engine that computes SLA deadlines accounting for business hours, holidays, and timezone.

**Design**:

```typescript
// apps/api/src/lib/sla-calculator.ts

export interface BusinessHoursConfig {
  timezone: string;                    // IANA timezone
  schedule: Record<string, { start: string; end: string } | null>;
  holidays?: string[];                 // ISO date strings
}

/**
 * Calculate the deadline timestamp given a start time and SLA target.
 * If businessHours=true, only counts time during business hours.
 */
export function calculateDeadline(
  startAt: Date,
  targetSeconds: number,
  businessHours: boolean,
  config: BusinessHoursConfig,
): Date {
  if (!businessHours) {
    return new Date(startAt.getTime() + targetSeconds * 1000);
  }
  // Walk through business hours windows from startAt, accumulating seconds
  // Skip weekends and holidays
  // Handle timezone conversions for DST boundaries
  // Return the exact timestamp when targetSeconds of business time have elapsed
}

/**
 * Calculate elapsed business-hours seconds between two timestamps.
 */
export function calculateElapsedBusinessSeconds(
  from: Date,
  to: Date,
  config: BusinessHoursConfig,
): number { /* ... */ }

/**
 * Determine which SLA policy applies to a conversation.
 * Evaluates policies in sortOrder, returns first match.
 */
export function matchSlaPolicy(
  conversation: { priority: string; inboxId: string; channel: string; tagIds: string[]; customFields: Record<string, unknown> },
  policies: SlaPolicy[],
): SlaPolicy | null {
  for (const policy of policies.sort((a, b) => a.sortOrder - b.sortOrder)) {
    if (!policy.isActive) continue;
    if (policy.conditions.every(c => evaluateCondition(c, conversation))) {
      return policy;
    }
  }
  return null;
}
```

**Testing**:
- `Unit: calculateDeadline — 3600s non-business-hours = startAt + 1 hour exactly`
- `Unit: calculateDeadline — 3600s business-hours starting at 16:30 (closes 17:00) = next day 09:30`
- `Unit: calculateDeadline — spans weekend (Friday 16:00, 7200s target) = Monday 10:00`
- `Unit: calculateDeadline — holiday on Monday, spans weekend = Tuesday 09:00`
- `Unit: calculateDeadline — DST transition handled correctly (spring forward)`
- `Unit: calculateElapsedBusinessSeconds — 09:00 to 17:00 same day = 28800 seconds`
- `Unit: calculateElapsedBusinessSeconds — Friday 16:00 to Monday 10:00 = 3600 seconds`
- `Unit: matchSlaPolicy — conversation with priority=urgent matches urgent policy first`
- `Unit: matchSlaPolicy — conversation with inbox_id matching condition = matched`
- `Unit: matchSlaPolicy — no conditions match any policy = returns null`

---

#### 4.3 — SLA Tracking Worker and Breach Detection

**What**: Background worker that attaches SLA policies to new conversations, calculates deadlines, checks for approaching breaches, and triggers escalation notifications.

**Design**:

```typescript
// apps/api/src/workers/sla-check.worker.ts

// Runs every 30 seconds via BullMQ repeatable job
// For each organisation:
// 1. Load active SLA policies
// 2. Find conversations with sla.status = 'active' and sla.firstResponseDue < now() + 15min
// 3. For approaching breaches (< 15 min): send warning notification
// 4. For actual breaches (past due): update sla.status = 'breached', create notification + audit log
// 5. Find new conversations without SLA: match policy, calculate deadline, set sla JSONB

// SLA state machine (stored in conversation.sla JSONB):
// 'pending' → 'active' (policy matched, deadline set)
// 'active' → 'at_risk' (< 15 min to deadline)
// 'active' | 'at_risk' → 'breached' (past deadline)
// 'active' | 'at_risk' → 'met' (first response sent before deadline)
// 'active' → 'paused' (conversation snoozed or waiting on customer)
// 'paused' → 'active' (customer replies, snooze ends)

export interface SlaState {
  policyId: string;
  policyName: string;
  status: 'pending' | 'active' | 'at_risk' | 'breached' | 'met' | 'paused';
  firstResponseDue: string | null;
  firstResponseAt: string | null;
  resolutionDue: string | null;
  resolvedAt: string | null;
  breached: boolean;
  pausedAt: string | null;
  totalPauseSeconds: number;
}
```

**Testing**:
- `Unit: SLA worker — new conversation matched to policy, sla JSONB set with correct deadline`
- `Unit: SLA worker — conversation with first response sent before deadline, sla.status = 'met'`
- `Unit: SLA worker — conversation past deadline without response, sla.status = 'breached'`
- `Unit: SLA worker — conversation approaching deadline, sla.status = 'at_risk'`
- `Unit: SLA worker — snoozed conversation has SLA paused`
- `Integration: SLA worker creates notification for breach — notification record exists in DB`
- `Integration: SLA worker creates audit_log entry for breach`
- `Integration: SLA worker with business hours — deadline only counts business time`
- `E2E: create SLA policy, receive email, wait past deadline — breach notification appears in UI`

---

#### 4.4 — SLA Dashboard UI

**What**: SLA overview panel showing at-risk and breached conversations, SLA compliance metrics, and policy configuration interface.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/analytics/sla/page.tsx

// Dashboard widgets:
// 1. SLA Compliance Rate: percentage of conversations meeting SLA (pie chart)
// 2. Active Breaches: count + list of currently breached conversations (sorted by breach duration)
// 3. At-Risk Conversations: approaching deadline within 15 minutes
// 4. Average First Response Time: mean and p95 across time period
// 5. SLA by Priority: compliance rate per priority level (bar chart)
// 6. Trend: daily SLA compliance over last 30 days (line chart)

export interface SlaAnalyticsResponse {
  complianceRate: number;         // 0.0 to 1.0
  activeBreaches: number;
  atRiskCount: number;
  avgFirstResponseSeconds: number;
  p95FirstResponseSeconds: number;
  byPriority: Array<{ priority: string; compliance: number; count: number }>;
  dailyTrend: Array<{ date: string; compliance: number; total: number }>;
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/analytics/sla` | SLA analytics for date range |
| GET | `/api/analytics/sla/breaches` | Currently breached conversations |

**Testing**:
- `Unit: SLA compliance chart — renders correct percentage`
- `Unit: breach list — shows conversations sorted by breach duration (longest first)`
- `Integration: GET /api/analytics/sla — returns correct compliance rate for test data`
- `Integration: GET /api/analytics/sla/breaches — returns only breached conversations`
- `E2E: navigate to SLA dashboard — all widgets render with data`

---

## Phase 5: Contact Management and CRM

### Purpose

Build the contact management system: automatic contact creation from emails, contact profiles with interaction history, contact merging, handle management (multiple email addresses per contact), and CRM sync with Salesforce and HubSpot. After this phase, agents see rich customer context alongside every conversation.

### Tasks

#### 5.1 — Contact CRUD and Handle Lookup

**What**: Contact management endpoints with JSONB handle lookup, contact detail page showing all conversations, and automatic contact creation from inbound emails.

**Design**:

```typescript
// apps/api/src/services/contact.service.ts

export class ContactService {
  /**
   * Find or create a contact by email handle.
   * Uses GIN-indexed JSONB containment query.
   */
  async findOrCreateByEmail(
    orgId: string,
    email: string,
    name?: string,
  ): Promise<Contact> {
    // Query: SELECT * FROM contact WHERE org_id = $1 AND handles @> '[{"type":"email","value":"$2"}]'
    const existing = await this.db.select().from(contact)
      .where(and(
        eq(contact.organisationId, orgId),
        sql`${contact.handles} @> ${JSON.stringify([{ type: 'email', value: email }])}::jsonb`,
      ))
      .limit(1);

    if (existing.length > 0) return existing[0];

    // Create new contact
    const [newContact] = await this.db.insert(contact).values({
      organisationId: orgId,
      displayName: name || email.split('@')[0],
      primaryEmail: email,
      handles: [{ type: 'email', value: email, isPrimary: true }],
    }).returning();

    return newContact;
  }

  /**
   * Merge two contacts: move all conversations from source to target,
   * combine handles, update conversation participants.
   */
  async mergeContacts(orgId: string, targetId: string, sourceId: string): Promise<Contact> { /* ... */ }
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/contacts` | List contacts (paginated, searchable) |
| GET | `/api/contacts/:id` | Contact detail with conversation history |
| POST | `/api/contacts` | Create contact manually |
| PATCH | `/api/contacts/:id` | Update contact fields |
| POST | `/api/contacts/:id/merge` | Merge another contact into this one |
| DELETE | `/api/contacts/:id/anonymise` | GDPR erasure (anonymise PII) |

**Testing**:
- `Unit: findOrCreateByEmail — existing contact found by email handle in JSONB`
- `Unit: findOrCreateByEmail — new contact created when no match`
- `Unit: mergeContacts — source handles merged into target, source conversations updated`
- `Integration: GET /api/contacts?search=jane — returns contacts matching name or email`
- `Integration: POST /api/contacts/:id/merge — conversations re-linked, source marked merged`
- `Integration: DELETE /api/contacts/:id/anonymise — PII fields cleared, isAnonymised=true`
- `Integration: anonymised contact — conversation history preserved but name shows "Anonymised Contact"`

---

#### 5.2 — Contact Sidebar in Conversation View

**What**: Sidebar component in the conversation detail showing contact profile, conversation history, company info, and custom fields.

**Design**:

```typescript
// apps/web/src/components/contact/ContactSidebar.tsx

// Displays:
// 1. Contact avatar, name, primary email
// 2. All handles (email, phone, social)
// 3. Company info (name, domain, industry)
// 4. Conversation count and last contacted date
// 5. Previous conversations list (last 10)
// 6. Custom fields defined by organisation
// 7. CRM links (Salesforce/HubSpot record, if synced)
// 8. Edit button (opens inline editor)

export interface ContactSidebarProps {
  contactId: string;
  onContactUpdated: (contact: ContactData) => void;
}
```

**Testing**:
- `Unit: ContactSidebar — renders contact name, email, and avatar`
- `Unit: ContactSidebar — shows previous conversations with subject and date`
- `Unit: ContactSidebar — shows custom fields from organisation settings`
- `E2E: open conversation — contact sidebar loads with correct contact data`
- `E2E: edit contact name in sidebar — name updates immediately`

---

#### 5.3 — CRM Integration (Salesforce and HubSpot)

**What**: Bidirectional sync between contacts and Salesforce Contacts/Leads and HubSpot Contacts, with periodic background sync.

**Design**:

```typescript
// apps/api/src/integrations/crm/crm-provider.ts

export interface CrmProvider {
  readonly name: 'salesforce' | 'hubspot';
  connect(credentials: OAuthTokens): Promise<void>;
  findContact(email: string): Promise<CrmContact | null>;
  createContact(data: CrmContactCreate): Promise<CrmContact>;
  updateContact(externalId: string, data: Partial<CrmContactCreate>): Promise<CrmContact>;
  listRecentChanges(since: Date): Promise<CrmContact[]>;
}

export interface CrmContact {
  externalId: string;
  email: string;
  name: string;
  company?: string;
  title?: string;
  phone?: string;
  customFields: Record<string, unknown>;
  lastModified: Date;
}

// Sync strategy:
// 1. When a new contact is created locally, check CRM for matching email
// 2. If found, link and pull CRM data into contact.crm JSONB
// 3. Background job runs every 5 minutes: pull recent CRM changes, update local contacts
// 4. On local contact update, optionally push changes to CRM (configurable)
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/integrations/crm/salesforce/connect` | Start Salesforce OAuth |
| POST | `/api/integrations/crm/hubspot/connect` | Start HubSpot OAuth |
| POST | `/api/integrations/crm/sync` | Trigger manual CRM sync |
| GET | `/api/integrations/crm/status` | Sync status and last sync time |

**Testing**:
- `Integration (mocked API): Salesforce findContact — returns CrmContact from mock response`
- `Integration (mocked API): HubSpot createContact — calls HubSpot API with correct payload`
- `Integration: CRM sync worker — new CRM contacts matched to local contacts by email`
- `Integration: CRM sync worker — CRM field updates reflected in contact.crm JSONB`
- `Integration: local contact update with CRM push enabled — calls CRM updateContact`

---

## Phase 6: Tags, Automation Rules, and Canned Responses

### Purpose

Implement the workflow automation layer: tag management, rule-based automation (auto-assign, auto-tag, auto-close), and canned response templates. After this phase, admins can configure rules like "emails from vip@bigclient.com are auto-tagged VIP, assigned to senior team, and prioritised high."

### Tasks

#### 6.1 — Tag Management

**What**: CRUD for tags with hierarchical support, colour coding, and conversation tagging/untagging.

**Design**:

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tags` | List tags for organisation |
| POST | `/api/tags` | Create tag |
| PATCH | `/api/tags/:id` | Update tag (name, colour, parent) |
| DELETE | `/api/tags/:id` | Delete tag (remove from all conversations) |
| POST | `/api/conversations/:id/tags` | Add tags to conversation |
| DELETE | `/api/conversations/:id/tags/:tagId` | Remove tag from conversation |

```typescript
// Tag array operations use PostgreSQL array operators:
// Add tag: UPDATE conversation SET tag_ids = array_append(tag_ids, $tagId) WHERE id = $convId
// Remove tag: UPDATE conversation SET tag_ids = array_remove(tag_ids, $tagId) WHERE id = $convId
// Filter by tag: SELECT * FROM conversation WHERE tag_ids @> ARRAY[$tagId]::uuid[]
```

**Testing**:
- `Unit: tag validation — duplicate tag name in same org rejected`
- `Integration: POST /api/conversations/:id/tags — tagIds array updated`
- `Integration: DELETE /api/tags/:id — tag removed from all conversations' tagIds arrays`
- `Integration: GET /api/conversations?tagIds=uuid — GIN index used (EXPLAIN confirms)`
- `E2E: add tag to conversation — tag appears in conversation list item and detail view`

---

#### 6.2 — Automation Rule Engine

**What**: Rule definition, evaluation, and execution engine. Rules consist of a trigger event, conditions, and actions. Evaluated on every relevant event.

**Design**:

```typescript
// apps/api/src/services/automation.service.ts

export interface AutomationRuleDefinition {
  trigger: 'conversation.created' | 'message.received' | 'conversation.assigned'
         | 'sla.breaching' | 'conversation.closed' | 'tag.added';
  conditions: RuleCondition[];
  actions: RuleAction[];
}

export interface RuleCondition {
  field: string;     // e.g. 'message.from_address', 'conversation.subject', 'conversation.priority'
  op: 'eq' | 'neq' | 'contains' | 'not_contains' | 'matches' | 'in' | 'not_in';
  value: string | string[];
}

export interface RuleAction {
  type: 'set_priority' | 'assign_user' | 'assign_team' | 'add_tag' | 'remove_tag'
      | 'set_status' | 'send_reply' | 'send_notification' | 'set_custom_field';
  params: Record<string, unknown>;
}

export class AutomationEngine {
  /**
   * Evaluate all active rules for a given trigger event.
   * Executes actions for the first matching rule (short-circuit).
   */
  async evaluate(
    orgId: string,
    trigger: string,
    context: {
      conversation: ConversationDetailData;
      message?: MessageData;
    },
  ): Promise<{ ruleId: string; actionsExecuted: string[] } | null> {
    const rules = await this.db.select().from(automationRule)
      .where(and(
        eq(automationRule.organisationId, orgId),
        eq(automationRule.isActive, true),
        sql`${automationRule.rule}->>'trigger' = ${trigger}`,
      ))
      .orderBy(automationRule.createdAt);

    for (const rule of rules) {
      const def = rule.rule as AutomationRuleDefinition;
      if (this.matchConditions(def.conditions, context)) {
        const executed = await this.executeActions(def.actions, context);
        await this.db.update(automationRule)
          .set({ runCount: sql`run_count + 1`, lastRunAt: new Date() })
          .where(eq(automationRule.id, rule.id));
        return { ruleId: rule.id, actionsExecuted: executed };
      }
    }
    return null;
  }
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/automation-rules` | List rules |
| POST | `/api/automation-rules` | Create rule |
| PATCH | `/api/automation-rules/:id` | Update rule |
| DELETE | `/api/automation-rules/:id` | Delete rule |
| POST | `/api/automation-rules/:id/test` | Dry-run rule against a conversation |

**Testing**:
- `Unit: matchConditions — subject contains "urgent" matches rule with contains operator`
- `Unit: matchConditions — from_address in ["vip@client.com"] matches`
- `Unit: matchConditions — AND logic: all conditions must match`
- `Unit: executeActions — set_priority action updates conversation.priority`
- `Unit: executeActions — add_tag action appends to conversation.tagIds`
- `Unit: executeActions — assign_user action updates conversation.assigneeId`
- `Unit: executeActions — send_reply action enqueues outbound email`
- `Integration: message.received triggers rule — conversation auto-tagged and assigned`
- `Integration: rule dry-run — returns what actions would execute without executing them`
- `Integration: rule with send_reply action — canned response sent to customer`
- `E2E: create rule "tag VIP when from *@bigclient.com", receive email — tag applied automatically`

---

#### 6.3 — Canned Responses

**What**: Template library with variable substitution for quick replies.

**Design**:

```typescript
// Variable substitution engine
// Templates use {{variable}} syntax
// Available variables:
// {{customer_name}}, {{customer_email}}, {{agent_name}}, {{conversation_subject}},
// {{organisation_name}}, {{ticket_id}}, plus any custom field keys

export function renderTemplate(
  template: string,
  variables: Record<string, string>,
): string {
  return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
    return variables[key] ?? match; // Leave unmatched variables as-is
  });
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/canned-responses` | List templates |
| POST | `/api/canned-responses` | Create template |
| PATCH | `/api/canned-responses/:id` | Update template |
| DELETE | `/api/canned-responses/:id` | Delete template |

**Testing**:
- `Unit: renderTemplate — {{customer_name}} replaced with "Jane Doe"`
- `Unit: renderTemplate — unknown {{unknown_var}} left as-is`
- `Unit: renderTemplate — multiple variables in same template all replaced`
- `Integration: GET /api/canned-responses — returns templates grouped by category`
- `E2E: select canned response in composer — template inserted with variables filled`

---

## Phase 7: Analytics and Reporting

### Purpose

Build the analytics engine: team performance dashboards, response time metrics, conversation volume trends, agent workload distribution, and exportable reports. After this phase, managers can monitor their team's email handling performance and identify bottlenecks.

### Tasks

#### 7.1 — Analytics Data Aggregation

**What**: Analytics query service that computes key metrics from conversation and message data, with date range filtering and grouping.

**Design**:

```typescript
// apps/api/src/services/analytics.service.ts

export interface AnalyticsParams {
  organisationId: string;
  dateFrom: Date;
  dateTo: Date;
  inboxId?: string;
  assigneeId?: string;
  groupBy?: 'day' | 'week' | 'month';
}

export interface OverviewMetrics {
  totalConversations: number;
  newConversations: number;
  closedConversations: number;
  avgFirstResponseSeconds: number;
  medianFirstResponseSeconds: number;
  p95FirstResponseSeconds: number;
  avgResolutionSeconds: number;
  slaComplianceRate: number;
  customerSatisfaction: number | null;
}

export interface AgentMetrics {
  userId: string;
  displayName: string;
  assigned: number;
  replied: number;
  closed: number;
  avgResponseSeconds: number;
  slaCompliance: number;
}

export interface VolumeDataPoint {
  date: string;
  inbound: number;
  outbound: number;
  newConversations: number;
  closedConversations: number;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/analytics/overview` | Key metrics for date range |
| GET | `/api/analytics/agents` | Per-agent metrics |
| GET | `/api/analytics/volume` | Conversation volume over time |
| GET | `/api/analytics/tags` | Conversation distribution by tag |
| GET | `/api/analytics/response-times` | Response time distribution (histogram) |
| GET | `/api/analytics/export` | CSV export of analytics data |

**Testing**:
- `Integration: GET /api/analytics/overview — returns correct totalConversations for date range`
- `Integration: GET /api/analytics/overview — avgFirstResponseSeconds computed correctly from message timestamps`
- `Integration: GET /api/analytics/agents — returns per-agent reply counts`
- `Integration: GET /api/analytics/volume?groupBy=day — returns daily data points`
- `Integration: GET /api/analytics/export — returns valid CSV with headers`
- `Unit: analytics query — filters by inboxId correctly`
- `Unit: analytics query — excludes snoozed time from response calculations`

---

#### 7.2 — Analytics Dashboard UI

**What**: Dashboard page with interactive charts for overview metrics, agent performance, volume trends, and tag distribution.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/analytics/page.tsx

// Uses Recharts for charting
// Components:
// - MetricCard: single number with trend arrow (up/down vs previous period)
// - ResponseTimeChart: line chart showing avg/p95 first response over time
// - VolumeChart: stacked bar chart (inbound vs outbound)
// - AgentLeaderboard: table sorted by response time or volume
// - TagDistribution: horizontal bar chart of conversation counts by tag
// - DateRangePicker: preset ranges (today, 7d, 30d, 90d) + custom
// - InboxFilter: dropdown to filter by specific inbox
```

**Testing**:
- `Unit: MetricCard — renders value and correct trend arrow direction`
- `Unit: VolumeChart — renders stacked bars with correct data`
- `Unit: DateRangePicker — selecting "Last 7 days" sets correct from/to`
- `E2E: navigate to analytics — dashboard loads with charts populated`
- `E2E: change date range — all charts update with new data`

---

## Phase 8: Knowledge Base

### Purpose

Build the customer-facing knowledge base: article management, public help center with search, and article suggestion in the conversation view. After this phase, support teams can create FAQ articles that customers can search independently, and agents can link articles to conversations.

### Tasks

#### 8.1 — Article Management

**What**: CRUD for knowledge base articles with collections, full-text search, and publish/draft workflow.

**Design**:

```typescript
// apps/api/src/routes/knowledge-base/articles.ts

export const articleCreateSchema = z.object({
  title: z.string().min(1).max(200),
  collection: z.string().default('general'),
  bodyHtml: z.string().min(1),
  status: z.enum(['draft', 'published']).default('draft'),
  meta: z.object({
    tags: z.array(z.string()).optional(),
    seoDescription: z.string().optional(),
  }).optional(),
});

// Full-text search using PostgreSQL tsvector:
// On INSERT/UPDATE, set search_vector = to_tsvector('english', title || ' ' || body_plain)
// Search query: SELECT * FROM kb_article WHERE search_vector @@ plainto_tsquery('english', $query)
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/knowledge-base/articles` | List articles (admin, with drafts) |
| POST | `/api/knowledge-base/articles` | Create article |
| GET | `/api/knowledge-base/articles/:id` | Get article |
| PATCH | `/api/knowledge-base/articles/:id` | Update article |
| DELETE | `/api/knowledge-base/articles/:id` | Delete article |
| POST | `/api/knowledge-base/articles/:id/publish` | Publish draft |
| GET | `/api/help/articles` | Public search (published only) |
| GET | `/api/help/articles/:slug` | Public article by slug |

**Testing**:
- `Unit: article slug generation — "How to Reset Password" becomes "how-to-reset-password"`
- `Unit: article slug collision — appends "-2" when slug exists`
- `Integration: POST /api/knowledge-base/articles — creates article with tsvector populated`
- `Integration: GET /api/help/articles?q=password — returns published articles matching query`
- `Integration: GET /api/help/articles?q=password — draft articles not returned`
- `Integration: article publish — sets publishedAt and status='published'`
- `E2E: create article, publish, search on help center — article found`

---

#### 8.2 — Article Suggestion in Conversation View

**What**: Suggest relevant knowledge base articles in the conversation detail sidebar based on conversation subject and message content.

**Design**:

```typescript
// apps/api/src/services/kb-suggestion.service.ts

export class KbSuggestionService {
  /**
   * Find relevant articles for a conversation.
   * Uses a combination of:
   * 1. Full-text search against conversation subject and latest message body
   * 2. Tag-based matching (conversation tags vs article tags in meta)
   * 3. Ranked by relevance score + view count
   */
  async suggestArticles(
    orgId: string,
    conversationId: string,
    limit: number = 5,
  ): Promise<Array<{ article: KbArticleData; relevanceScore: number }>> { /* ... */ }
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/conversations/:id/suggested-articles` | Suggested KB articles |
| POST | `/api/conversations/:id/link-article` | Link article to conversation |

**Testing**:
- `Integration: suggestArticles — returns articles matching conversation subject keywords`
- `Integration: suggestArticles — articles ranked by relevance score`
- `Integration: link-article — article linked to conversation (visible to agent)`
- `E2E: open conversation about "password reset" — KB article about password reset suggested in sidebar`

---

## Phase 9: AI Features — Reply Drafting, Sentiment, and Intent

### Purpose

Implement the AI-native features that differentiate this platform: contextual reply drafting using full conversation history, sentiment analysis per message, intent classification, and conversation summarization. After this phase, agents see AI-generated reply suggestions, sentiment indicators, and automatic conversation summaries.

### Tasks

#### 9.1 — AI Service and LLM Integration

**What**: Core AI service that manages LLM calls with streaming, token usage tracking, and provider abstraction via Vercel AI SDK.

**Design**:

```typescript
// apps/api/src/services/ai.service.ts
import { generateText, streamText } from 'ai';
import { openai } from '@ai-sdk/openai';

export class AiService {
  private model = openai('gpt-4o');

  async generateReplyDraft(
    conversation: ConversationDetailData,
    messages: MessageData[],
    contact: ContactData,
    knowledgeBaseArticles: KbArticleData[],
    agentInstructions?: string,
  ): Promise<{ draft: string; confidence: number }> {
    const result = await generateText({
      model: this.model,
      system: `You are a customer support agent for ${conversation.organisationName}.
Your role is to draft a helpful, professional reply to the customer's email.

Customer context:
- Name: ${contact.displayName}
- Company: ${contact.company?.name || 'Unknown'}
- Previous conversations: ${contact.conversationCount}

Relevant knowledge base articles:
${knowledgeBaseArticles.map(a => `- ${a.title}: ${a.bodyPlain.slice(0, 500)}`).join('\n')}

${agentInstructions ? `Agent instructions: ${agentInstructions}` : ''}

Write a complete email reply. Be concise, professional, and empathetic. Address the customer's specific issue.`,
      prompt: messages.map(m => `[${m.direction === 'inbound' ? 'Customer' : 'Agent'}] ${m.bodyPlain}`).join('\n\n'),
    });

    return {
      draft: result.text,
      confidence: 0.85, // Placeholder; future: calibrated confidence from model
    };
  }

  async analyseSentiment(text: string): Promise<{
    sentiment: 'positive' | 'neutral' | 'negative' | 'frustrated';
    score: number; // -1.0 to 1.0
  }> { /* ... */ }

  async classifyIntent(text: string): Promise<{
    intent: string;        // e.g. 'order_status', 'refund_request', 'password_reset'
    confidence: number;
  }> { /* ... */ }

  async summariseConversation(messages: MessageData[]): Promise<string> { /* ... */ }
}
```

**Testing**:
- `Unit (mocked LLM): generateReplyDraft — returns structured draft with contact name used`
- `Unit (mocked LLM): analyseSentiment — "I'm very frustrated" returns sentiment='frustrated', score < -0.5`
- `Unit (mocked LLM): classifyIntent — "Where is my order?" returns intent='order_status'`
- `Unit (mocked LLM): summariseConversation — returns single paragraph summary`
- `Integration: AI service with rate limiting — handles rate limit errors with retry`
- `Integration: AI service — tracks token usage per organisation for billing`

---

#### 9.2 — AI Enrichment Worker

**What**: Background worker that runs sentiment analysis and intent classification on new inbound messages, and updates the conversation and message AI JSONB fields.

**Design**:

```typescript
// apps/api/src/workers/ai-enrichment.worker.ts

// BullMQ worker processing 'ai-enrichment' queue
// Triggered when a new inbound message is received
// Steps:
// 1. Run sentiment analysis on message body
// 2. Update message.ai JSONB with sentiment results
// 3. Run intent classification on message body
// 4. Update conversation.ai JSONB with intent, sentiment summary, suggested priority
// 5. If conversation has no summary, generate one
// 6. Emit 'ai.enrichment_complete' event for UI update

// Rate limiting: max 10 concurrent LLM calls per organisation
// Fallback: if LLM unavailable, skip enrichment (non-blocking)
```

**Testing**:
- `Integration: new message triggers enrichment — message.ai has sentiment field after worker runs`
- `Integration: new message triggers enrichment — conversation.ai has intent field`
- `Integration: LLM API timeout — worker retries with backoff, conversation unaffected`
- `Integration: LLM API down — enrichment skipped, no error propagated to user`

---

#### 9.3 — AI Reply Suggestions UI

**What**: AI-generated reply draft in the conversation view, with accept/edit/reject actions and feedback tracking.

**Design**:

```typescript
// apps/web/src/components/ai/AiReplyPanel.tsx

// Displays below the message thread, above the composer:
// 1. "AI Suggestion" label with confidence indicator
// 2. Preview of the draft reply
// 3. Three buttons: "Use this reply" (inserts into composer), "Edit" (inserts as editable), "Dismiss"
// 4. Feedback: thumbs up/down after using a suggestion (improves future suggestions)

export interface AiReplyPanelProps {
  conversationId: string;
  onAccept: (draft: string) => void;  // Inserts into composer
  onDismiss: () => void;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/conversations/:id/ai/draft` | Generate AI reply draft |
| POST | `/api/conversations/:id/ai/draft/feedback` | Submit feedback on draft |
| GET | `/api/conversations/:id/ai/summary` | Get conversation summary |

**Testing**:
- `Unit: AiReplyPanel — renders draft text with confidence percentage`
- `Unit: AiReplyPanel — "Use this reply" calls onAccept with draft text`
- `Unit: AiReplyPanel — "Dismiss" calls onDismiss and hides panel`
- `Integration: POST /api/conversations/:id/ai/draft — returns AI-generated draft`
- `Integration: POST /api/conversations/:id/ai/draft/feedback — records accepted/rejected`
- `E2E: open conversation, click "Generate AI Reply" — draft appears, click "Use" — draft inserted in composer`

---

#### 9.4 — Sentiment and Intent Display

**What**: Visual indicators for message sentiment and conversation intent in the UI.

**Design**:

```typescript
// Sentiment badge on each message: emoji + colour
// positive → green, "Satisfied"
// neutral → grey, "Neutral"
// negative → orange, "Unhappy"
// frustrated → red, "Frustrated"

// Intent badge on conversation: shown in conversation header
// e.g., "Order Status Inquiry" with confidence percentage

// Conversation list: colour-coded dot for overall sentiment
```

**Testing**:
- `Unit: SentimentBadge — renders correct emoji and colour for each sentiment value`
- `Unit: IntentBadge — renders intent label with confidence`
- `E2E: conversation with frustrated customer — red sentiment badge visible on messages`

---

## Phase 10: Notifications, Webhooks, and Integrations

### Purpose

Build the notification system (in-app, email, Slack), outbound webhooks for external integrations, and the Slack integration. After this phase, agents receive timely alerts for assignments, SLA breaches, and mentions, and external systems can subscribe to platform events.

### Tasks

#### 10.1 — In-App Notification System

**What**: Notification creation, delivery via Socket.IO, notification center UI, and read/unread management.

**Design**:

```typescript
// apps/api/src/services/notification.service.ts

export class NotificationService {
  async create(params: {
    userId: string;
    type: 'assignment' | 'mention' | 'sla_warning' | 'sla_breach' | 'new_message' | 'reply_collision';
    title: string;
    body?: string;
    resourceType: string;
    resourceId: string;
  }): Promise<void> {
    // 1. Insert into notification table
    // 2. Emit via Socket.IO to user's personal room
    // 3. If user has email notifications enabled, enqueue notification email
  }

  async markRead(userId: string, notificationIds: string[]): Promise<void> { /* ... */ }
  async markAllRead(userId: string): Promise<void> { /* ... */ }
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/notifications` | List notifications (paginated) |
| POST | `/api/notifications/read` | Mark specific notifications as read |
| POST | `/api/notifications/read-all` | Mark all as read |
| GET | `/api/notifications/unread-count` | Unread count for badge |

**Testing**:
- `Integration: create notification — record inserted and Socket.IO event emitted`
- `Integration: GET /api/notifications — returns user's notifications, newest first`
- `Integration: POST /api/notifications/read — marks specified notifications as read`
- `Integration: unread count after marking read — decremented correctly`
- `E2E: assign conversation to agent — agent receives notification in real time`

---

#### 10.2 — Outbound Webhook Delivery

**What**: Webhook registration and reliable event delivery with retry, signature verification, and delivery logging.

**Design**:

```typescript
// apps/api/src/workers/webhook-delivery.worker.ts

// BullMQ worker processing 'webhook-delivery' queue
// For each webhook event:
// 1. Load matching webhooks (by event type)
// 2. Build payload with event data
// 3. Sign payload with HMAC-SHA256 using webhook secret
// 4. POST to webhook URL with X-Signature-256 header
// 5. Record delivery status (success/failure, HTTP status, response time)
// 6. On failure: retry with exponential backoff (1min, 5min, 30min, 2hr)
// 7. After 5 failures: disable webhook and notify admin

// Webhook payload format:
export interface WebhookPayload {
  event: string;                    // e.g. 'conversation.created'
  timestamp: string;                // ISO 8601
  organisationId: string;
  data: Record<string, unknown>;    // Event-specific data
}

// Signature: HMAC-SHA256(secret, JSON.stringify(payload))
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/webhooks` | List webhooks |
| POST | `/api/webhooks` | Register webhook |
| PATCH | `/api/webhooks/:id` | Update webhook |
| DELETE | `/api/webhooks/:id` | Delete webhook |
| GET | `/api/webhooks/:id/deliveries` | Delivery log |
| POST | `/api/webhooks/:id/test` | Send test event |

**Testing**:
- `Unit: HMAC signature generation — matches expected signature for known payload+secret`
- `Integration: webhook delivery — POST sent to registered URL with correct signature header`
- `Integration: webhook delivery failure — retried with exponential backoff`
- `Integration: 5 consecutive failures — webhook disabled, admin notified`
- `Integration: test webhook — test event delivered successfully`

---

#### 10.3 — Slack Integration

**What**: Slack app integration for receiving notifications in Slack channels, replying to conversations from Slack, and assignment commands.

**Design**:

```typescript
// apps/api/src/integrations/slack/slack-integration.ts

// Slack events handled:
// 1. Incoming: Slack slash commands (/assign, /close, /reply)
// 2. Incoming: Slack interactive messages (button clicks for assignment)
// 3. Outgoing: Post message to Slack channel when conversation is created/assigned/breached

// Setup flow:
// 1. Admin installs Slack app via OAuth
// 2. Admin maps Slack channels to inboxes (e.g., #support → Support inbox)
// 3. Events flow: platform event → check Slack mapping → post to mapped channel

export interface SlackNotificationConfig {
  channelId: string;
  events: string[];  // ['conversation.created', 'sla.breached', 'conversation.assigned']
  inboxId?: string;  // Filter to specific inbox
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/integrations/slack/install` | Start Slack OAuth |
| GET | `/api/integrations/slack/callback` | Handle Slack OAuth callback |
| POST | `/api/integrations/slack/events` | Slack event webhook endpoint |
| POST | `/api/integrations/slack/commands` | Slack slash command endpoint |
| GET | `/api/integrations/slack/channels` | List available Slack channels |
| POST | `/api/integrations/slack/mappings` | Map channel to inbox |

**Testing**:
- `Integration (mocked Slack API): new conversation → Slack message posted to mapped channel`
- `Integration (mocked Slack API): SLA breach → Slack message with red warning`
- `Integration (mocked Slack API): /assign command → conversation assigned, confirmation posted`
- `Integration: Slack event verification — invalid signature returns 401`
- `E2E: install Slack, map channel, receive email — Slack notification appears`

---

## Phase 11: Audit Trail and GDPR Compliance

### Purpose

Build the comprehensive audit log, GDPR data subject access request (DSAR) workflow, contact anonymisation, and data retention policies. After this phase, the platform meets ISO 27001 audit logging requirements and GDPR Article 17 (right to erasure).

### Tasks

#### 11.1 — Audit Log System

**What**: Automatic audit log entries for all significant actions, with before/after change tracking and query endpoints.

**Design**:

```typescript
// apps/api/src/services/audit.service.ts

export class AuditService {
  async log(params: {
    organisationId: string;
    actorId: string | null;
    actorType: 'user' | 'system' | 'automation' | 'ai';
    action: string;
    resourceType: string;
    resourceId: string;
    changes?: Record<string, { old: unknown; new: unknown }>;
    context?: Record<string, unknown>;
  }): Promise<void> { /* INSERT INTO audit_log */ }
}

// Actions logged:
// conversation.created, conversation.assigned, conversation.status_changed,
// conversation.priority_changed, conversation.tagged, conversation.merged
// message.sent, message.draft_created
// contact.created, contact.updated, contact.merged, contact.anonymised
// sla_policy.created, sla_policy.updated
// sla.breached, sla.met
// user.invited, user.role_changed, user.deactivated
// automation_rule.created, automation_rule.triggered
// inbox.connected, inbox.disconnected
// webhook.created, webhook.delivery_failed
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/audit-log` | Query audit log (paginated, filtered by resource/actor/action) |
| GET | `/api/audit-log/export` | Export audit log as CSV |

**Testing**:
- `Integration: assign conversation — audit_log entry created with changes showing old/new assignee`
- `Integration: close conversation — audit_log entry with status change`
- `Integration: GET /api/audit-log?resourceType=conversation&resourceId=uuid — returns filtered entries`
- `Integration: GET /api/audit-log?actorType=automation — returns only automation-triggered entries`
- `Integration: audit log export — CSV with correct columns and data`

---

#### 11.2 — GDPR Compliance: Anonymisation and Data Export

**What**: Contact anonymisation (right to erasure), data subject access request (DSAR) export, and data retention policy enforcement.

**Design**:

```typescript
// apps/api/src/services/gdpr.service.ts

export class GdprService {
  /**
   * Anonymise a contact (GDPR Article 17 — right to erasure).
   * Clears all PII fields, preserves conversation structure.
   */
  async anonymiseContact(orgId: string, contactId: string): Promise<void> {
    await this.db.update(contact)
      .set({
        displayName: 'Anonymised Contact',
        primaryEmail: null,
        handles: [],
        company: {},
        crm: {},
        customFields: {},
        isAnonymised: true,
        updatedAt: new Date(),
      })
      .where(and(
        eq(contact.id, contactId),
        eq(contact.organisationId, orgId),
      ));

    // Update conversation participants JSONB to replace contact name/email
    // Create audit_log entry for anonymisation
  }

  /**
   * Generate DSAR export: all data related to a contact.
   */
  async generateDataExport(orgId: string, contactId: string): Promise<{
    contact: ContactData;
    conversations: ConversationDetailData[];
    messages: MessageData[];
  }> { /* ... */ }

  /**
   * Apply data retention policy: archive/delete conversations older than retention period.
   */
  async enforceRetention(orgId: string, retentionDays: number): Promise<{
    archived: number;
    deleted: number;
  }> { /* ... */ }
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/contacts/:id/anonymise` | Anonymise contact |
| POST | `/api/contacts/:id/data-export` | Generate DSAR export (ZIP) |
| POST | `/api/admin/retention/enforce` | Run retention policy |
| GET | `/api/admin/retention/preview` | Preview what would be affected |

**Testing**:
- `Integration: anonymise contact — display_name='Anonymised Contact', handles=[], primaryEmail=null`
- `Integration: anonymise contact — conversation participants updated to hide PII`
- `Integration: anonymise contact — audit_log entry created`
- `Integration: DSAR export — ZIP contains JSON files with all contact data`
- `Integration: retention enforcement — conversations older than N days archived`
- `Integration: already anonymised contact — second anonymisation is no-op`

---

## Phase 12: Production Hardening — Performance, Security, and Deployment

### Purpose

Prepare the platform for production deployment: rate limiting, security headers, Docker multi-stage builds, database connection pooling, monitoring, health checks, and deployment documentation. After this phase, the platform can be self-hosted or deployed to a cloud environment with confidence.

### Tasks

#### 12.1 — Security Hardening

**What**: Implement OWASP API Security Top 10 protections, CORS configuration, CSP headers, rate limiting, and input sanitization.

**Design**:

```typescript
// apps/api/src/plugins/rate-limit.ts
import rateLimit from '@fastify/rate-limit';

// Rate limits:
// - Auth endpoints (login, signup): 10 requests/minute per IP
// - API endpoints: 100 requests/minute per user
// - Webhook endpoints: 1000 requests/minute per IP (for provider callbacks)
// - AI endpoints: 20 requests/minute per user (LLM cost control)

// OWASP API Security considerations:
// API1 (Broken Object Level Authorization): All queries scoped by organisationId from session
// API2 (Broken Authentication): Lucia handles session management; bcrypt for passwords
// API3 (Broken Object Property Level Authorization): Zod schemas strip unknown properties
// API5 (Broken Function Level Authorization): requirePermission guards on all routes
// API8 (Security Misconfiguration): Strict CORS, CSP, no stack traces in production
```

```typescript
// apps/api/src/config.ts
export const config = {
  cors: {
    origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
    credentials: true,
  },
  rateLimit: {
    auth: { max: 10, timeWindow: '1 minute' },
    api: { max: 100, timeWindow: '1 minute' },
    webhook: { max: 1000, timeWindow: '1 minute' },
    ai: { max: 20, timeWindow: '1 minute' },
  },
  security: {
    bcryptRounds: 12,
    sessionMaxAge: 30 * 24 * 60 * 60, // 30 days
    csrfEnabled: true,
  },
};
```

**Testing**:
- `Integration: rate limit exceeded on auth endpoint — returns 429 with Retry-After header`
- `Integration: CORS request from unauthorized origin — blocked`
- `Integration: API request without session cookie — returns 401`
- `Integration: request to org A's data with org B's session — returns 404 (not 403, to prevent enumeration)`
- `Unit: XSS in message body — sanitized before storage`
- `Unit: SQL injection attempt in search parameter — parameterized query prevents injection`

---

#### 12.2 — Docker Production Build and Health Checks

**What**: Multi-stage Docker build for minimal production images, health check endpoints, graceful shutdown, and docker-compose production profile.

**Design**:

```dockerfile
# Dockerfile (multi-stage)
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@9 --activate

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/package.json apps/api/
COPY apps/web/package.json apps/web/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile

FROM base AS api
WORKDIR /app
COPY --from=deps /app/node_modules node_modules
COPY . .
RUN pnpm --filter api build
EXPOSE 3001
CMD ["node", "apps/api/dist/index.js"]

FROM base AS web
WORKDIR /app
COPY --from=deps /app/node_modules node_modules
COPY . .
RUN pnpm --filter web build
EXPOSE 3000
CMD ["pnpm", "--filter", "web", "start"]
```

```typescript
// apps/api/src/routes/health.ts
// GET /health — basic liveness check
// GET /health/ready — readiness check (DB connected, Redis connected, workers running)

export interface HealthResponse {
  status: 'ok' | 'degraded' | 'unhealthy';
  version: string;
  uptime: number;
  checks: {
    database: 'ok' | 'error';
    redis: 'ok' | 'error';
    workers: 'ok' | 'error';
    storage: 'ok' | 'error';
  };
}
```

**Testing**:
- `Integration: docker build — multi-stage build completes without errors`
- `Integration: docker compose up — all services healthy`
- `Integration: GET /health — returns ok with all checks passing`
- `Integration: GET /health/ready with Redis down — returns degraded status`
- `Integration: SIGTERM signal — graceful shutdown (in-flight requests complete, workers drain)`
- `E2E: docker compose production profile — full workflow (signup, connect inbox, receive email, reply) works`

---

#### 12.3 — Database Performance and Connection Pooling

**What**: Connection pooling, query optimization, database indexes verification, and migration safety checks.

**Design**:

```typescript
// apps/api/src/db/pool.ts
import { Pool } from 'pg';
import { drizzle } from 'drizzle-orm/node-postgres';

export function createPool() {
  return new Pool({
    connectionString: process.env.DATABASE_URL,
    max: parseInt(process.env.DB_POOL_MAX || '20'),
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 5000,
    statement_timeout: 30000,         // 30s query timeout
  });
}

// Slow query logging:
// Log any query taking > 1 second
// Include query text (parameterized) and execution plan

// Index verification queries:
// Ensure all expected indexes exist and are being used
// Run EXPLAIN ANALYZE on key queries to verify index usage
```

**Testing**:
- `Integration: concurrent 50 requests — connection pool handles without errors`
- `Integration: conversation list query with 10k conversations — response time < 200ms`
- `Integration: EXPLAIN ANALYZE on conversation list query — uses idx_conv_org_status_last index`
- `Integration: EXPLAIN ANALYZE on contact handle lookup — uses idx_contact_handles GIN index`
- `Unit: slow query logging — queries > 1s are logged with execution plan`

---

#### 12.4 — CI/CD Pipeline

**What**: GitHub Actions workflow for linting, testing, building, and publishing Docker images.

**Design**:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:migrate
      - run: pnpm test -- --coverage
        env:
          DATABASE_URL: postgres://postgres:test@localhost:5432/test
          REDIS_URL: redis://localhost:6379

  docker:
    needs: [lint-and-typecheck, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

**Testing**:
- `Integration: CI workflow — lint, typecheck, test, build all pass on clean checkout`
- `Integration: Docker image push — image published to GHCR on main branch merge`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                         ─── required by everything
    │
Phase 2: Email Integration                  ─── requires Phase 1
    │
Phase 3: Core Inbox UI                      ─── requires Phase 2
    │
    ├── Phase 4: SLA Tracking               ─── requires Phase 3
    │       │
    ├── Phase 5: Contact Management & CRM   ─── requires Phase 3
    │       │
    ├── Phase 6: Tags, Automation, Canned   ─── requires Phase 3
    │       │
    └── Phase 7: Analytics & Reporting      ─── requires Phase 3, benefits from Phase 4
            │
    Phase 8: Knowledge Base                  ─── requires Phase 3
            │
    Phase 9: AI Features                     ─── requires Phase 3, Phase 5
            │
    Phase 10: Notifications & Integrations   ─── requires Phase 3, Phase 4
            │
    Phase 11: Audit Trail & GDPR             ─── requires Phase 5, benefits from Phase 10
            │
    Phase 12: Production Hardening           ─── requires all previous phases
```

**Parallelism opportunities**:
- Phases 4, 5, 6, 7, and 8 can all be developed concurrently after Phase 3 is complete.
- Phase 9 (AI Features) can start after Phase 3 but benefits from Phase 5 (contact context).
- Phase 10 (Notifications) can start after Phase 3 but benefits from Phase 4 (SLA breach notifications).
- Phase 11 (GDPR) can start after Phase 5 but benefits from Phase 10 (audit logging in integrations).
- Phase 12 (Production Hardening) should be the final phase, after all features are implemented.

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (with real PostgreSQL and Redis in CI).
4. ESLint passes with zero warnings (`pnpm lint`).
5. TypeScript strict mode passes with zero errors (`pnpm typecheck`).
6. Docker build succeeds for both api and web targets.
7. Feature works end-to-end (manual verification or E2E test).
8. New API endpoints appear in auto-generated OpenAPI spec (`/api/docs`).
9. Database migrations are created and apply cleanly to a fresh database.
10. New configuration options documented in `.env.example`.
11. Audit log entries are created for all user-facing state changes.
12. No secrets or credentials are committed to the repository.
