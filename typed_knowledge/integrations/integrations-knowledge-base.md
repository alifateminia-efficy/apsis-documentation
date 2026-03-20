---
title: Integrations Domain Knowledge Base
generated: 2026-03-20T08:32:14.220Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (4 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Integrations Domain Knowledge Base](#integrations-domain-knowledge-base)
  - [1. Domain Overview](#1-domain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Integration Manager](#integration-manager)
    - [Mappings Manager](#mappings-manager)
    - [Delta Sync Worker](#delta-sync-worker)
    - [Full Sync Manager](#full-sync-manager)
    - [Full Sync Producer](#full-sync-producer)
    - [Full Sync Consumer](#full-sync-consumer)
    - [All Sub Queue](#all-sub-queue)
    - [All Sub Worker](#all-sub-worker)
    - [Batch Production Worker](#batch-production-worker)
    - [Outbound Worker](#outbound-worker)
    - [Retry Driver Lambda](#retry-driver-lambda)
    - [Outbound Manager](#outbound-manager)
    - [Profile List Sync Consumer](#profile-list-sync-consumer)
    - [Integrations Backend](#integrations-backend)
    - [Audience (Profile Store)](#audience-profile-store)
    - [Athena (Query Engine)](#athena-query-engine)
    - [Connector Library (Generic Connector)](#connector-library-generic-connector)
    - [Clear Integration Endpoint (Uninstallation)](#clear-integration-endpoint-uninstallation)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling & Resilience](#error-handling-resilience)
    - [Observability & Logging](#observability-logging)
    - [Rate Limiting & Backpressure](#rate-limiting-backpressure)
    - [Data Consistency & Eventual Consistency](#data-consistency-eventual-consistency)
    - [Deployment & Operations](#deployment-operations)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Inbound Sync Rules](#inbound-sync-rules)
    - [Outbound Sync Rules](#outbound-sync-rules)
    - [Integration Lifecycle Rules](#integration-lifecycle-rules)
    - [Feature Availability Rules](#feature-availability-rules)
    - [Data Governance Rules](#data-governance-rules)
    - [Scheduling Rules](#scheduling-rules)
    - [Scale Limitations & Gotchas](#scale-limitations-gotchas)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence (Verified & Consistent)](#high-confidence-verified-consistent)
    - [Medium Confidence (Partially Verified, Some Gaps)](#medium-confidence-partially-verified-some-gaps)
    - [Low Confidence / Gaps ❌ (Unresolved Questions)](#low-confidence-gaps-unresolved-questions)
    - [Conflicts Resolved](#conflicts-resolved)
    - [Contradictions / Uncertainties](#contradictions-uncertainties)
    - [Data Quality Issues](#data-quality-issues)
  - [9. Recommended Next Steps for Knowledge Base Maintenance](#9-recommended-next-steps-for-knowledge-base-maintenance)

---

# Integrations Domain Knowledge Base

## 1. Domain Overview

The Integrations domain (codebase name: "Justin") is the middleware layer connecting APSIS One's customer data platform with external CRM, marketing, and eCommerce systems. It manages bidirectional synchronization of contact profiles, consent/subscription status, and activity events. The domain spans inbound (real-time webhooks + bulk imports) and outbound (activity delivery + consent sync) pipelines, handling data translation, field mapping validation, eventual consistency during bulk operations, and retry logic for failed transmissions. Boundaries: Integrations owns connector-to-Audience data flows and mapping configuration; Audience owns profile storage; external systems own their data; the domain does NOT own end-user campaign tools or CMS integrations (separate legacy systems).

---

## 2. Architecture Map

```
EXTERNAL SYSTEMS (CRM/Loyalty/eCommerce)
    ↓ (webhooks + API calls)
CONNECTOR LIBRARIES (system-specific adapters)
    ↓
JUSTIN CORE LAYER
├─ Integration Manager (schema fetch, install)
├─ Mappings Manager (field/subscription validation)
├─ Delta Sync Worker (real-time inbound)
├─ Full Sync Producer/Consumer (bulk import)
├─ All Sub Queue + All Sub Worker (event routing)
├─ Batch Production Worker (outbound batching)
└─ Outbound Worker (send to external)
    ↓
AWS INFRASTRUCTURE
├─ SQS (buffering, FIFO ordering)
├─ Kafka (partitioned event streaming)
├─ CloudWatch Events (scheduling)
├─ Secrets Manager (credentials)
└─ ECS/Lambda (task execution)
    ↓
AUDIENCE (profile store) ↔ ATHENA (query engine)
    ↓ (activity events)
    ↓
OUTBOUND SYNC PIPELINE
    ↓
EXTERNAL SYSTEMS (activity sink)

CROSS-CUTTING: React (real-time profile), Integrations Backend (API layer)
```

---

## 3. Module Reference

### Integration Manager
- **Purpose**: Orchestrates connector library invocation during installation to discover external system schema; supports both long-running operations (schema fetch) and quick queries.
- **Key files**: Originally Lambda, now ECS task; connector library invocation logic
- **Tech stack**: ECS (long-running), connector libraries (partner-implemented), HTTP/2 for streaming
- **Data flow**: User requests schema → Manager invokes connector library → Fetches schema from external system → Returns field list to UI
- **Business rules**: 
  - Must invoke correct connector library for system type
  - Schema discovery must account for on-premise systems with different versions (feature flag checking on customer instance required)
  - Migration from Lambda to ECS occurred for operational reasons
- **Configuration**: Connector library system identifier, timeout thresholds for schema fetch
- **Integration points**: Connector libraries, Integrations Backend (API), External systems (schema API)
- **Gotchas**: 
  - Transition from Lambda to ECS task may have performance implications
  - Schema discovery timeouts can block installation; implement backoff and logging
  - On-premise systems may have custom versions not exposed through standard schema endpoints
- **Tribal knowledge**: Originally Lambda but switched to ECS task for operational reasons (cost or performance); if reverting, understand the history

### Mappings Manager
- **Purpose**: Validates field and subscription mappings; enforces "Justin's Laws" (no mapping loops); caches mappings for runtime use; provides CRUD API for integration backend
- **Key files**: Dedicated database with user/table-level access control; validation engine
- **Tech stack**: SQL database, rule validation engine
- **Data flow**: User saves mappings → Manager validates (no loops, Justin's Laws) → Stores in database → Triggers cache invalidation in Delta Sync Worker on update
- **Business rules**:
  - Mapping cannot create loops
  - Field mappings must enforce Justin's Laws (exact rules beyond "no loops" not fully documented)
  - Subscription mappings support both native consent resources and "virtual consent" (legacy: mapped to contact fields)
  - Sync conditions (e.g., "only sync active") are applied at message processing time (not stored in Mappings Manager)
  - Consent is bidirectional; attribute changes are NOT synced outbound
- **Configuration**: Mapping rules, Justin's Laws constraints
- **Integration points**: Delta Sync Worker (reads cached mappings), Full Sync Consumer (reads mappings), Integrations Backend (CRUD API), Audience (subscription topics)
- **Gotchas**:
  - Cache staleness in Delta Sync Worker can diverge from authoritative mappings if invalidation fails
  - Virtual consent (legacy) adds complexity; newer integrations should use native consent resources
  - Exact list of Justin's Laws not documented in transcript
- **Tribal knowledge**: Cache consistency is critical; monitor for divergence between Mappings Manager and Delta Sync Worker behavior

### Delta Sync Worker
- **Purpose**: Consumes real-time webhook updates from external systems; applies field and subscription mappings; sends profiles to Audience in real-time
- **Key files**: ECS task (long-running), SQS FIFO queue configuration
- **Tech stack**: ECS, SQS FIFO (message group by CRM ID), connector library transformations
- **Data flow**: External system webhook → SQS FIFO queue (grouped by CRM ID) → Delta Sync Worker polls → Queries Mappings Manager (cached) → Applies mappings → Sends to Audience
- **Business rules**:
  - Messages processed in order per CRM ID (FIFO queue partitioning)
  - Sync conditions applied at processing time; profiles violating conditions deleted from Audience (feature flag controlled)
  - During full sync, visibility timeout delays messages until sync completes (prevents out-of-order processing)
  - Only processes messages if integration is installed and active
- **Configuration**: SQS FIFO queue URL, visibility timeout value (set dynamically by All Sub Worker), mappings cache location
- **Integration points**: External system webhooks, SQS FIFO queue, Mappings Manager (reads), Audience (profile sink), All Sub Worker (coordination)
- **Gotchas**:
  - SQS FIFO retention is limited (14 days); messages beyond retention are lost
  - If visibility timeout expires before full sync completes, messages become visible and may be processed out-of-order
  - Stale mappings cache can cause divergence from Mappings Manager
  - External systems may batch multiple webhook events; ensure connector library unpacks correctly
- **Tribal knowledge**: The visibility timeout + buffering strategy is a workaround the team "wishes it didn't need"; ideal solution would pause delta processing during full sync, but historical constraints prevented it

### Full Sync Manager
- **Purpose**: Orchestrates bulk contact import from external systems; manages eventual consistency by buffering delta messages during sync; spins up infrastructure on-demand
- **Key files**: ECS task, Full Sync Producer, Full Sync Consumer coordination
- **Tech stack**: ECS, SQS (Full Sync Queue), All Sub Queue (buffering), Audience ingestion API
- **Data flow**: User triggers full sync → Manager validates no sync in progress → Producer fetches paginated contacts → Queue buffers → Consumer applies mappings → Audience ingests → Manager waits for ingestion delay → Marks successful; meanwhile All Sub Worker buffers delta messages using visibility timeout
- **Business rules**:
  - Only one full sync per integration at a time
  - Full sync must complete and wait for ingestion delay before allowing delta messages to resume normal processing
  - Sync conditions applied at consumer processing time (after download)
  - All delta messages arriving during full sync are buffered with SQS visibility timeout
  - Without this buffering, delta messages could arrive before full sync completes, violating eventual consistency
- **Configuration**: Full Sync Queue URL, ingestion delay estimate (from Audience team), visibility timeout calculation
- **Integration points**: Full Sync Producer (fetch), Full Sync Consumer (process), All Sub Queue/Worker (buffering coordination), Audience (ingestion)
- **Gotchas**:
  - Full sync can take hours or days for large contact bases (millions of records); do not expect immediate completion
  - Buffering strategy can break if visibility timeout is miscalculated; buffer messages may exceed SQS retention (14 days) if sync lasts >14 days
  - Producer pagination errors (external system timeout) can stall entire sync; implement exponential backoff and logging
  - External system API rate limits may constrain producer fetch speed
  - Ingestion delay (eventual consistency window) is non-zero; Full Sync Manager must wait for it
  - If full sync fails and state stuck in "Downloading" or "Finalizing", recovery logic needed
  - Full sync at 8M contact scale took 1.5 days without server-side filtering; with sync conditions applied on Apsis side, filtering happens AFTER download (inefficient)
- **Tribal knowledge**: The team is exploring server-side sync condition filtering (CRM-side) to avoid downloading entire dataset when only subset matches conditions; Site Shop demonstrates this works (returns 1 contact instead of 1M)

### Full Sync Producer
- **Purpose**: Fetches paginated contact lists from external system using connector library; translates to Justin internal format; handles external system timeouts and rate limiting
- **Key files**: ECS task, connector library (system-specific pagination)
- **Tech stack**: ECS, connector library, HTTP clients with backoff
- **Data flow**: External system paginated API (via connector library) → Each page translated to internal format → Placed on Full Sync Queue
- **Business rules**:
  - Must handle external system pagination limits and rate limits
  - Each batch translated to internal format before queuing
  - Marks completion when all pages fetched (Consumer watches for completion signal)
- **Configuration**: Pagination batch size (tuned per CRM), timeout thresholds, retry/backoff parameters
- **Integration points**: Connector library (external fetch), Full Sync Queue (message sink), Full Sync Manager (orchestration)
- **Gotchas**:
  - External system pagination limits may be tight; batch size reduction may not help if endpoint is fundamentally slow
  - Profile pagination succeeds reliably; consent pagination fails at scale (FSC Enterprise: 22 seconds per 500-consent batch, linearly degrading)
  - For 8M contacts with 8M+ consents, consent fetching would take 98+ hours with serial retrieval
  - Timeout errors at CRM proxy level (HTTP 504) are external failures, not APSIS code issues
  - Public internet access required for on-premise deployments; VPN tunnels not supported
- **Tribal knowledge**: Contact pagination works at scale; consent pagination is the bottleneck and requires CRM vendor optimization or server-side filtering to reduce dataset

### Full Sync Consumer
- **Purpose**: Applies field/subscription mappings to paginated contacts; sends to Audience; handles sync condition filtering; optimizes consent comparison
- **Key files**: ECS task or Java service, Full Sync Queue message consumer
- **Tech stack**: Java/ECS, SQS message polling, in-memory consent export comparison
- **Data flow**: Full Sync Queue → Retrieve mappings from Mappings Manager → Apply mappings → Apply sync conditions (APSIS-side filtering) → Retrieve consent data → Compare against existing consent export (in-memory) → Send differing updates to Audience
- **Business rules**:
  - Applies same mapping logic as Delta Sync Worker
  - Applies sync conditions AFTER data download (architectural limitation for enterprise scale)
  - Consent comparison loads entire existing export into memory to avoid redundant updates
  - Consent is synced bidirectionally; attribute changes are NOT synced outbound
- **Configuration**: Mappings cache location, consent comparison flag (recommended disabled for enterprise customers), SQS queue polling parameters
- **Integration points**: Full Sync Queue (message source), Mappings Manager (reads), Audience (profile/consent sink), Integration Report Store (logging)
- **Gotchas**:
  - Sync condition filtering on Apsis side means download entire dataset first, then filter (inefficient at scale)
  - Memory-intensive consent comparison can exhaust heap if existing consent dataset is large; recommended to disable for enterprise customers
  - Consent comparison optimization trades off redundancy against stability; disabling it sends all consents regardless of changes
  - No exponential backoff documented for Audience sink errors
- **Tribal knowledge**: The team wants to move sync condition filtering to CRM side to avoid downloading filtered-out records; this requires CRM vendor API implementation (endpoint: `/api/v1/sync-conditions/{section}`)

### All Sub Queue
- **Purpose**: Central event queue for all subscription/activity events from Audience; buffers delta sync messages during full sync; routes outbound events
- **Key files**: SQS FIFO queue (partitioned by integration)
- **Tech stack**: AWS SQS FIFO
- **Data flow**: Audience emits events (subscription changes, activity events) → All Sub Queue → All Sub Worker validates and routes → Either buffers (if full sync ongoing) or routes to Kafka (for outbound)
- **Business rules**:
  - Messages partitioned by integration to maintain per-integration order
  - All events from Audience flow through this queue (not just subscriptions; name is legacy)
  - Buffering uses SQS visibility timeout to delay messages during full sync
- **Configuration**: Queue URL, FIFO configuration, retention policy (default 14 days)
- **Integration points**: Audience (event source), All Sub Worker (event consumer)
- **Gotchas**:
  - SQS message retention is 14 days; messages beyond retention are lost (impacts long-running full syncs with buffering)
  - Buffering strategy depends on visibility timeout being set correctly by All Sub Worker
- **Tribal knowledge**: Named "All Sub Queue" but handles all subscription AND activity events; name reflects original design, now has broader scope

### All Sub Worker
- **Purpose**: Validates events; checks if full sync is ongoing; buffers delta messages using SQS visibility timeout; routes outbound events to Kafka; enforces activity sync eligibility
- **Key files**: ECS task (long-running)
- **Tech stack**: ECS, SQS, Kafka, Full Sync Manager coordination
- **Data flow**: All Sub Queue → Check: integration installed + should activity sync? → If full sync ongoing: set visibility timeout and return message to queue (buffer); If outbound: route to Kafka partition
- **Business rules**:
  - Integration must be installed and configured for the activity type (email, SMS, etc.)
  - If full sync is in progress, delay message with visibility timeout (depends on Full Sync Manager state)
  - Outbound events routed to Kafka partitioned by integration
  - Consent changes bypass batching (sent one-at-a-time with low latency)
  - Only opt-out consent is redriven from dead letter queue; opt-in is NOT redriven (prevent consent inconsistency)
- **Configuration**: Full Sync Manager state query endpoint, Kafka partition routing, visibility timeout parameters
- **Integration points**: All Sub Queue (source), Full Sync Manager (sync status query), Kafka (outbound routing), Batch Production Worker (downstream)
- **Gotchas**:
  - Check "is full sync ongoing?" must return correct state; if check fails or is skipped, delta messages process out-of-order
  - Visibility timeout must be set dynamically based on Full Sync Manager's estimated sync duration + ingestion delay
  - Kafka partitioning is "unordered globally, partitioned by integration"; events from different integrations may interleave
  - Consent opt-in is intentionally NOT redriven from dead letter queue; only opt-out is redriven (contractual requirement)
- **Tribal knowledge**: The dual-role (buffering + routing) is core to eventual consistency architecture; visibility timeout is the key mechanism

### Batch Production Worker
- **Purpose**: Polls Kafka partition; groups messages by integration; creates batches when size reaches 200K messages or polling time exceeds 1 second; prevents DDOSing customer systems
- **Key files**: ECS task
- **Tech stack**: ECS, Kafka consumer, SQS FIFO producer
- **Data flow**: Kafka partition (unordered globally, partitioned by integration) → Group messages by integration → Batch (size: 200K or timeout: 1s) → Place on SQS FIFO queue
- **Business rules**:
  - Batch size hardcoded to 200K messages (stays safely below 256K SQS message limit)
  - Polling timeout hardcoded to 1 second (balances latency vs. batch efficiency)
  - Batches maintain per-integration order
  - If external system cannot accept 200K batch, integration will fail and messages enter dead letter queue
- **Configuration**: Kafka partition (from All Sub Worker), SQS FIFO queue URL, batch size (200K, not configurable), polling timeout (1s, not configurable)
- **Integration points**: Kafka partition (source), SQS FIFO queue (sink), Outbound Worker (downstream)
- **Gotchas**:
  - Batch size is hardcoded and cannot be overridden by connector libraries
  - If external system cannot accept large batches, integration breaks; connector library cannot split batches
  - Hardcoded limits prevent per-integration tuning for systems with different capacity
- **Tribal knowledge**: Batching is essential to prevent DDOSing customers; sending 1M events one-by-one would trigger 1M API calls. Kafka partitioning ensures per-integration order while enabling batching.

### Outbound Worker
- **Purpose**: Consumes batched messages from SQS FIFO queue; uses connector library to send to external system; handles retries and dead letter queue routing; tracks batch IDs for idempotency
- **Key files**: ECS task
- **Tech stack**: ECS, SQS FIFO queue, connector library (send), dead letter queue
- **Data flow**: SQS FIFO batch → Connector library send to external system → On success: mark complete; On failure after N retries: route to dead letter queue
- **Business rules**:
  - External systems must implement idempotency via batch ID tracking (during redrive, batches lose eventual consistency)
  - Retry count before dead letter queue placement not specified in transcript (❌ unresolved)
  - Opt-in messages are NOT redriven from dead letter queue (consent consistency requirement)
  - All other message types retry indefinitely via daily Retry Driver Lambda
- **Configuration**: SQS FIFO queue URL, dead letter queue URL, max retry count (❌ not specified), connector library endpoint
- **Integration points**: SQS FIFO queue (source), Batch Production Worker (upstream), Connector library (external send), Dead letter queue (failure sink), External systems (activity sink)
- **Gotchas**:
  - External system feature availability must be checked both in APSIS code and on customer instance (on-premise systems may have version variations)
  - Dead letter queue messages lose eventual consistency during redrive; external system MUST track batch IDs
  - Batch size cannot be split; if 200K batch fails, entire batch enters dead letter queue
  - Timeouts at external system proxy level (HTTP 504) will cause batch failures
  - Opt-in message failure is FINAL (not redriven); losing opt-in is acceptable business cost vs. consent inconsistency risk
- **Tribal knowledge**: External systems must implement idempotency; this is a contractual requirement enforced via partner agreements and supplier SLAs

### Retry Driver Lambda
- **Purpose**: Runs daily at 6:00 AM UTC; re-queues all messages from dead letter queue back to SQS FIFO queue for indefinite retry
- **Key files**: Lambda function, CloudWatch scheduled event
- **Tech stack**: Lambda, CloudWatch Events, SQS
- **Data flow**: CloudWatch trigger (6:00 AM) → Retry Driver Lambda → Query dead letter queue → Re-queue all messages to SQS FIFO queue → Outbound Worker retries
- **Business rules**:
  - Runs daily at fixed 6:00 AM UTC (not configurable)
  - Re-queues ALL messages from dead letter queue (no selective retry)
  - Opt-in messages are NOT redriven (caught at Outbound Worker level)
  - No configurable retry limit; retries continue indefinitely
- **Configuration**: CloudWatch event trigger (6:00 AM), SQS dead letter queue URL, SQS FIFO queue URL (re-queue destination)
- **Integration points**: Dead letter queue (source), SQS FIFO queue (destination), CloudWatch Events (scheduling), Outbound Worker (downstream processing)
- **Gotchas**:
  - Fixed 6:00 AM schedule may not align with customer business hours or CRM maintenance windows
  - If thousands of messages in dead letter queue, re-queue can be slow (no parallelism documented)
  - SQS message retention is 14 days; messages in dead letter queue beyond 14 days are lost (but daily re-queue keeps extending retention)
  - Re-driving indefinitely can mask systemic issues (e.g., external system bug); requires monitoring to detect patterns
  - No observability into re-queue success; if Lambda fails, messages remain in dead letter queue
- **Tribal knowledge**: Dead letter queue monitoring is critical; team holds Monday operational meetings to catch patterns early and escalate to external partners

### Outbound Manager
- **Purpose**: Bridge between campaign editor front-end and external systems; initiates outbound activity sync (emails, SMS, forms, etc.); queries Integrations Backend to check integration capabilities
- **Key files**: REST API layer
- **Tech stack**: REST API, SQS (All Sub Queue)
- **Data flow**: User confirms "Sync to CRM" in campaign editor → Outbound Manager → Message to All Sub Queue → Outbound processing pipeline
- **Business rules**:
  - Campaign editor queries Integrations Backend to determine which integrations can receive specific activity types (emails, SMS)
  - Outbound field mappings apply GLOBALLY to all form submissions for an integration (not per-form) — historical limitation
- **Configuration**: All Sub Queue URL, Integrations Backend query endpoint
- **Integration points**: Campaign editor (front-end), All Sub Queue (activity message routing), Integrations Backend (capability query)
- **Gotchas**:
  - Field mappings are global per integration, not per-form; customers wanting form-specific mappings have no option (historical limitation, CRM teams have complained but not prioritized)
  - Capability query must account for feature flags (both APSIS support and customer instance support)
- **Tribal knowledge**: The global field mapping limitation is a historical pain point; addressed via architectural redesign but implementation deferred

### Profile List Sync Consumer
- **Purpose**: Imports contact lists from external systems; adds tags to existing profiles; scheduled via CloudWatch events at 6:00 AM fixed time; processes sequentially per integration
- **Key files**: ECS task, CloudWatch event rule
- **Tech stack**: ECS, CloudWatch Events, Connector library, Audience tagging API
- **Data flow**: CloudWatch trigger (6:00 AM) → Profile List Sync Consumer → Fetch list from external system → Fetch all profiles on list → Batch tag requests to Audience
- **Business rules**:
  - Profile lists do NOT create new profiles; they only add tags to existing profiles (existing profiles must come from full sync or delta sync)
  - Runs once daily at 6:00 AM UTC (not user-configurable)
  - Processes sequentially per integration (no parallel workers for same integration to avoid DDOSing customer)
  - If one import fails, subsequent imports queue and may not complete before next 6:00 AM trigger
  - Sync conditions are NOT applied to profile list imports (only to inbound sync)
- **Configuration**: CloudWatch rule (6:00 AM UTC), profile list API endpoint (via connector library), batch tagging parameters
- **Integration points**: Connector library (external list fetch), Audience (tag add requests), CloudWatch Events (scheduling)
- **Gotchas**:
  - Profile list import does NOT create new profiles; confused customers assume it does
  - Fixed 6:00 AM schedule with sequential processing means if many integrations have imports, later ones may not complete before next trigger
  - List definitions may be cached in external system; membership changes may lag hours
  - Dynamic lists may return different results each import; tag churn can be high
  - Static lists may have human errors (inactive contacts); no filtering applied
  - If external system API fails during fetch, entire import fails and is retried next day
- **Tribal knowledge**: Only tags existing profiles; this is fundamentally different from contact sync which creates profiles

### Integrations Backend
- **Purpose**: REST API layer providing integration management capabilities; queries which integrations are installed; checks if integrations can receive specific activity types (emails, SMS); returns feature availability
- **Key files**: REST API endpoints
- **Tech stack**: REST API (language not specified), database queries
- **Data flow**: Campaign editor queries endpoint → Backend checks installed integrations + feature flags → Returns list of integrations capable of receiving activity type
- **Business rules**:
  - Must check both in-code feature flag (what this CRM type supports) and customer instance capability (version/configuration on customer's system)
  - Missing either check layer leads to bugs
- **Configuration**: Feature flag database, integration database
- **Integration points**: Campaign editor (front-end query), Integration Manager (lookup), Integrations (feature lookup)
- **Gotchas**:
  - Two-layer feature flag checking required; missing one layer causes bugs
  - On-premise systems may have version variations not exposed through standard queries
- **Tribal knowledge**: Feature availability checking is subtle; must verify both centralized config and customer instance state

### Audience (Profile Store)
- **Purpose**: Central profile store receiving synced data from external systems; source of events for outbound sync; provides profile storage, subscription management, ingestion delay
- **Key files**: Profile database, event emission system
- **Tech stack**: Profile database, event streaming
- **Data flow**: Inbound sync → Profile storage + subscription updates → Event emission → Outbound sync; Athena joins for unified data queries
- **Business rules**:
  - Ingestion has eventual consistency delay (exact value not specified; Full Sync Manager must wait for it)
  - Subscription topics map to external system consent fields (via subscription mappings)
  - Events emitted on profile changes (triggering outbound sync)
- **Configuration**: Profile schema, subscription topic definitions, ingestion delay estimate
- **Integration points**: Delta Sync Worker (inbound), Full Sync Consumer (bulk inbound), All Sub Queue (event source), Athena (query join)
- **Gotchas**:
  - Ingestion delay is non-zero; Full Sync Manager must account for it
  - Subscription mapping must handle virtual consent (legacy) and native resources
- **Tribal knowledge**: Ingestion delay value is held by Audience team; Integrations team must coordinate on value for Full Sync Manager timeout calculations

### Athena (Query Engine)
- **Purpose**: Query engine for unified data scenarios; joins streamed external CRM data with profile exports; supports complex multi-table queries for personalization (e.g., reach CEOs of companies whose employees attended events)
- **Key files**: Query execution engine
- **Tech stack**: Query engine, HTTP/2 streaming
- **Data flow**: Email tool requests unified data via HTTP/2 stream → Integrations fetches paginated CRM query results → On-the-fly Apache Columnar translation → Streams to Audience → Athena joins with profile exports → Returns joined result set → Email tool resolves personalization variables
- **Business rules**:
  - Handles millions of rows via streaming (avoids buffering entire dataset in memory)
  - Pre-made queries must be defined in external CRM (user responsibility)
  - Connector library must support paginated query fetching
- **Configuration**: HTTP/2 endpoint, query discovery mechanism
- **Integration points**: Email tool (query request), Integrations (CRM stream source), Audience (profile export)
- **Gotchas**:
  - HTTP/2 streaming means connection must stay open; disconnect = stream ends (no resume capability)
  - Columnar translation must be correct; errors corrupt streamed data
  - Limited production validation (only one customer as of session date); treat as production-capable but unproven at scale
  - Join performance depends on complexity and data size
- **Tribal knowledge**: Unified Data is production-ready but lightly tested; deploy to new customers cautiously with close monitoring for HTTP/2 errors and join performance issues

### Connector Library (Generic Connector)
- **Purpose**: System-specific implementation of standardized API contract; handles fetch, send, schema discovery, pagination, idempotency for external systems; partner-implemented following published spec
- **Key files**: Partner-implemented files; spec location: S3 bucket (to be confirmed after session)
- **Tech stack**: Partner language/framework; must implement REST API contract
- **Data flow**: All inbound and outbound data flows through connector for external system translation
- **Business rules**:
  - Must implement idempotency via batch ID tracking for outbound operations
  - Must handle pagination for contacts and consents (separate optimization strategies may be needed)
  - Must support feature discovery (what capabilities does this connector have)
  - Must handle sync conditions (future: register on CRM side via `/api/v1/sync-conditions/{section}`)
  - Must implement both full sync (paginated fetch) and incremental sync (webhook handling)
- **Configuration**: External system credentials (from customer), authentication method (system-specific), pagination batch size
- **Integration points**: Integration Manager (schema fetch), Delta Sync Worker (inbound translation), Full Sync Producer (paginated fetch), Full Sync Consumer (mapping application), Outbound Worker (send), Profile List Sync Consumer (list import), Unified Data streaming (paginated CRM query fetch)
- **Gotchas**:
  - Partner must implement and maintain; quality and performance depend on partner's engineering resources
  - Feature availability checking on customer instance is partner's responsibility
  - Idempotency is partner's responsibility; APSIS cannot guarantee consistency without it
  - External system API rate limits and timeouts are managed by partner
  - Contact pagination succeeds at scale; consent pagination is often the bottleneck (partner must optimize)
  - On-premise systems may have custom configurations not exposed through standard APIs
- **Tribal knowledge**: Partner model is the future; supplier model (e.g., CRM Consultana for Dynamics) was slow and expensive. Sideshop partnership for Dynamics exists but commercial execution lagging. Intermail brings own customers (revenue-sharing). Generic Connector API Spec is published and versioned; partners implement to this spec.

### Clear Integration Endpoint (Uninstallation)
- **Purpose**: Forcefully remove customer integration from APSIS database when external CRM is unreachable or misconfigured, using `ignore_external_system_errors` flag to bypass CRM communication failures
- **Key files**: REST API endpoint, uninstallation function (internal)
- **Tech stack**: REST API, AWS Secrets Manager (for authorization key)
- **Data flow**: Request (Account ID + Section ID + Integration ID + Delete Integration Secret Key) → Load customer credentials → Call uninstall with ignore_external_system_errors=true → Delete database entries → Delete stored credentials → Delete Squid Proxy entries → Return status
- **Business rules**:
  - Force deletion is server-side only, not exposed as self-service UI button (risk mitigation)
  - Customer must manually remove webhook configurations from their CRM after forced deletion
  - Internal Delete Integration Secret Key used (not delegation keys)
  - ignore_external_system_errors flag allows deletion to proceed despite CRM communication failures
- **Configuration**: Delete Integration Secret Key (AWS Secrets Manager), Account ID (legacy format or UUID), Section ID, Integration ID
- **Integration points**: AWS Secrets Manager (key), Integration Manager (logging), Squid Proxy (cleanup), Customer CRM (webhook removal — manual)
- **Gotchas**:
  - 404 errors are silent; endpoint doesn't indicate which parameter (Account ID/Section ID/Integration ID) is wrong
  - Account ID typos (e.g., 'Meltex' vs 'Maltex') produce 401 Unauthorized from CRM rather than validation error
  - Squid Proxy database delete may hang, but ignore_external_system_errors allows overall deletion to complete
  - After forced deletion, customer's CRM continues sending webhooks (silently rejected); customer must be contacted to clean up
  - Delete Integration Secret Key is extremely sensitive (works across all integration types); guard like production database credentials
- **Tribal knowledge**: Forcing deletion without cleaning up external webhooks is a legal/compliance risk; customers must be explicitly told to remove webhook configs on their CRM side

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- **Inbound (CRM → APSIS)**: Connector library handles CRM authentication (API keys, OAuth, custom); APSIS stores customer credentials in Secrets Manager
- **Outbound (APSIS → CRM)**: Outbound Worker uses stored customer credentials via connector library
- **Internal**: Clear Integration endpoint uses Delete Integration Secret Key from Secrets Manager (future: may use delegation keys)
- **Webhook validation**: Implied but not detailed; webhook signature validation recommended

### Error Handling & Resilience
- **Inbound errors**: Sync conditions filter failures; profiles with condition violations deleted (or ignored, feature flag controlled); mapping errors logged
- **Full sync errors**: Producer pagination errors trigger exponential backoff; Consumer errors route to Integration Report Store; state may stick in "Downloading"/"Finalizing" without recovery logic
- **Outbound errors**: Retries with exponential backoff; after max retries (count not specified), route to dead letter queue; daily re-queue indefinitely (no limit)
- **Consent sync errors**: Opt-in failures final (not redriven); opt-out failures redriven daily
- **Feature flag errors**: Graceful degradation if feature flag fetch fails (implied)

### Observability & Logging
- **Integration Manager**: Logs all lifecycle events; can query by Account ID/Section ID/Integration ID
- **AltaSync Manager**: Monitors for orphaned webhooks (customer CRM sending data after integration deletion)
- **Integration Report Store**: Stores sync execution reports, logs, diagnostics
- **Dead letter queue monitoring**: Team holds Monday operational meetings to review failures and escalate to partners
- **Postman collections**: Reference documentation for connector endpoints and payloads; used for debugging
- **⚠️ PII protection**: Webhook data NOT logged to avoid exposing PII; only request presence logged

### Rate Limiting & Backpressure
- **Inbound batching**: SQS FIFO queue buffers delta messages (natural backpressure)
- **Full sync pagination**: Connector library implements exponential backoff for external system rate limits
- **Outbound batching**: Batch Production Worker creates 200K-message batches (prevents DDOSing customers)
- **Outbound retry**: Indefinite retry with daily re-queue (no backoff documented; may need tuning)
- **Profile list imports**: Sequential processing per integration (prevents parallelism)

### Data Consistency & Eventual Consistency
- **Delta sync**: Messages processed in order per CRM ID (SQS FIFO partitioning)
- **Full sync**: Visibility timeout buffering ensures delta messages arrive after full sync completes
- **Consent sync**: Bidirectional; attribute changes NOT synced outbound (historical customer preference)
- **Outbound batching**: Kafka partitioning per integration (global interleaving acceptable)
- **Dead letter queue redrive**: Loses eventual consistency; external systems must implement idempotency via batch ID tracking

### Deployment & Operations
- **Infrastructure**: ECS tasks (long-running), Lambda (scheduled), CloudWatch Events (cron-like scheduling at 6:00 AM for profile list imports and dead letter queue redrive)
- **Configuration**: Environment variables (queue URLs, endpoint URLs, feature flags), database schema (mappings, installations table), AWS Secrets Manager (credentials, authorization keys)
- **Monitoring alerts**: Dead letter queue depth, full sync duration, webhook orphaning detection (AltaSync Manager)
- **Support workflow**: Issues must be submitted via formal help channel (single source of truth); R&D avoids responding to Slack/email/direct messages

---

## 5. Business Rules Reference

### Inbound Sync Rules
- **Only one CRM integration per Audience section is allowed** (legacy limitation due to shared CRM ID field)
  - **Exception**: Post-2020 integrations use system-specific IDs (e.g., dynamics_id, salesforce_id) and support multiple CRM integrations per section
  - **Migration**: Data migration project identified for fall 2024 to migrate legacy integrations
- **Sync conditions (e.g., "only sync active contacts") are applied at message processing time**; profiles violating conditions are deleted from Audience
  - **Feature flag controlled**: Can be disabled if customers want to preserve profiles despite condition violations
  - **Future state**: Conditions will be registered on CRM side (server-side filtering) to avoid downloading filtered-out records
- **Inbound sync conditions are NOT applied to profile list imports** (only to delta sync and full sync)
- **Profile list imports do NOT create new profiles; they add tags to existing profiles** (critical distinction)
- **Mapping cannot create loops and must enforce Justin's Laws** (exact rules beyond "no loops" not fully specified)
- **External system feature availability must be checked both in APSIS code (what feature is supported) and on customer instance** (does this specific instance have this feature)
  - **Especially relevant**: On-premise systems with multiple versions may have version-specific capabilities

### Outbound Sync Rules
- **Consent is synced bidirectionally; attribute changes are NOT synced outbound**
  - **Context**: Historical customer preference (CRM systems considered the "data master")
  - **Never revisited**: Confirm assumption with stakeholders before changing
- **Consent filtering**: Opt-in messages are NOT redriven from dead letter queue; only opt-out messages are redriven
  - **Rationale**: Prevent consent inconsistency; losing opt-in is acceptable business cost
- **Opt-in message failure is FINAL** (not redriven; no retry path)
- **Outbound field mappings apply globally to all form submissions for an integration, not per-form** (historical limitation)
  - **Customer workaround**: Create separate integrations or manage fields manually post-send
  - **CRM teams complained but not prioritized**: Lower priority than other issues
- **Batch size is hardcoded to 200K messages** (stays safely below 256K SQS limit)
  - **If external system cannot accept 200K batch**: Integration will fail and messages accumulate in dead letter queue
  - **No workaround**: Connector library cannot split batches; requires code change or external system upgrade
- **Batch creation timeout hardcoded to 1 second** (balances latency vs. efficiency)
  - **Not tunable**: Per-integration customization not supported
- **Batching is essential** to prevent DDOSing customers (1M events one-by-one would be 1M API calls)
- **Retry limit for dead letter queue placement**: Not specified ❌ (unresolved)
- **External systems must implement idempotency via batch ID tracking**
  - **During dead letter queue redrive**: Batches lose eventual consistency; APSIS cannot guarantee consistency without external idempotency
  - **Contractual requirement**: Enforced via supplier/partner agreements

### Integration Lifecycle Rules
- **Force deletion via Clear Integration endpoint is only performed manually by support/engineering team**
  - **Not exposed** as self-service UI button to customers
  - **Risk mitigation**: If deletion fails on CRM side, webhooks remain uncleaned
  - **Legal liability unclear**: When customer-initiated force deletion leaves orphaned webhooks
- **After forced deletion, customer must manually remove webhook configurations from their CRM system**
  - **Hard requirement**: Forced deletion may succeed in APSIS but fail to remove webhooks on CRM side
  - **Orphaned webhooks are a legal/compliance risk**: Customer data continues flowing to APSIS endpoints that no longer exist
- **API key rotation in customer's CRM does not require integration uninstall**
  - **Customer should use**: Update API Key endpoint rather than uninstalling
  - **Only uninstall if**: Customer wants to remove integration entirely
- **Webhook orphaning is a legal/compliance risk** that must be communicated to customer post-deletion

### Feature Availability Rules
- **Feature flag implementation has two layers**:
  1. What does this CRM type support (in APSIS code)
  2. Does this customer's specific instance support it (external system version check)
  - **Missing either layer**: Bugs will result
- **Campaign editor queries Integrations Backend** to determine which integrations can receive specific activity types
  - **Returns**: List of installed integrations with email/SMS capability
  - **UI shows**: "Sync to CRM" tab only for integrations capable of receiving this activity type

### Data Governance Rules
- **On-premise deployments require public internet access** to CRM endpoints
  - **VPN tunnels not supported** in Integrations domain (no maintenance resources)
  - **VPN supported in other products** (e.g., Pro) but not Integrations
  - **Customer choice**: If public access unacceptable, customer may not use APSIS Integrations
- **Consent export comparison optimization is disabled for enterprise customers** with large existing consent datasets
  - **Reason**: In-memory export loading can exhaust heap for 8M+ records
  - **Trade-off**: Disabling sends redundant consent updates instead of optimizing away duplicates

### Scheduling Rules
- **Profile list imports scheduled at 6:00 AM UTC (fixed time)**
  - **Not user-configurable**: Limitation that customers have complained about
  - **Sequential processing per integration**: Prevents parallelism; later imports may not complete before next 6:00 AM trigger
- **Dead letter queue redrive scheduled at 6:00 AM UTC daily**
  - **Fixed schedule**: No flexibility
  - **Can be slow**: If thousands of messages, re-queue takes time; no parallelism documented
  - **Executes regardless**: Of previous day's redrive status (no idempotency check)
- **Full sync has no scheduled trigger**
  - **Customer-initiated or scheduled by customer**: Not driven by APSIS
  - **Sequential worker processing per integration**: Prevents DDOSing customer infrastructure

### Scale Limitations & Gotchas
- **Load testing tested only up to 2M contacts**; now dealing with 8M contacts (4x scale)
- **Profile pagination succeeds at scale; consent pagination fails at scale**
  - **Separate optimization strategies required** for each
  - **FSC Enterprise consent endpoint**: 22 seconds per 500 entries (linearly degrading); CRM vendor optimization needed
- **For 8M contacts with 8M+ consents: serial retrieval would take 98+ hours**
  - **Exceeds gateway timeout** by orders of magnitude
  - **Server-side filtering required** (CRM-side) to reduce dataset
- **SQS message retention is 14 days**; messages beyond retention lost
  - **Impact**: Full syncs > 14 days with buffering will lose delta messages
  - **Mitigation**: Shorter buffering window or server-side filtering to reduce sync duration

---

## 6. Known Issues & Workarounds

| Issue | Severity | Affected Component | Workaround |
|-------|----------|-------------------|-----------|
| **CRM ID limitation: only one CRM integration per Audience section** due to shared CRM ID field | High | Legacy integrations | Use separate Audience sections for multiple CRM systems; post-2020 integrations with system-specific IDs support multiple CRM integrations per section; data migration project identified for fall 2024 |
| **Infinite dead letter queue redrive without configurable limit** | Medium | Retry Driver Lambda, dead letter queue | Manually purge dead letter queue messages if recurrence not expected; monitor operational meetings for patterns |
| **Outbound field mappings not form-specific** (all form submissions use same mappings) | Low | Outbound Manager, form submission events | Create separate integrations or manage fields manually post-send; CRM teams complained but not prioritized |
| **Full sync eventual consistency buffering**: delta messages buffered with visibility timeout; if timeout expires before sync completes, messages processed out-of-order | Medium | All Sub Worker, full sync buffering | Visibility timeout set to estimated full sync duration + ingestion delay; monitor full sync duration and adjust if needed; team wishes this architecture wasn't necessary |
| **Batch size hardcoded to 200K** (if external system cannot accept, outbound sync fails) | Medium | Batch Production Worker, Outbound Worker | Contact external system partner to increase batch size capability; scale down (requires code change, not configurable); no alternative exists |
| **Profile list import sequential processing per integration** (no parallelism; later imports may not complete before next trigger) | Low | Profile List Sync Consumer, CloudWatch scheduling | Stagger import schedules or reduce number of list imports; currently 6:00 AM is fixed; no user-configurable scheduling available |
| **Opt-in messages NOT redriven from dead letter queue** (only opt-out redriven; if opt-in fails, it's final) | Medium | Retry Driver Lambda, consent sync | Manually re-queue opt-in messages from dead letter queue if needed; this is contractual requirement to prevent consent inconsistency |
| **Redrive consistency caveat**: batches lose eventual consistency during dead letter queue redrive; external systems must track batch IDs for idempotency | Medium | Retry Driver Lambda, Outbound Worker, external system | Document as contractual requirement in partner/supplier agreements; if external system doesn't implement idempotency, APSIS cannot guarantee consistency; monitor dead letter queue redrives for duplicates |
| **Unified Data: limited production validation** (only one customer as of session date) | Low | Unified Data HTTP/2 streaming, Athena joins, email tool personalization | Deploy cautiously to new customers; monitor closely for HTTP/2 streaming errors, Athena join performance issues, memory usage; have rollback plan ready |
| **Consent endpoint pagination timeout at scale** (8M contacts × multiple consents; 22 seconds per 500-consent batch) | High | FSC Enterprise Connector, Full Sync Consumer | Server-side sync condition filtering to reduce dataset before consent retrieval (requires CRM vendor implementation); interim: CRM pagination/caching optimization needed |
| **Sync condition filtering happens on APSIS side AFTER downloading all data** (not on CRM side before transmission) | High | Full Sync Consumer, Generic Connector, Sync Conditions Module | No workaround for customers requiring filtered syncs at scale; architectural limitation requires server-side implementation (future: `/api/v1/sync-conditions/{section}` endpoint on CRM) |
| **Memory-intensive consent export comparison can exhaust heap** for large existing consent datasets | High | Consent Processing Module, Full Sync Consumer | Remove or disable consent comparison optimization; trade redundancy for stability |
| **On-premise deployments require public internet access** (VPN tunnels not supported in Integrations domain) | High | Generic Connector, FSC Enterprise Connector | Customer must configure public-facing CRM endpoint; no alternative available; customer may choose not to use APSIS if public access unacceptable |
| **Issues reaching R&D through multiple channels** (help channel, Slack, email, direct messages) causing context-switching and invisible work | Medium | Support workflow, organizational process | Enforce single-channel workflow: all integration issues must be via formal help channel; redirect all other inquiries; requires cultural enforcement |
| **Load testing gap**: only tested up to 2M contacts; now dealing with 8M contacts (4x scale) | Medium | Full Sync Consumer, Quality Assurance | Load testing with 8M contact dataset required before claiming enterprise-scale support; exponential consent processing cost may exceed linear expectations |
| **Profile pagination succeeds; consent pagination fails** (different optimization strategies required) | High | FSC Enterprise Connector, CRM consent endpoint | Profile optimization already working; separate consent endpoint optimization required from CRM vendor |
| **CRM vendor consent endpoint has inefficient pagination or caching** (returns 504 timeouts for large batches) | High | CRM system (external), FSC Enterprise Connector | Reduce batch size (may not help if pagination is fundamentally slow); CRM vendor must optimize pagination/caching logic (long-term) |
| **Squid Proxy database delete hangs** during uninstallation | Medium | Squid Proxy, Uninstallation Function | Set ignore_external_system_errors flag to true; endpoint times out waiting for delete but overall deletion proceeds |
| **Clear Integration endpoint returns 404 with no diagnostic message** when Account ID/Section ID/Integration ID incorrect | Medium | Clear Integration Endpoint | Manually verify each parameter against Integration Manager; use exact account name/ID format from Integration Manager |
| **Account ID typos (e.g., 'Meltex' vs 'Maltex') result in 401 Unauthorized** error from CRM rather than validation error | Medium | Clear Integration Endpoint | Double-check Account ID spelling against Integration Manager; legacy account IDs are exact customer names and case-sensitive |
| **Orphaned webhooks**: customer's CRM continues sending webhook requests after forced deletion | High | Webhook Management, Integration Manager, Legal/Compliance | Contact customer proactively with manual instruction to remove webhook config from CRM side; monitor logs for webhook request attempts |
| **Webhook data not visible in logs** to avoid PII exposure; customer may not realize they're sending data post-deletion | Medium | Integration Manager, AltaSync Manager | Contact customer proactively with reminder to clean up webhooks; explain that requests are rejected but they should still remove config |
| **Generic Connector specification incomplete or hard to find** | Medium | Generic Connector API Spec (location TBD) | Spec location: S3 bucket (to be confirmed after session); team should publish stable URL |
| **Integration ID must match exact system identifier** (e.g., 'Lime', 'Dynamics365', 'FSC_Enterprise_2') | Medium | All connectors | Use exact ID from Integration Manager; no flexibility in naming |
| **Mappings cache staleness**: Delta Sync Worker cache can diverge from Mappings Manager if invalidation fails | Medium | Delta Sync Worker, Mappings Manager | Monitor for divergence between Mappings Manager and Delta Sync Worker behavior; implement cache consistency checks |
| **Full sync state can stick** in "Downloading" or "Finalizing" if process crashes | Medium | Full Sync Manager | Implement recovery logic to detect and clear stuck states; current recovery process not documented |
| **Feature flag checking has two layers** (APSIS code + customer instance); missing one layer causes bugs | Medium | Integrations Backend, Feature flag system | Always verify both layers when implementing feature detection |

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Justin** | Core integration middleware layer for APSIS One (named after Justin Timberlake: "the greatest integration system you'll ever hear on the radio"); provides generic infrastructure for integration patterns and connector libraries for system-specific logic |
| **Connector Library** | System-specific implementation of generic connector API spec; handles schema discovery, data fetch/send, translation to/from Justin internal format, and idempotency for external systems; partner-implemented following published spec |
| **Generic Connector API Spec** | Standardized contract that external system partners implement; defines fields, events, operations, and feature discovery for any CRM/system integrated via Justin; published and versioned |
| **CRM ID** | Shared field on Audience profiles used to identify external CRM records; legacy integrations depend on this field (limitation: only one CRM integration per Audience section) |
| **Delta Sync** | Real-time synchronization of incremental contact updates from external systems to Audience; triggered by webhooks; applies field mappings and subscription mappings |
| **Full Sync** | Bulk import of all existing contacts from external system to Audience; runs once during integration setup (optionally re-run); includes handling of eventual consistency with delta messages during import |
| **Sync Condition** | Conditional logic applied during message processing (e.g., "only sync active contacts"); if violated, message ignored (or profile deleted if feature flag enabled); currently applied on APSIS side AFTER download (future: CRM-side) |
| **Field Mapping** | Mapping between external system contact fields and Audience profile attributes; validated by Mappings Manager (no loops, Justin's Laws); used in both inbound and outbound sync |
| **Subscription Mapping** | Mapping between external system consent/topic fields and Audience subscription topics; handles consent basis (native resource or "virtual consent" mapped to contact fields) |
| **Virtual Consent** | Legacy approach to consent mapping where consent basis mapped to fields on contact card (vs. native consent resource); used in some legacy connectors |
| **Justin's Laws** | Validation rules enforced by Mappings Manager on field/subscription mappings; prevents mapping loops and enforces other constraints (exact rules beyond "no loops" not fully specified) |
| **All Sub Queue** | Central event queue for all subscription/activity events from Audience; buffers delta sync messages during full sync; routes outbound activities and consent changes |
| **All Sub Worker** | ECS task that validates events, checks full sync status, buffers delta messages using SQS visibility timeout, and routes outbound events to Kafka |
| **Batch Production Worker** | ECS task that polls Kafka partition, groups messages by integration, and creates batches when size reaches 200K or polling time exceeds 1 second |
| **Outbound Manager** | Bridge between campaign editor front-end and external systems; initiates outbound activity sync (emails, SMS, forms, etc.) |
| **Outbound Worker** | ECS task that consumes batched messages from SQS FIFO queue, sends to external systems via connector library, and routes failures to dead letter queue |
| **Dead Letter Queue** | SQS queue receiving batches after max retries from Outbound Worker; Retry Driver re-queues daily at 6:00 AM for indefinite retry |
| **Retry Driver Lambda** | Lambda function running daily at 6:00 AM; re-queues all messages from dead letter queue back to SQS FIFO queue for indefinite retry |
| **Profile List** | Abstraction representing contact lists from external systems (static or dynamic query-based); import adds tags to existing Audience profiles (does NOT create new profiles) |
| **Eventual Consistency** | Property of distributed systems where data eventually becomes consistent across all nodes; during full sync, buffering strategy ensures delta messages arrive after full sync to prevent out-of-order updates |
| **Ingestion Delay** | Estimated time for Audience to ingest all profiles from Justin; Full Sync Manager waits for this delay before marking sync as complete to ensure eventual consistency |
| **SQS Visibility Timeout** | SQS feature that temporarily hides message from other workers; used to delay delta sync messages during full sync without consuming them |
| **Unified Data** | Advanced feature enabling complex multi-table CRM queries for personalization; uses HTTP/2 streaming to avoid loading millions of rows into memory; joins external CRM data with Audience profiles via Athena |
| **HTTP/2 Streaming** | Protocol for streaming large result sets from integrations to Audience without buffering entire payload; used for Unified Data to handle millions of CRM records |
| **Apache Columnar Format** | Format for encoding tabular data used in Unified Data; translated on-the-fly during HTTP/2 streaming to avoid memory buffering |
| **Partner Model** | Current integration approach where external partners implement generic connector API spec; partners own customer relationships, development, and revenue (vs. legacy supplier model) |
| **Supplier Model (Legacy)** | Earlier approach where APSIS contracted with vendor for custom implementation; APSIS paid hourly, owned SLAs, slower time-to-market |
| **Feature Flag** | Configuration enabling/disabling features per integration or account; used for capability discovery and experimental features |
| **FIFO Queue** | First-In-First-Out SQS queue ensuring message ordering; used for delta sync (partitioned by CRM ID) and outbound batches (partitioned by integration) |
| **Kafka Partition** | Kafka topic partition for outbound events; partitioned by integration ensuring per-integration order but allowing global interleaving |
| **Profile Cloud** | Original integration platform for APSIS predecessor (2013–2014 era); predecessor to Audience in APSIS One |
| **Athena** | Query engine in Audience used for joined data in Unified Data scenarios; joins streamed external CRM data with profile exports |
| **React** | Real-time profile store (mentioned in transcript but not detailed); used for real-time profile state during email personalization |
| **Sideshop** | Current partner for Microsoft Dynamics 365 integration; controls generic connector implementation; migration from legacy supplier-based Dynamics ongoing |
| **Intermail** | Partner implementing Intermail Loyalty (custom) and Relation Plus (generic connector-based); brings own customers to APSIS; revenue-sharing model |
| **SLA (Service Level Agreement)** | Commitment to uptime, response time, and support availability; Integrations team commits to same SLA as rest of APSIS One platform (24/7 pager duty) |
| **Abandoned Cart Infrastructure** | Historical integration pattern for eCommerce systems; hourly fetch of carts, emitting events to Audience for marketing automation |
| **Segment Evaluation API** | API used by legacy CMS integrations (Drupal, FP Server, Sitecore); CMS calls APSIS at render time to check if user is in segment |
| **CMS Integration** | Integration with content management systems (legacy: Drupal, FP Server, Sitecore); different from CRM integrations; no profile sync; user gets segment dropdown in CMS editor |
| **Idempotency** | Property ensuring repeated operations have same effect as single operation; external systems must track batch IDs during redrive to prevent duplicate events |
| **Clear Integration Endpoint** | REST API endpoint that forcefully removes customer integration from APSIS database using ignore_external_system_errors flag |
| **ignore_external_system_errors flag** | Boolean flag allowing uninstallation to proceed despite CRM communication failures; when true, APSIS deletes all database entries regardless of CRM errors |
| **Delete Integration Secret Key** | Single authorization key stored in AWS Secrets Manager; grants permission to call Clear Integration endpoint; works universally across all integration types |
| **Holy Trinity** | The three required parameters to identify unique integration installation: Account ID, Section ID, and Integration ID |
| **Force Deletion** | Uninstallation process that succeeds even when external CRM is misconfigured, unreachable, or has changed API credentials |
| **Orphaned Webhooks** | Webhook configurations remaining in customer's CRM after forced integration deletion from APSIS; continue sending contact data to endpoints that no longer exist |
| **Integration Manager (logging)** | System that logs and stores integration lifecycle events; used to verify installation details, check deletion status, and audit customer activity |
| **AltaSync Manager** | Monitoring system that observes if customer's CRM continues sending webhook data after integration deletion |
| **Squid Proxy** | Database component tracking proxy routing for integration connections; deleted during uninstallation cleanup but may rarely hang |
| **Legacy Connector** | Integration connectors (FCC Enterprise, Lime) predating Generic Connector; maintained for backward compatibility |
| **Delegation Key** | Customer-specific authorization key used to call certain APIs on behalf of customer; potential future alternative to internal Delete Integration Secret Key |
| **Zero-Point Story** | Task tracking unit (not estimated in story points) used for customer escalations and investigations; provides visibility and historical knowledge |
| **API Key Rotation** | Process where customer regenerates API credentials in their CRM without uninstalling integration; handled via Update API Key endpoint |
| **Installation Table** | Database table storing authoritative mapping of customer accounts to integration versions and deployed sections |
| **Display Name** | Customer-visible label for connector shown in UI and documentation; can be changed freely; used for folder creation in Apsis One |
| **Integration ID** | Internal logical identifier for connector; part of API contract; must remain stable; used throughout logs and database |
| **Additional Entities** | Configuration section in connector installer files defining non-primary entities supported by connector (e.g., lead entities) |
| **Lead Entity** | Separate entity type in CRM systems representing prospective customers not yet fully qualified; platform creates dedicated lead ID attribute folders for CRMs supporting this |
| **Folder Creation** | Automatic creation of folder structures in Apsis One when CRM integration installed; now uses Display Name instead of Integration ID |
| **Lead ID Attribute Folder** | Dedicated folder created in Apsis for storing lead identifier attributes for CRMs supporting separate lead entities |
| **Delegated Keys (IAM)** | Internal service credentials from Identity and Access Management system; allow maintenance operations without requiring customer account access |
| **syncConditionSupported** | Boolean property on connector indicating whether it supports server-side sync condition filtering; currently true for Site Shop; will be true for FSC Enterprise once CRM vendor implements endpoint |
| **SYNC_CONDITIONS_ENDPOINT** | Proposed endpoint `/api/v1/sync-conditions/{section}` for CRM to register and enforce sync conditions (not yet implemented) |

---

## 8. Confidence Notes

### High Confidence (Verified & Consistent)
- Architecture overview (Justin as middleware layer)
- Core components (Delta Sync Worker, Full Sync Manager, All Sub Worker, Outbound Worker, Retry Driver)
- Inbound sync flow (webhooks → SQS → Delta Sync Worker → Audience)
- Outbound sync flow (Audience events → Kafka → Batch Production Worker → SQS → Outbound Worker)
- CRM ID limitation and post-2020 fix with system-specific IDs
- Consent bidirectionality (only consent synced back, not attributes)
- Opt-in not redriven from dead letter queue (contractual requirement)
- Partner model vs. supplier model shift
- Batch size hardcoding (200K, not configurable)
- Profile list imports do NOT create new profiles
- Sync conditions applied on APSIS side AFTER download
- Visibility timeout buffering during full sync
- CloudWatch scheduled events (6:00 AM for profile list imports, dead letter queue redrive)
- VPN tunnels not supported for on-premise deployments

### Medium Confidence (Partially Verified, Some Gaps)
- Exact specification of "Justin's Laws" beyond "no loops" — not fully detailed
- Retry count before dead letter queue placement — **NOT SPECIFIED** in any transcript
- Ingestion delay value — held by Audience team, not specified in Integrations team knowledge
- Full Sync Manager recovery logic for stuck states — not documented
- Consent comparison optimization details — mentioned but not deeply explored
- Postman collection locations and scope — partially mentioned, exact paths not provided
- Feature flag architecture details — two-layer checking mentioned but incomplete documentation
- Unified Data HTTP/2 streaming error handling — lightly tested, edge cases unknown
- Site Shop connector details — mentioned as success case for server-side filtering but limited implementation detail
- CRM vendor engineering timelines for server-side filtering implementation — uncertain
- Delegation keys future approach — noted as optimization but requires security/compliance review

### Low Confidence / Gaps ❌ (Unresolved Questions)
1. **Exact retry count before dead letter queue placement** — Not specified in any transcript
2. **Ingestion delay value from Audience** — Used by Full Sync Manager but exact value not provided
3. **Exact implementation of Justin's Laws validation** — Only "no loops" mentioned; other rules not specified
4. **SQS queue retention configuration** — Assumed 14-day default but not explicitly confirmed
5. **Full Sync Manager recovery logic** — How does stuck state ("Downloading"/"Finalizing") get resolved? Not documented
6. **Integration Manager → ECS migration reason** — Why Lambda changed to ECS task? Cost, performance, or other?
7. **Postman collection storage location** — Mentioned for debugging but exact repository/URL not provided
8. **Generic Connector API Spec location** — S3 bucket referenced but exact bucket/path not confirmed
9. **Squid Proxy purpose in detail** — What is proxy routing? Why is it deleted during uninstall?
10. **React real-time profile store usage** — Mentioned but never detailed; actively used or deprecated?
11. **CRM vendor idempotency implementation** — How do different systems (Dynamics, Lime, Maxo) track batch IDs? Not documented
12. **Feature flag evaluation order** — Is in-code flag checked first or customer instance check first?
13. **Full sync error recovery** — If Producer timeout or Consumer crash occurs, what happens to Full Sync Queue?
14. **Orphaned webhook cleanup timeline** — How long does customer have to clean up webhooks? Is there a deadline?
15. **Consent comparison memory threshold** — At what number of records does in-memory export comparison cause heap exhaustion?
16. **Kafka consumer group ID** — What group ID does Batch Production Worker use? Is it configurable?
17. **All Sub Worker full sync status check** — How is "is full sync ongoing?" state determined? Query to Full Sync Manager? Database flag?
18. **Email tool unified data request format** — How does email tool request unified data query? API call? UI selector?
19. **CRM connector feature discovery** — How does integration system know what features a CRM supports? Hardcoded flags? API call?
20. **Sideshop migration commercial status** — What is current blocker preventing full Dynamics migration to Sideshop model?

### Conflicts Resolved
- **Display names vs. Integration IDs**: Clarified in PR transcript that display names can change (UI labels) but integration IDs are locked (API contracts)
- **Full sync state progression**: Verified as "Downloading" → "Finalizing" → "Successful" with visibility timeout buffering
- **Consent endpoint optimization**: Clarified that problem is CRM vendor's consent pagination (22 sec per 500), not APSIS architecture

### Contradictions / Uncertainties
- ⚠️ **Exact ingestion delay value**: Mentioned multiple times as critical for Full Sync Manager but actual value never provided — team should obtain from Audience team
- ⚠️ **Retry count**: Outbound Worker mentions "after max retries" but count never specified — affects dead letter queue timing
- ⚠️ **Full sync recovery**: State can "stick" but no recovery process documented — operational risk
- ⚠️ **React profile store**: Mentioned once in architecture overview but never used or explained — may be deprecated or internal implementation detail

### Data Quality Issues
- Some module descriptions are high-level; detailed implementation (e.g., exact database schema, error codes) not provided
- Configuration section incomplete; some settings noted as "to be determined after session"
- Tribal knowledge heavily weighted toward known issues and workarounds; not all positive patterns documented
- CRM vendor implementation details (Sideshop, Intermail, Maxo) mentioned but not deeply explored
- Legacy system details (Profile Cloud, Segment Evaluation API for CMS) provided for context but not maintained as active systems

---

## 9. Recommended Next Steps for Knowledge Base Maintenance

1. **Obtain missing authoritative values**:
   - Get Ingestion Delay from Audience team
   - Confirm Retry Count from Outbound Worker implementation
   - Clarify exact Justin's Laws beyond "no loops"
   - Confirm Generic Connector API Spec location (S3 path)

2. **Document operational procedures**:
   - Full Sync Manager stuck state recovery
   - Squid Proxy purpose and deletion logic
   - Full Sync Consumer error recovery
   - Feature flag evaluation order

3. **Clarify ambiguous components**:
   - React real-time profile store (active or deprecated?)
   - Sideshop Dynamics migration current status
   - CRM vendor batch ID tracking implementations

4. **Create reference diagrams** (not yet provided):
   - Data flow diagram (inbound, full sync, outbound)
   - State machine for full sync progression
   - Eventual consistency buffering timeline
   - Kafka/SQS/FIFO queue topology

5. **Consolidate CRM-specific knowledge**:
   - Postman collection locations and usage
   - Per-connector gotchas and workarounds
   - Feature support matrix by CRM type

6. **Establish architectural decision tracking**:
   - Site Shop server-side filtering model (proposed for other connectors)
   - Consent endpoint optimization strategy
   - Profile list import sequencing improvement

---

**Last Updated**: 2026-03-20 (Session 4 / Final Consolidation)  
**Extracted From**: 4 KT Sessions (Benjamin, Erik)  
**Domain Owner**: Integrations Team (Justin)  
**Maintenance Notes**: This KB deduplicates 4 sessions, preserves exact paths/names, flags unresolved questions with ❌, and marks uncertain info with ⚠️. Review quarterly or after significant architecture changes.