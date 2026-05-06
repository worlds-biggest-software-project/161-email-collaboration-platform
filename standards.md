# Standards & API Reference

> Project: Email Collaboration Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls, encryption, and audit logging for email collaboration platforms handling confidential business communications; Annex A A.8.3 (Information access restriction), A.8.24 (Cryptography), and A.5.14 (Information transfer) are directly applicable. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of personal data in email content; email collaboration platforms processing employee and customer communications must comply with PII handling, retention, and disclosure requirements. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **RFC 5321 — SMTP: Simple Mail Transfer Protocol** — The foundational IETF standard for email transmission; defines the protocol for sending email between mail servers; the foundation of all email delivery infrastructure. URL: https://datatracker.ietf.org/doc/html/rfc5321

- **RFC 5322 — Internet Message Format** — IETF standard defining the syntax of email message headers and body; governs the structure of all email messages processed by collaboration platforms. URL: https://datatracker.ietf.org/doc/html/rfc5322

- **RFC 9051 — IMAP4rev2 (Internet Message Access Protocol)** — Updated IMAP standard (2021, superseding RFC 3501); the primary protocol for accessing and managing email in mailboxes; used by email collaboration platforms for inbox synchronisation, folder management, and flag operations. URL: https://datatracker.ietf.org/doc/html/rfc9051

- **RFC 8620 — JMAP: JSON Meta Application Protocol** — Modern IETF standard (2019) providing a REST/JSON-based alternative to IMAP; designed for efficient email access by mobile and web clients with push notifications, batch operations, and efficient sync; the emerging standard for next-generation email client integrations. URL: https://datatracker.ietf.org/doc/html/rfc8620

- **RFC 8621 — JMAP for Mail** — Extension to RFC 8620 defining JMAP-specific operations for email messages, mailboxes, threads, and identities. URL: https://datatracker.ietf.org/doc/html/rfc8621

- **RFC 2045-2049 — MIME: Multipurpose Internet Mail Extensions** — IETF standards defining the format for email message bodies with multiple parts, content types, encodings, and attachments; foundational for rich email content in collaboration platforms. URL: https://datatracker.ietf.org/doc/html/rfc2045

- **RFC 7208 — SPF: Sender Policy Framework** — IETF standard for email sender authentication via DNS TXT records; mandatory for all domains used by email collaboration platforms to prevent spoofing; enforced by Gmail and Outlook since 2025. URL: https://datatracker.ietf.org/doc/html/rfc7208

- **RFC 6376 — DKIM: DomainKeys Identified Mail** — IETF standard for cryptographic email signing; provides message integrity and sender verification; mandated by Gmail (February 2024), Outlook (May 2025), and Yahoo for bulk senders; becoming regulatory requirement in 2026. URL: https://datatracker.ietf.org/doc/html/rfc6376

- **RFC 7489 — DMARC: Domain-based Message Authentication, Reporting, and Conformance** — Builds on SPF and DKIM to provide domain-level email authentication policy; required by Gmail and Outlook for all senders; DMARCbis (next-generation DMARC) progressing as IETF Proposed Standard in 2025. URL: https://datatracker.ietf.org/doc/html/rfc7489

- **RFC 8058 — One-Click List-Unsubscribe** — Defines the List-Unsubscribe-Post header for one-click unsubscribe; required by Gmail and Yahoo for bulk senders from 2024; relevant to email collaboration platforms handling marketing or transactional email workflows. URL: https://datatracker.ietf.org/doc/html/rfc8058

- **RFC 4918 — WebDAV** — Foundation for CalDAV (calendar sync) and CardDAV (contact sync) used alongside email in collaboration platform suites. URL: https://datatracker.ietf.org/doc/html/rfc4918

- **RFC 6352 — CardDAV** — Protocol for addressbook data over HTTP/WebDAV; used by email collaboration platforms for contact management and sync with enterprise address books. URL: https://datatracker.ietf.org/doc/html/rfc6352

- **RFC 6749 — OAuth 2.0** — Authorization framework used by all major email APIs (Gmail API, Microsoft Graph, Nylas) for delegated access to email on behalf of users; mandatory for programmatic email integration without storing user passwords. URL: https://datatracker.ietf.org/doc/html/rfc6749

### Data Model & API Specifications

- **OpenAPI 3.1** — Used by Front, Help Scout, and email API platforms (Nylas, Unipile) to describe their REST management APIs; enables SDK code generation and automated integration testing. URL: https://spec.openapis.org/oas/latest.html

- **Google Gmail API v1** — REST/JSON API providing access to Gmail mailbox data (threads, messages, labels, drafts, settings); the authoritative API for Gmail integration in email collaboration platforms; supports push notifications via Pub/Sub for real-time inbox events. URL: https://developers.google.com/workspace/gmail/api/reference/rest

- **Microsoft Graph API — Mail Resources** — REST/JSON API for Outlook/Exchange mail (messages, mail folders, attachments, rules); replaces Exchange Web Services (EWS, deprecated October 1, 2026); supports OAuth 2.0 delegated and application permissions. URL: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview

- **MIME Multipart** — The MIME standards (RFC 2045-2049) define the structured format for rich email content including HTML bodies, plain text alternatives, inline images, and file attachments; all email collaboration platforms must parse and compose MIME messages correctly.

### Security & Authentication Standards

- **GDPR Article 6 & 32 — Lawful Processing and Security of Processing** — Email collaboration platforms processing employee and customer emails must have lawful basis for processing; must implement appropriate access controls, encryption in transit and at rest, and data retention policies; data residency requirements increasingly common for EU enterprise customers. URL: https://gdpr-info.eu/art-32-gdpr/

- **GDPR Article 17 — Right to Erasure** — Email collaboration platforms must support deletion of personal data from shared inboxes and archived emails on request; particularly relevant for customer-facing email collaboration tools. URL: https://gdpr-info.eu/art-17-gdpr/

- **HIPAA — §164.312 — Technical Safeguards** — Email collaboration platforms used for healthcare communications must implement encryption, access controls, and audit logging; requires Business Associate Agreements (BAAs) with platform providers when processing PHI. URL: https://www.hhs.gov/hipaa/for-professionals/security/

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS email collaboration platforms; Front, Help Scout, Missive, and Nylas maintain SOC 2 Type II reports. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration ensuring email collaboration platform access is tied to enterprise identity lifecycle. URL: https://docs.oasis-open.org/security/saml/v2.0/

- **SCIM 2.0 (RFC 7643/7644)** — Used for automated user provisioning and deprovisioning in enterprise email collaboration tool deployments. URL: https://datatracker.ietf.org/doc/html/rfc7643

- **MTA-STS (RFC 8461) — SMTP MTA Strict Transport Security** — Defines policy for requiring TLS encryption for email delivery; increasingly required by enterprise email security standards; relevant to email collaboration platform outbound email routing. URL: https://datatracker.ietf.org/doc/html/rfc8461

- **OWASP API Security Top 10 (2023)** — Governs REST API security for shared inbox and email management APIs; API1 (Broken Object Level Authorization) is critical — users must only access conversations they are authorised to view. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Email is an active integration target for the MCP ecosystem in 2025-2026:

- **email-mcp (IMAP + SMTP MCP Server)** — Open-source MCP server providing full IMAP + SMTP support for reading, searching, sending, managing, and organising email from any AI assistant via the Model Context Protocol. URL: https://github.com/codefuturist/email-mcp

- **JMAP Email MCP Server** — Open-source MCP server (wyattjoh/jmap-mcp) for interacting with JMAP email servers using the jmap-jam client library and Deno; implements the modern RFC 8620/8621 protocol for AI-native email access. URL: https://github.com/wyattjoh/jmap-mcp

- **Agentic Email Architecture (2025-2026)** — Emerging pattern where AI email agents use JMAP (RFC 8620) directly for efficient email access; JMAP's batch operations, push notifications, and structured data model make it significantly more suitable for AI agent workflows than legacy IMAP. URL: https://jaime.win/agentic-email/

---

## Similar Products — Developer Documentation & APIs

### Gmail API (Google Workspace)

- **Description:** REST API for accessing Gmail mailboxes (threads, messages, labels, drafts, settings); supports push notifications via Google Pub/Sub for real-time inbox events; rate limit: 1 billion units/day; sending quotas: 2,000/day (Workspace), 500/day (personal).
- **API Documentation:** https://developers.google.com/workspace/gmail/api/reference/rest
- **SDKs/Libraries:** Google API client libraries (Python, Java, .NET, Go, JavaScript, PHP, Ruby); google-auth for OAuth 2.0
- **Developer Guide:** https://developers.google.com/workspace/gmail/api/guides
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, MIME, iCalendar (RFC 5545)
- **Authentication:** OAuth 2.0 (user authorization); Service accounts with domain-wide delegation for workspace-wide access

### Microsoft Graph API — Mail

- **Description:** REST/JSON API for Outlook/Exchange mail management across Microsoft 365, Outlook.com, and Exchange Online; replaces Exchange Web Services (EWS, deprecated October 1, 2026); supports message rules, shared mailboxes, and online meeting creation.
- **API Documentation:** https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
- **SDKs/Libraries:** Microsoft Graph SDKs (Python, .NET, Go, Java, JavaScript/TypeScript)
- **Developer Guide:** https://developer.microsoft.com/en-us/graph
- **Standards:** REST/JSON, OpenAPI 3.1, OAuth 2.0 (Azure AD), MIME, iCalendar
- **Authentication:** Azure AD OAuth 2.0 (delegated or application permissions); managed identity for Azure workloads

### Front

- **Description:** Leading shared inbox and email collaboration platform; unifies email, SMS, social, and chat channels; REST Core API for messages, contacts, channels, inboxes, teammates, and analytics; webhooks for real-time events; 100+ native integrations.
- **API Documentation:** https://dev.frontapp.com/docs/welcome
- **API Reference:** https://dev.frontapp.com/reference/inboxes
- **SDKs/Libraries:** front-sdk (JavaScript); REST API (JSON); Postman collection
- **Developer Guide:** https://dev.frontapp.com/docs/welcome
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Webhooks
- **Authentication:** Bearer token (API token from Front settings); OAuth 2.0 for partner apps

### Help Scout

- **Description:** Customer-focused shared inbox platform with a help desk orientation; Inbox API 2.0 (HTTPS, OAuth 2, CORS); Docs API for help centre articles; Beacon JavaScript SDK for in-app help widgets; strong focus on human-touch customer support.
- **API Documentation:** https://developer.helpscout.com/mailbox-api/
- **SDKs/Libraries:** helpscout-python; helpscout-node; REST API (JSON); Beacon JS SDK
- **Developer Guide:** https://developer.helpscout.com/
- **Standards:** REST/JSON (Inbox API v2), OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0 (app credentials); API keys for legacy integrations

### Missive

- **Description:** Team email collaboration platform with real-time co-writing and shared inboxes; REST API for conversations, messages, posts, labels, and contacts; strong for small teams requiring synchronous email collaboration with live presence indicators.
- **API Documentation:** https://missiveapp.com/help/api-documentation
- **SDKs/Libraries:** REST API (JSON); Webhooks; Zapier integration
- **Developer Guide:** https://missiveapp.com/help/api-documentation
- **Standards:** REST/JSON, OpenAPI (partial), OAuth 2.0, Webhooks
- **Authentication:** API token (per team); OAuth 2.0 for integrations

### Nylas

- **Description:** Unified email (+ calendar + contacts) API platform aggregating Gmail, Microsoft 365/Exchange, IMAP/SMTP, and Yahoo; REST API with consistent interface across all providers; supports send, sync, webhooks, and full MIME parsing.
- **API Documentation:** https://developer.nylas.com/docs/api/
- **SDKs/Libraries:** nylas-python; nylas-node; nylas-java; nylas-ruby; nylas-go
- **Developer Guide:** https://developer.nylas.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, MIME, iCalendar (RFC 5545), IMAP, SMTP
- **Authentication:** OAuth 2.0 (per-user access tokens via Nylas auth flows); API key for management

### Superhuman

- **Description:** Premium AI-powered email client for individuals and teams; focused on speed, keyboard shortcuts, and AI triage; no public REST API (closed platform); integrations via Zapier and Zapier webhooks only; targeting enterprise high-performers.
- **API Documentation:** No public API (closed platform)
- **SDKs/Libraries:** Zapier integration; no official SDK
- **Developer Guide:** Not available
- **Standards:** Proprietary; underlying email via IMAP/SMTP and Gmail API / Microsoft Graph
- **Authentication:** Google OAuth 2.0; Microsoft OAuth 2.0

### Unipile (Unified Email + LinkedIn + WhatsApp API)

- **Description:** Unified communication API aggregating email (Gmail, Outlook, IMAP), LinkedIn, WhatsApp, and Slack under a single REST API; positioned for AI sales assistant and recruitment automation use cases.
- **API Documentation:** https://developer.unipile.com/
- **SDKs/Libraries:** REST API (JSON); Python SDK; Node.js SDK
- **Developer Guide:** https://www.unipile.com/email-api-guide/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** API key; OAuth 2.0 for end-user email account connections

---

## Notes

- **DMARC/DKIM/SPF as regulatory requirements (2025-2026)**: Gmail began rejecting non-compliant bulk email in November 2025; Outlook followed in May 2025; DMARCbis is progressing as an IETF Proposed Standard in 2025. Email collaboration platforms must enforce these authentication standards for all outbound email.

- **EWS deprecation (October 1, 2026)**: Microsoft Exchange Web Services is being retired; all email integrations with Exchange/Outlook must migrate to Microsoft Graph API before this date — affects all email collaboration platforms with Exchange support.

- **JMAP (RFC 8620) as the AI-native email protocol**: JMAP's batch operations, push notifications, and structured JSON data model make it significantly more efficient for AI agent email workflows than legacy IMAP; the JMAP MCP server represents the emerging architecture for AI-integrated email platforms.

- **Open-source landscape**: There is no dominant open-source shared inbox collaboration platform; Roundcube (GPL, web client), Dovecot (LGPL, IMAP server), and Postfix (IPL, MTA) are the core open-source email infrastructure components; James (Apache, full email server with JMAP support) is the most complete open-source email server stack.

- **GDPR and email archives**: Email collaboration platforms must implement DSAR (Data Subject Access Request) workflows and deletion capabilities across shared inboxes and email archives — a frequently overlooked compliance requirement.

- **SPF/DKIM/DMARC for shared inbox domains**: Email collaboration platforms using custom sending domains (e.g., `support@company.com`) require proper SPF, DKIM, and DMARC configuration for each customer domain — a key operational consideration for SaaS email collaboration tools.
