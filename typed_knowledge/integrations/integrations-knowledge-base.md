---
title: Integrations Domain Knowledge Base
generated: 2026-03-19T13:38:05.872Z
generated_by: n8n KT Refinement Pipeline
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (41 sessions)
purpose: AI coding agent context — load this file when working on the Integrations domain
---

# Table of Contents

- [Integrations Domain Knowledge Base](#integrations-domain-knowledge-base)
  - [1. Domain Overview](#1-domain-overview)
  - [2. Architecture Map](#2-architecture-map)
    - [High-Level Data Flows](#high-level-data-flows)
    - [Core Components & Relationships](#core-components-relationships)
  - [3. Module Reference](#3-module-reference)
    - [Integration Manager](#integration-manager)
    - [Mappings Manager](#mappings-manager)
    - [Delta Sync Manager](#delta-sync-manager)
    - [Delta Sync Worker](#delta-sync-worker)
    - [Full Sync Manager](#full-sync-manager)
    - [Full Sync Producer](#full-sync-producer)
    - [Full Sync Consumer](#full-sync-consumer)
    - [Sluice Worker](#sluice-worker)
    - [All Sub Worker (Audience Subscription Worker)](#all-sub-worker-audience-subscription-worker)
    - [Batch Production Worker](#batch-production-worker)
    - [Outbound Worker](#outbound-worker)
    - [Broker Service](#broker-service)
    - [Squid Proxy (Reverse Proxy)](#squid-proxy-reverse-proxy)
    - [Generic Connector](#generic-connector)
    - [Keyspace Management](#keyspace-management)
    - [Profile List Sync (Queries & Profiles)](#profile-list-sync-queries-profiles)
    - [Unified Data (HTTP/2 Streaming)](#unified-data-http2-streaming)
    - [Form Submission Handler](#form-submission-handler)
    - [Consent Management & Bidirectional Sync](#consent-management-bidirectional-sync)

---

# Integrations Domain Knowledge Base

## 1. Domain Overview

The Integrations domain is a multi-system bridge that synchronizes customer profile data, events, and consent between Apsis (email/SMS/marketing automation platform) and external CRM systems (Salesforce, Microsoft Dynamics, Efficy Enterprise, Tribe, HubSpot, Maxo, and others). It handles inbound data sync (contacts, attributes, consents, lists), outbound event propagation (email/SMS/form activities, MA flows, event tool ratings, NPS surveys), webhook management, field mapping, and real-time/batch synchronization. The domain must maintain eventual consistency, enforce data security, respect rate limits, and prevent profile duplication across keyspaces while enabling customers to use their CRM as a system of record for attributes and Apsis as a marketing execution and consent management layer.

---

## 2. Architecture Map

### High-Level Data Flows

**INBOUND (CRM → Apsis):**
- CRM webhooks → Delta Sync Manager → Delta Sync Worker → Profile/Audience DB
- CRM full contacts (paginated) → Full Sync Producer → Full Sync Queue → Full Sync Consumer → Audience DB
- CRM consent events → Consent mapping → Profile timeline
- CRM profile lists/segments → Queries & Profiles engine → Audience tags

**OUTBOUND (Apsis → CRM):**
- Apsis events (email, SMS, form, MA task, event tool) → All Sub Worker (verify, route) → Kafka partition → Batch Production Worker (group, batch 200KB or 1s timeout) → Outbound Worker → CRM via connector libraries
- Apsis consent changes → Consent handlers → CRM bidirectional sync
- Form submissions → Form Submission Handler → Outbound Manager → CRM

### Core Components & Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│ EXTERNAL CRM SYSTEMS (Salesforce, Dynamics, Efficy, Tribe, etc) │
└─────────┬─────────────────────────────────────────────────────┬─┘
          │ Webhooks (inbound) │ API calls (outbound)
          │                    │
┌─────────▼────────────────────┴─────────────────────────────────┐
│              MEMBRANE / GENERIC CONNECTOR                       │
│  (API contract, OAuth, field mapping templates, capabilities)  │
└─────┬──────────────────────────────────────────────────────┬───┘
      │                                                      │
┌─────▼────────────────────┐                    ┌──────────▼──────┐
│  DELTA SYNC             │                    │  OUTBOUND       │
│  - Webhook receiver     │                    │  - All Sub      │
│  - Real-time events     │                    │  - Batch Prod   │
│  - Message queuing      │                    │  - Outbound Wkr │
│  - Consumer processing  │                    │  - Dead Letter Q │
└─────┬────────────────────┘                    └──────────┬──────┘
      │                                                     │
      │  FULL SYNC                                         │
      │  - Producer (paginate CRM)                         │
      │  - Consumer (apply mappings)                       │
      │  - Sync conditions filtering                       │
      │                                                     │
      │  PROFILES & TAGS                                   │
      │  - Query engine (complex CRM queries)              │
      │  - Consent mappings (bidirectional)                │
      │  - Field mappings (universal/specific/connection)  │
      │                                                     │
      └─────────────────┬──────────────────────────────────┘
                        │
                  ┌─────▼─────────┐
                  │  APSIS CORE   │
                  │  - Audience   │
                  │  - Profiles   │
                  │  - Consent    │
                  │  - Events     │
                  └───────────────┘
```

---

## 3. Module Reference

### Integration Manager
- **Purpose**: CRUD operations for integrations; handles installation/uninstallation workflows; retrieves CRM schema metadata; manages event listener registration.
- **Key files**: `integration_manager.go`; connector libraries (`lib/connectors/*`)
- **Tech stack**: Go; ECS task; REST API
- **Data flow**: Installation request → validate credentials → register webhooks in CRM → create keyspace → store metadata in DB. On uninstall: remove listeners, delete mappings, attempt CRM cleanup, then delete credentials.
- **Business rules**:
  - Installation fails if CRM webhook registration fails (atomic operation).
  - Keyspace discriminator generated from section discriminator hash (first 8 chars) + connector logical name.
  - One integration per section for Maxo/Dynamics (deep linking constraint); multiple allowed for Efficy Enterprise.
  - Do NOT expose force-delete in UI without explicit disclaimer about orphaned webhooks.
- **Configuration**: Connector config stored per CRM type (`integration_id`, `display_name`, `additional_entities` for leads/silhouettes).
- **Integration points**: External CRM systems (OAuth, API keys); AWS Parameter Store (credential storage); Apsis database (metadata persistence).
- **Gotchas**:
  - Reinstalling integration reverts keyspace mapping to bootstrapped defaults, overwriting manual DB changes.
  - If CRM API key rotated after installation, uninstall may fail at webhook deletion step.
  - Consent/subscription mappings must exist in BOTH Apsis and CRM before sync can occur.

### Mappings Manager
- **Purpose**: Persists and validates field mappings (Apsis ↔ CRM), consent/subscription mappings, and sync conditions. Provides lookup via cache for Delta/Full Sync Workers.
- **Key files**: Mappings database table; caching layer
- **Tech stack**: ECS task; SQL database; Apsis Audience schema
- **Data flow**: Admin configures mappings in UI → Mappings Manager persists → cache invalidated → workers query cache for field/consent transforms.
- **Business rules**:
  - Only mapped fields trigger webhooks in external CRM.
  - Field mappings are unidirectional (CRM → Apsis) except for consent (bidirectional).
  - Subscription mapping: maps external CRM subscription/profile type to Apsis subscription folder + channel (email/SMS).
  - Sync conditions use AND logic only; all conditions must be true for profile to sync.
  - Changes to mappings require full sync to backfill existing profiles with new attributes.
- **Configuration**: Three-level hierarchy: universal (all CRMs), integration-specific (per CRM type), connection-specific (per customer). Field type defined at each level.
- **Integration points**: External CRM schema (via Integration Manager), Apsis profile schema.
- **Gotchas**:
  - Removing a mapping does NOT delete previously synced data from Apsis.
  - Sync condition operator set is limited (currently only equals, contains behind feature flag); no negation/OR.
  - Consent mappings must use record ID field to identify profiles; if record ID empty, consent update fails silently.

### Delta Sync Manager
- **Purpose**: Inbound webhook entry point for real-time CRM events (profile/consent changes). Validates payload format, queues messages for worker processing.
- **Key files**: Delta Sync Manager (CloudWatch log group); webhook receiver endpoint
- **Tech stack**: HTTP endpoint; SQS FIFO queue producer; message validation
- **Data flow**: CRM webhook POST → validate signature/format → extract mapped fields → enqueue to SQS FIFO (grouped by message_group_id) → Delta Sync Worker consumes.
- **Business rules**:
  - Messages grouped by `[account_section integration_id CRM_id]` (FIFO ordering within group).
  - Webhook signature verification via HMAC-SHA256 using stored webhook secret.
  - Only mapped fields in webhook trigger processing; unmapped fields ignored.
- **Configuration**: Webhook registration URL and secret generated during installation; secret stored in Credential Store (encrypted).
- **Integration points**: External CRM webhooks (inbound HTTP POST).
- **Gotchas**:
  - If webhook payload contains unmapped fields, entire message may be rejected or silently skipped.
  - Order preservation per FIFO group is critical; ensure message_group_id includes CRM_id to prevent blocking unrelated profiles.
  - Webhook timeout (retry by CRM) if response delayed; keep response time <1 second.

### Delta Sync Worker
- **Purpose**: Processes Delta Sync Manager queue messages; applies sync conditions, validates data formats (e.g., phone number), updates Apsis profiles and consent state.
- **Key files**: Consumer code (CloudWatch log group); validation logic
- **Tech stack**: ECS task; message consumer; Audience API client
- **Data flow**: SQS message → evaluate sync conditions → format validation → Audience API call (create/update profile or consent) → log result.
- **Business rules**:
  - Sync conditions evaluated on APSIS side, not CRM side (except new proposal to move to CRM for large datasets).
  - Phone number validation: must match standard format (e.g., `+46700xxxxx`); invalid format silently skipped.
  - Consent update uses record ID field to match profile; if ID not in keyspace, update fails silently.
  - If producer fails, consumer should exit gracefully without completing sync (unclear if this is enforced).
- **Configuration**: Caching TTL for credentials and event definitions (~10 minutes recommended).
- **Integration points**: Apsis Audience API (profile updates, consent changes).
- **Gotchas**:
  - Concurrent map writes race condition can occur (exit code 2); appears transient despite unchanged code for 2 years.
  - Validation failures (phone format) do not propagate errors to user; attribute silently omitted from sync.
  - If keyspace changed in DB but CRM still sends old identifier type, profile match fails.

### Full Sync Manager
- **Purpose**: Orchestrates on-demand full data synchronization from CRM to Apsis. Spawns producer and consumer ECS tasks, monitors progress, waits for Audience ingestion delay before marking complete.
- **Key files**: Full Sync Manager orchestration code; Full Sync Report (integration detail view)
- **Tech stack**: ECS task orchestration; CloudWatch Logs; SQS FIFO queue (temporary, per sync)
- **Data flow**: User triggers full sync → Manager spawns Producer task → Producer paginates CRM, counts contacts, updates `total_is_known` DB flag → queues messages → Consumer spawns and processes → applies mappings, exports consent from Audience, compares, sends updates → Manager waits for ingestion completion (15-min timeout standard) → marks sync complete.
- **Business rules**:
  - Full sync required for initial data backfill; required after adding new mapped attributes; can be triggered manually anytime.
  - `total_is_known` flag must be set by producer to signal consumer that all messages will arrive (critical for consumer termination logic).
  - Producer and consumer are separate ECS tasks; if producer fails, signal not propagated (unclear if consumer orphans).
  - Consumer consensus export timeout: 15 minutes without callback mechanism (architectural limitation).
  - Consent deduplication optimization: if Audience already has opt-in for contact/topic, don't resend (skip redundant write).
- **Configuration**: Page size, worker thread count, retry policy (no hardcoded values found in transcripts).
- **Integration points**: CRM system (paginated API), Audience service (consent export, profile updates).
- **Gotchas**:
  - Concurrent map writes in consumer can cause exit code 2 (unrecoverable error); restart sync required.
  - Audience export timeout does NOT fail the sync; optimization aborted and all consents reprocessed regardless of Audience state.
  - Memory crash possible for 8M+ contacts; in-memory consent export optimization causes heap exhaustion.
  - One-line fix proposed: add activity ID to batch grouping key to enable parallel form submission processing (currently forms of same type block each other on CRM error).

### Full Sync Producer
- **Purpose**: Paginated CRM API client that fetches all contacts in batches; transforms to internal format; enqueues to temporary SQS queue.
- **Key files**: Producer code; connector libraries (pagination logic)
- **Tech stack**: ECS task; HTTP client; CRM connector libraries
- **Data flow**: CRM paginated GET (page 0, 1, 2, ...) → normalize to Justin format → SQS put → repeat until empty page → set `total_is_known=true` in DB → exit.
- **Business rules**:
  - Downloads ALL contacts from CRM (no pre-filtering at CRM level); sync conditions applied later by consumer.
  - Producer errors should update DB with status=failed and total_is_known=true, but currently this propagation is incomplete (bug).
  - Pagination: requests contacts until empty page received; must handle CRM-specific cursor formats.
  - Does not validate credentials; relies on upstream credential storage.
- **Configuration**: CRM API credentials (decrypted from Credential Store by producer at runtime).
- **Integration points**: External CRM system (paginated contact fetch API).
- **Gotchas**:
  - Consent data download: ~2 consents per contact × 8M contacts = 16M records. If all downloaded without filtering, causes massive bandwidth waste and timeout.
  - Credentials cached in-memory; validation failures not clear (rely on CRM error response).
  - Producer failures (invalid credentials, format changes) not propagated to consumer, causing indefinite waits.

### Full Sync Consumer
- **Purpose**: Processes Full Sync Producer queue messages; applies field/consent mappings and sync conditions; sends profiles and consent updates to Audience.
- **Key files**: Consumer code (CloudWatch log group); validation logic
- **Tech stack**: ECS task; message consumer; Audience API client; Redis (worker thread coordination)
- **Data flow**: SQS message → evaluate sync conditions → validate data formats → Audience API call → log result. For consent: export current state from Audience, compare with incoming, send deltas.
- **Business rules**:
  - Only processes messages if `total_is_known` flag set in DB OR status in [cancelled, failed] (termination condition).
  - Consent export optimization: if Audience already has opt-in for contact/topic, skip redundant message (saves API calls).
  - Worker threads initialized from Redis config; consumer exits if producer fails (if implemented).
- **Configuration**: Worker thread count, consent export timeout (15 minutes).
- **Integration points**: Apsis Audience API (profile create/update, consent export and update).
- **Gotchas**:
  - Consent export timeout does NOT fail the sync; consumer continues and reprocesses all consents.
  - If producer crashes without setting total_is_known, consumer waits indefinitely (up to ~8-hour hard timeout per context).
  - Race condition in concurrent map writes (exit code 2); transient and requires full sync restart.
  - Consumer does not validate whether producer actually completed; relies entirely on DB flag and queue state.

### Sluice Worker
- **Purpose**: Gates real-time delta sync updates during full sync operations. Buffers messages with visibility timeout if full sync active; forwards to Delta Sync Worker otherwise.
- **Key files**: Sluice Worker code
- **Tech stack**: ECS task; SQS message consumer
- **Data flow**: Sluice queue input → check if full sync active for this integration → if active, increase VisibilityTimeout and re-queue; if inactive, forward to delta_sync_worker_queue.
- **Business rules**:
  - Full sync status checked per integration (not globally); enables other integrations to continue delta sync while one is syncing.
  - Special handling for FSC Enterprise 12.0: if sync condition fields missing from profile, make callback to CRM to fetch full record.
  - Ensures eventual consistency: newer real-time updates don't overwrite older full sync data mid-sync.
- **Configuration**: Full sync status check (likely polls DB).
- **Integration points**: Apsis profile metadata (full sync status lookup).
- **Gotchas**:
  - FSC Enterprise 12.0 callback for missing fields can create latency; risk of double API calls if fields later populated.
  - Visibility timeout backoff may cause messages to age; monitor SQS metrics for queue depth anomalies.
  - If full sync hangs, sluice will buffer messages indefinitely.

### All Sub Worker (Audience Subscription Worker)
- **Purpose**: First-stage processing of outbound events (email, SMS, form, MA task, event tool). Validates integration exists and is enabled; routes to Kafka partition for batching.
- **Key files**: All Sub Worker code; Kafka producer
- **Tech stack**: ECS task; SQS FIFO consumer; Kafka producer
- **Data flow**: All Sub Queue (SQS FIFO, from Audience events) → extract profile/activity/event data → query Integration Manager to verify integration active → verify activity can be synced (feature flag) → produce to Kafka partition (keyed by `[account section integration_id activity_id]` to enable parallel processing of unrelated activities).
- **Business rules**:
  - Only routes events for integrations that are installed and active.
  - Feature flag checked: some CRM systems don't support form sync, event tool sync, etc.; if not supported, message dropped.
  - Kafka partitioning by activity ID (not just activity type) enables independent retry/backoff per activity.
  - Event filtering: personal accounts (PA*) can be safely ignored if needed (low-priority).
- **Configuration**: Feature flags per integration (form_sync_enabled, event_tool_sync_enabled, etc.).
- **Integration points**: Audience event stream (inbound), Integration Manager (metadata queries).
- **Gotchas**:
  - If Integration Manager unreachable, messages blocked in queue (critical dependency).
  - Feature flag mismatch (CRM doesn't support form sync but flag enabled) causes silent message drop.
  - If activity ID not included in Kafka partition key, forms of same type block each other on CRM error (critical performance bug).

### Batch Production Worker
- **Purpose**: Consumes Kafka partitions; groups events by integration; batches until 200KB size or 1-second timeout; produces to Outbound Queue.
- **Key files**: Batch Production Worker code (one per activity type: email, SMS, form, event_tool, flow)
- **Tech stack**: ECS task; Kafka consumer; SQS FIFO producer
- **Data flow**: Kafka partition input (pre-grouped by account/section/integration/activity_id) → accumulate messages in memory → when 200KB reached or 1s elapsed, generate batch ID (UUID) and send to Outbound Queue → repeat.
- **Business rules**:
  - Batch size capped at 200KB (leaves 56KB buffer below SQS 256KB limit).
  - Timeout set to 1 second; if message arrives and no batch pending, start new batch timer.
  - Each batch assigned unique batch ID for deduplication in external systems.
- **Configuration**: 200KB size limit, 1-second timeout.
- **Integration points**: Kafka (inbound), SQS Outbound Queue (outbound).
- **Gotchas**:
  - If batcher crashes, in-memory batch lost; no persistence (data loss possible).
  - 200KB limit may cause frequent small batches if event payload large; monitor batch size distribution.

### Outbound Worker
- **Purpose**: Consumes Outbound Queue batches; uses connector libraries to send event data to external CRM systems; handles retries and dead-letter routing.
- **Key files**: Outbound Worker code; connector libraries (batch sending logic)
- **Tech stack**: ECS task; SQS FIFO consumer; HTTP client (via connector libraries); Broker Service (for credential decryption)
- **Data flow**: Outbound Queue batch → Broker Service (decrypt CRM credentials, add auth header) → Squid Proxy (domain whitelist check) → CRM API endpoint (batch event send) → on success, batch deleted; on 4xx/5xx, message returned to queue with backoff delay (15-min max, 4-hour total window) → on 4-hour exhaustion, dead-letter routing.
- **Business rules**:
  - Exponential backoff: 15s, 30s, 1m, 5m, 15m, then 15m fixed until 4-hour window exhausted.
  - All CRM errors trigger retry (cannot distinguish permanent vs. transient from CRM response).
  - Batch ID included in request for external system deduplication.
  - Credential decryption requires KMS key in same region (multi-region KMS limitation for disaster recovery).
- **Configuration**: Backoff policy (max interval, total duration); credential decryption key (KMS).
- **Integration points**: External CRM systems (via Broker Service and Squid Proxy).
- **Gotchas**:
  - CRM error response vague ('unexpected error'); outbound worker cannot distinguish permanent errors and retries indefinitely.
  - Outbound mapping misconfiguration (mapping form field to non-existent CRM attribute) causes CRM database error; blocks all subsequent forms of same type.
  - FCC Enterprise returns HTTP 200 for invalid JSON requests; error only visible in response body (violates HTTP semantics).
  - One FIFO queue grouping issue: grouping by activity-type only (not activity-type + activity_id) causes one failing form to block all other forms for 4+ hours (proposal: add activity_id to grouping key).

### Broker Service
- **Purpose**: HTTP proxy that decrypts customer CRM credentials from encrypted storage, injects into Authorization header, forwards request to destination CRM endpoint.
- **Key files**: Broker Service code; credential lookup logic
- **Tech stack**: HTTP middleware; AWS KMS (decryption); AWS Secrets Manager (credential storage lookup)
- **Data flow**: Outbound Worker → Broker Service (with account/section/integration/activity data) → Broker retrieves encrypted credentials from Credential Store → KMS decrypts → add Authorization header → forward to Squid Proxy → forward to CRM endpoint → return response.
- **Business rules**:
  - Developers have NO access to plaintext credentials; decryption happens only in Broker Service.
  - Credentials keyed by connection ID (not integration type); enables per-customer credential isolation.
  - KMS key must be in same region as credential storage; multi-region KMS requires replica keys.
- **Configuration**: KMS key ID, Credential Store endpoint, credential lookup SQL/API.
- **Integration points**: AWS KMS (decryption), Credential Store (encrypted credential retrieval).
- **Gotchas**:
  - KMS key is single-region only (current state); disaster recovery to different region requires multi-region KMS setup.
  - Decryption latency adds to each request; monitor if request volume high.
  - If KMS key deleted or access revoked, all outbound requests immediately fail with cryptographic errors.

### Squid Proxy (Reverse Proxy)
- **Purpose**: Network security layer that validates outbound traffic against whitelist of allowed CRM domains per customer. Blocks requests to non-whitelisted destinations.
- **Key files**: Squid configuration files (`allowed_staging.txt`, `allowed_prod.txt`); external access control script
- **Tech stack**: Squid proxy; domain whitelist ACLs; external access control script (Python/Bash/Go)
- **Data flow**: Request from Broker Service → Squid checks destination domain against static whitelist (allowed_*.txt) → if miss, pipes to external access control script → script queries database (allowed_domains table) for per-customer domain whitelist → return allow/deny → forward or block.
- **Business rules**:
  - Only HTTPS (port 443) traffic allowed.
  - Static whitelist for platform dependencies (AWS, Apsis services) never changes; per-customer dynamic lookup for CRM domains.
  - Dynamic whitelist stored in `allowed_domains` table: populated during integration installation.
  - Single-region deployment; multi-region Squid may require separate configurations.
- **Configuration**: 
  - Static files: `allowed_staging.txt`, `allowed_prod.txt` (baked into Docker image at build time)
  - Dynamic: `allowed_domains` database table (account/section/integration/domain mapping)
  - External script: Python/Bash/Go script that queries allowed_domains and returns allow/deny
- **Integration points**: Database (allowed_domains table lookup); Docker image build (static whitelist inclusion).
- **Gotchas**:
  - Adding new OAuth/external service domain requires updating allowed_*.txt and redeploying (no hot reload).
  - If dynamic script crashes, Squid returns 403 for all requests.
  - Database lookup latency adds to each request; ensure script performs efficient queries.
  - Testing Squid locally requires understanding Docker image build and Squid ACL syntax.

### Generic Connector
- **Purpose**: Unified REST API contract that external CRM systems must implement to integrate with Apsis via Membrane platform. Defines standard endpoints for authentication, schema discovery, contact CRUD, event webhooks, and consent management.
- **Key files**: `generic-api-spec.yaml` (OpenAPI specification); published to S3 bucket (`integration-files.apps.1`)
- **Tech stack**: OpenAPI 3.0 specification; language-agnostic (HTTP/REST)
- **Data flow**: Apsis (via Membrane) → Generic Connector endpoints (implemented by external CRM) → CRM business logic → Apsis/Membrane receives standardized responses.
- **Business rules**:
  - All CRM systems must return proper HTTP status codes (404 for not found, 200 for success, not 200 with error in body).
  - Field names in responses must use standard Apsis schema (email, first_name, last_name, etc.); CRM-specific names mapped at integration level.
  - Consent endpoints support bidirectional operations (read current state, write updates).
  - No versioning in endpoint paths; new features added as optional fields in request/response (loose schema validation).
  - Profile lists (segments, groups) returned as separate entity type with unique identifiers.
- **Configuration**: CRM system implementing connector must provide:
  - OAuth 2.0 endpoint (or API key endpoint)
  - Field schema endpoint
  - Contact list endpoint (paginated)
  - Consent/subscription list endpoint
  - Webhook registration endpoint (HTTP POST to register callback URL)
  - Contact create/update/delete endpoints (if outbound sync enabled)
- **Integration points**: Any external CRM system; Apsis (via Membrane).
- **Gotchas**:
  - FCC Enterprise 12.0 custom behavior: returns "unexpected error" for constraint violations, not specific error codes.
  - URL length limits for query parameters (~2000 chars); complex filters may require POST-based filtering or chunking.
  - Pagination cursor format varies by CRM; must be preserved exactly across requests.

### Keyspace Management
- **Purpose**: Logical namespaces within Apsis that isolate profile identifiers for each CRM integration. Each integration bootstraps its own keyspace (e.g., 'join_cx_keyspace', 'salesforce_keyspace') to prevent profile ID collisions.
- **Key files**: Keyspace configuration; discriminator generation logic
- **Tech stack**: Apsis profile database; keyspace ID generation (hash-based)
- **Data flow**: Integration installation → generate keyspace discriminator (hash of section_discriminator, first 8 chars) + logical name → store mapping in DB → full sync/delta sync route profiles to this keyspace.
- **Business rules**:
  - One keyspace per integration per section (in most cases).
  - Keyspace for main contacts (person entity): named by integration (e.g., 'fsc_enterprise_keyspace').
  - Separate keyspace for leads/silhouettes (e.g., 'fsc_enterprise_lead_keyspace') to prevent silhouette IDs from matching person IDs.
  - Profile merge (e.g., when lead becomes customer) requires explicit merge operation; never automatic.
  - Keyspace discriminator is deterministic (same section → same discriminator); never deleted on uninstall (persists for future reconnect).
  - Email keyspace is special: stores profiles by email address (unique key per email); does not support duplicates.
  - Consent state is NOT per-keyspace; stored globally (all keyspaces see same consent).
- **Configuration**: Keyspace discriminator format: `integration.keyspaces.<hash_first_8_chars>_<integration_logical_name>`.
- **Integration points**: Apsis profile storage; external CRM systems (their ID spaces).
- **Gotchas**:
  - If customer adds CRM integration using email as profile identifier but system uses CRM ID keyspace, profile lookups fail.
  - Manual DB changes to keyspace mapping revert on integration reinstall.
  - Merging profiles across keyspaces is manual and non-trivial; no UI for this yet (Solution 3 workaround required).

### Profile List Sync (Queries & Profiles)
- **Purpose**: Imports profiles matching CRM queries/segments as Apsis tags. Executes CRM native query, fetches matching contacts, applies tag to each profile. Supports recurring daily syncs.
- **Key files**: Profile List Sync Lambda (CloudWatch-triggered); worker ECS tasks
- **Tech stack**: AWS Lambda (trigger) + ECS tasks (worker); CRM query API
- **Data flow**: CloudWatch triggers daily (or manual request) → Lambda spawns worker → worker calls CRM query API → CRM executes query, returns contact list → worker fetches profiles for each contact ID (via keyspace lookup) → applies tag (auto-generated tag name from query name) → repeat daily.
- **Business rules**:
  - One worker per integration (no parallelization to avoid customer DDoS).
  - Tag name auto-generated from CRM segment/query name.
  - Recurring imports sync at fixed schedule (e.g., daily at 6:00 AM).
  - Tag application is idempotent (re-running import reapplies same tags).
- **Configuration**: CloudWatch event rule (daily 6:00 AM default); recurring flag per integration.
- **Integration points**: CRM query/segment API; Apsis tag system.
- **Gotchas**:
  - CRM query execution timeout may fail if query complex or dataset large.
  - No delta detection; full profile list re-imported each cycle (potential inefficiency).
  - Tag names may collide if query names match across systems; namespace isolation not enforced.

### Unified Data (HTTP/2 Streaming)
- **Purpose**: Fetches complex multi-table CRM queries (e.g., 'CEOs of companies whose staff attended event') paginated via HTTP/2; translates to Apache Columnar format; streams to Athena for joins with Apsis profile data. Used by email personalization engine.
- **Key files**: CRM query definitions; data provider connector; Athena join queries
- **Tech stack**: HTTP/2 streaming connection; Apache Columnar serialization; Athena (data warehouse)
- **Data flow**: Email tool requests CRM data with complex query → Integrations calls CRM via Unified Data endpoint → CRM returns paginated results via HTTP/2 stream → translates to Columnar format → streams to S3/Athena → email tool joins with Apsis profiles.
- **Business rules**:
  - Pre-made queries only (no ad-hoc query construction by users).
  - Pagination via HTTP/2 stream (avoids buffering entire result set in memory).
  - Columnar format enables efficient Athena joins with Apsis data.
- **Configuration**: Per-query definitions (in email tool or CRM).
- **Integration points**: Email tool (Maxo), CRM systems, Athena (data warehouse).
- **Gotchas**:
  - HTTP/2 streaming must be maintained throughout entire query execution; connection breaks lose partial data.
  - Columnar format transformation may be lossy for certain data types (complex objects).
  - Athena join performance depends on data volume and partition pruning; monitor query costs.

### Form Submission Handler
- **Purpose**: Processes form submissions, maps form field data to CRM attributes via outbound mappings, sends to CRM system, handles CRM response (new contact ID or existing match).
- **Key files**: Form submission processing code; merge worker integration
- **Tech stack**: HTTP endpoint (receives form POST); SQS/message queue; CRM connector libraries
- **Data flow**: Form submission (POST) → extract profile key + fields → apply outbound mapping (Apsis attribute → CRM field) → construct payload → Outbound Manager sends to CRM → CRM response (new contact ID, existing match, or no match) → Merge Worker merges keyspaces if required.
- **Business rules**:
  - Form submission requires CRM ID OR (email AND phone) for valid submission (secondary identification).
  - Submission creates profile in form/email keyspace initially.
  - CRM response determines entity type (contact, lead, silhouette); merge worker routes to correct keyspace.
  - Outbound mappings optional; if not configured, form fields not sent to CRM (metadata only).
- **Configuration**: Outbound mapping configuration (form field → CRM attribute).
- **Integration points**: CRM system (via Outbound Manager); Merge Worker; Apsis profile store.
- **Gotchas**:
  - Misconfigured outbound mapping (field references non-existent CRM attribute) causes CRM database error; blocks all forms of same type.
  - If form prefills from email keyspace but CRM returns contact ID, merge must occur to link profiles; without merge, future CRM updates via contact ID don't update form-originated profile.
  - Lead to contact conversion (when lead becomes customer) is manual merge operation; no automatic promotion.

### Consent Management & Bidirectional Sync
- **Purpose**: Synchronizes opt-in/opt