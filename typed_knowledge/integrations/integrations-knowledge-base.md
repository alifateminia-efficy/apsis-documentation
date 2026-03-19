---
title: Integrations Domain Knowledge Base
generated: 2026-03-19T21:47:37.972Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (4 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Integrations Domain Knowledge Base](#integrations-domain-knowledge-base)
  - [1. Domain Overview](#1-domain-overview)
  - [2. Architecture Map](#2-architecture-map)
    - [Core Components](#core-components)
    - [Data Flow](#data-flow)
    - [Supporting Infrastructure](#supporting-infrastructure)
  - [3. Module Reference](#3-module-reference)
    - [Integration Manager](#integration-manager)
    - [Mappings Manager](#mappings-manager)
    - [Delta Sync Worker](#delta-sync-worker)
    - [Delta Sync Buffer Queue](#delta-sync-buffer-queue)
    - [Full Sync Manager](#full-sync-manager)
    - [Full Sync Producer](#full-sync-producer)
    - [Full Sync Consumer](#full-sync-consumer)
    - [Profile List Sync](#profile-list-sync)
    - [Outbound Manager](#outbound-manager)
    - [All Sub Worker](#all-sub-worker)
    - [Batch Production Worker](#batch-production-worker)
    - [Outbound Worker](#outbound-worker)
    - [Consent Outbound (Unidirectional)](#consent-outbound-unidirectional)
    - [Outbound Mappings](#outbound-mappings)
    - [Unified Data](#unified-data)
    - [Connector Libraries (System-Specific)](#connector-libraries-system-specific)
    - [Generic Connector (Partnership Model)](#generic-connector-partnership-model)
    - [Clear Integration Endpoint](#clear-integration-endpoint)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling](#error-handling)
    - [Logging & Observability](#logging-observability)
    - [Data Privacy & Compliance](#data-privacy-compliance)
    - [Scaling & Performance](#scaling-performance)
    - [Deployment & Lifecycle](#deployment-lifecycle)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Sync Direction & Bidirectionality](#sync-direction-bidirectionality)
    - [Sync Conditions (Filtering)](#sync-conditions-filtering)
    - [Profile Creation](#profile-creation)
    - [Integration Lifecycle](#integration-lifecycle)
    - [Message Ordering & Consistency](#message-ordering-consistency)
    - [Batching & Delivery](#batching-delivery)
    - [Outbound Mappings](#outbound-mappings)
    - [Webhook Lifecycle](#webhook-lifecycle)
    - [Recurring Imports](#recurring-imports)
    - [Data Retention & Cleanup](#data-retention-cleanup)
    - [CRM Key Space (⚠️ Architectural Debt)](#crm-key-space-architectural-debt)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [Performance & Scaling Issues](#performance-scaling-issues)
    - [Integration Lifecycle Issues](#integration-lifecycle-issues)
    - [Connector & Configuration Issues](#connector-configuration-issues)
    - [Customer Support & Process Issues](#customer-support-process-issues)
    - [Data & Metadata Issues](#data-metadata-issues)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence (Multiple Sources, Aligned)](#high-confidence-multiple-sources-aligned)
    - [Medium Confidence (Single Source, Reasonable Detail)](#medium-confidence-single-source-reasonable-detail)
    - [Lower Confidence (Incomplete, Contradictions, or Gaps)](#lower-confidence-incomplete-contradictions-or-gaps)
    - [Known Gaps & Disagreements](#known-gaps-disagreements)
    - [Explicit Uncertainties Flagged in Original Data](#explicit-uncertainties-flagged-in-original-data)
    - [Confidence by Component](#confidence-by-component)
  - [Tribal Knowledge Consolidated](#tribal-knowledge-consolidated)
    - [Gotchas (Common Trip-Ups)](#gotchas-common-trip-ups)
    - [Historical Context](#historical-context)
    - [Optimization Opportunities](#optimization-opportunities)
    - [Red Flags for Review](#red-flags-for-review)
  - [Summary & Recommendations](#summary-recommendations)
    - [What You Own](#what-you-own)
    - [Critical Issues to Address](#critical-issues-to-address)
    - [Quick Wins (Weeks)](#quick-wins-weeks)
    - [Medium-Term (Months)](#medium-term-months)
    - [Long-Term (Quarters)](#long-term-quarters)

---

# Integrations Domain Knowledge Base

## 1. Domain Overview

The Integrations domain manages bidirectional synchronization of customer data between APSIS One (the central profile platform) and external CRM/business systems (Dynamics 365, Lime CRM, FSC Enterprise, Salesforce, etc.). It handles inbound contact/consent data ingestion via full and delta syncs, outbound activity/engagement event delivery, field mapping, consent management, and integration lifecycle operations (install, configure, delete). The domain owns the "Justin" middleware stack—a modular architecture separating generic infrastructure from system-specific connector libraries—enabling APSIS to scale across multiple external systems without building integration logic for each.

---

## 2. Architecture Map

### Core Components
- **Integration Manager** (ECS task) — Schema discovery, connector routing, feature capability checks
- **Mappings Manager** (ECS task) — Field/consent mapping validation, webhook subscription management, cache layer
- **Delta Sync Worker** (ECS task) — Real-time webhook message processing (SQS FIFO → Audience)
- **Delta Sync Buffer Queue** (SQS FIFO) — Buffers real-time messages during full sync; replays after completion
- **Full Sync Manager** (ECS task) — Orchestrates bulk downloads; spins up Producer/Consumer stacks
- **Full Sync Producer** (ECS task) — Paginates contact data from CRM; converts to Justin format; queues batches
- **Full Sync Consumer** (ECS task) — Applies mappings from queue; sends to Audience
- **Profile List Sync** (ECS task) — Imports static/dynamic lists; tags existing profiles (no profile creation)
- **Outbound Manager** (ECS task) — Activity sync coordination (emails/SMS/forms to external systems)
- **All Sub Worker** (ECS task) — Validates and routes engagement events from Audience → Kafka by integration
- **Batch Production Worker** (ECS task) — Batches Kafka events (200KB cap or 1s timeout) → SQS FIFO
- **Outbound Worker** (ECS task) — Sends batches to external systems via connectors; DLQ + daily retry via Lambda

### Data Flow
```
External System ←→ Webhooks/APIs ←→ [Justin Middleware] ←→ Audience (profile store)
                                          ↓
                               Connector Libraries (system-specific)
```

### Supporting Infrastructure
- **Connector Libraries** — Per-CRM translation layer (standardized contract)
- **Generic Connector** — Partner-implemented REST API spec for new integrations
- **RDS Database** — Integration metadata, credentials, mappings, webhooks
- **Secrets Manager** — API keys, Delete Integration Secret Key
- **SQS FIFO Queues** — Delta Sync, Full Sync, Outbound, Consent routing (ordered processing)
- **Kafka** — Event fan-out for multi-integration batching
- **CloudWatch/Lambda** — Profile list recurring import triggers (6:00 AM UTC), DLQ retry driver

---

## 3. Module Reference

### Integration Manager
- **Purpose**: Acts as interface between front-end and external systems; fetches schema, routes connector calls, handles feature discovery.
- **Key files**: 
  - `integration-manager` service (ECS long-running task)
  - `connector libraries` (per integration type)
  - `feature flag configuration`
- **Tech stack**: ECS (long-running), HTTP/2 (for Unified Data streaming), Connector libraries (polyglot)
- **Data flow**: Front-end → Integration Manager → Connector Library → External System API → Schema/feature metadata → Front-end
- **Business rules**:
  - Feature discovery (e.g., "Can I sync emails to this CRM?") checked before activity creation UI shows sync option
  - Schema discovery called every time user opens field mapping UI (performance risk for large external systems)
  - HTTP/2 streaming used for Unified Data relational CRM queries (no in-memory result set limitation)
- **Configuration**: 
  - Feature flag per integration type controls availability
  - Instance-level capability check required (APSIS + external system both must support feature)
- **Integration points**: External system APIs, Unified Data endpoint, Mappings Manager
- **Gotchas**:
  - ⚠️ Schema discovery is called frequently without caching; can be slow for large external systems
  - Integration IDs (internal, immutable) vs. display names (customer-facing, mutable) — confusion common in logs
  - E-Deal is FSC Corporate with new display name; search logs for `FSC_CORPORATE` not "E-Deal"
- **Tribal knowledge**: 
  - Converted from Lambda to ECS for persistent state (schema caching, integration config) — Lambda cold starts and timeouts were problematic
  - Display name changes (e.g., "E-Deal") do NOT change integration ID (`FSC_CORPORATE`), which is embedded in API contracts

### Mappings Manager
- **Purpose**: Manages field mappings and consent mappings; enforces validation rules; updates webhooks on mapping changes; maintains cache for workers.
- **Key files**:
  - `mappings-manager` service (ECS task)
  - `mappings` database (RDS)
  - `mapping cache` (in-memory, invalidated on changes)
- **Tech stack**: ECS, RDS, cache layer
- **Data flow**: User configures mappings → Validates (Justin's Laws) → Stores in DB → Invalidates cache → Updates webhooks → Workers query cache during sync
- **Business rules**:
  - **Justin's Laws** (validation rules enforced during mapping creation; exact rules not detailed but prevent invalid mappings)
  - Field data syncs inbound only (CRM → APSIS); consent syncs bidirectionally (CRM ↔ APSIS)
  - Sync conditions: filter rules (e.g., "only sync active contacts") configured in APSIS but supposed to be enforced server-side by CRM
    - ⚠️ **Current issue**: FSC Enterprise does NOT implement server-side filtering; APSIS downloads all data and filters locally (major performance bottleneck)
    - Reference implementation: Dynamics Side Shop (Site Shop) successfully filters server-side
  - Outbound Mappings are account-level, not form-specific (technical limitation from pre-2022)
  - Webhooks auto-subscribe when mappings created; auto-update when mappings change
- **Configuration**: 
  - `sync_condition_property_enabled` (true for generic connector) — tells CRM to enforce filter rules
  - Cache invalidation strategy (timing not specified)
- **Integration points**: Integration Manager, Delta Sync Worker, Full Sync Consumer, Outbound Worker, Webhook subscription service
- **Gotchas**:
  - ⚠️ Sync conditions are just a data structure on APSIS side — they're only effective if CRM implements server-side enforcement
  - Mapping changes take effect immediately; brief window where delta sync may process stale mappings during cache invalidation
  - Some external systems use custom webhook endpoints instead of standard Delta Sync Manager endpoint (less ideal but necessary)
- **Tribal knowledge**: 
  - "Justin's Laws" is a domain-specific validation framework but rules are not documented in this KB
  - Outbound Mappings at account level was a historical compromise during Maxo development; form-specific mappings would require rework of form submission event pipeline

### Delta Sync Worker
- **Purpose**: Consumes real-time webhook messages from SQS FIFO queue; applies field and consent mappings; sends profile updates to Audience.
- **Key files**:
  - `delta-sync-worker` service (ECS long-running task)
  - `Delta Sync Queue` (SQS FIFO, CRM_ID as message group ID for per-contact ordering)
- **Tech stack**: ECS, SQS FIFO, Audience profile API
- **Data flow**: Webhook → Delta Sync Queue (FIFO) → Delta Sync Worker → Query Mappings Manager for applicable mappings → Audience profile update
- **Business rules**:
  - SQS FIFO queue with CRM_ID as message group ID ensures ordering per contact while allowing parallel processing of different contacts
  - If full sync running: delta messages buffered in Delta Sync Buffer Queue with visibility timeout, replayed in order after full sync completes
  - Consent messages bypass Batch Production Worker (sent individually, not batched)
- **Configuration**: 
  - Mappings cache queried for applicable field/consent mappings per message
  - Visibility timeout duration (exact value not specified but used to delay buffered messages)
- **Integration points**: External systems (webhooks), Mappings Manager (mapping lookup), Audience (profile ingestion), Delta Sync Buffer Queue (if full sync running)
- **Gotchas**:
  - ⚠️ FIFO queue ensures order but also creates sequential processing bottleneck if one contact has many updates (not parallelizable per contact)
  - Buffering during full sync delays real-time updates by full sync duration; this is intentional to avoid inconsistency
  - If same contact updated multiple times during full sync, buffered messages replayed in order = eventual consistency, not immediate consistency
- **Tribal knowledge**:
  - CRM_ID is used as single key space for all CRM integrations (architectural debt — proper solution would use system-specific key spaces like `Dynamics_ID`, `Salesforce_ID`)
  - "Jonas Super Urgent Release" in early APSIS development established this limitation; never properly fixed for multiple CRM systems per section

### Delta Sync Buffer Queue
- **Purpose**: Temporarily buffers real-time delta messages while full sync in progress; replays messages in order after full sync completes to maintain eventual consistency.
- **Key files**: `Delta Sync Buffer Queue` (SQS FIFO with visibility timeout)
- **Tech stack**: SQS FIFO
- **Data flow**: If full sync running → delta messages → set visibility timeout → after full sync completes → replay in order → Delta Sync Worker
- **Business rules**:
  - Visibility timeout ensures buffered messages are not processed until full sync completes
  - Messages replayed in order (FIFO guarantee)
  - Prevents duplicate updates or missing intermediate state during full sync window
- **Integration points**: Delta Sync Worker, Full Sync Manager (signals completion)
- **Gotchas**:
  - ⚠️ If visibility timeout expires before full sync completes, messages will appear again and be double-processed (potential data corruption)
  - No documented way to know full sync progress; relies on Full Sync Manager signaling completion
- **Tribal knowledge**: 
  - This component was added specifically to solve eventual consistency problem discovered during integration work; previously, real-time and bulk messages could interleave causing inconsistency
  - Demonstrates architecture decision to prefer buffering complexity over accepting inconsistent states

### Full Sync Manager
- **Purpose**: Orchestrates bulk data download from external systems; spins up on-demand producer/consumer stacks; monitors progress; signals completion when Audience ingestion catches up.
- **Key files**:
  - `full-sync-manager` service (ECS task)
  - `Full Sync Queue` (SQS FIFO)
- **Tech stack**: ECS, SQS FIFO, CloudWatch
- **Data flow**: User clicks 'Start' → Manager spins up Producer (fetches paginated data) + Consumer (applies mappings, sends to Audience) → Producer queues batches → Consumer processes → Manager monitors → signals 'Successful' when Audience catches up
- **Business rules**:
  - Spins up dedicated, on-demand Producer/Consumer stack per full sync operation
  - Monitors CloudWatch metrics to detect when Audience ingestion catches up with final message timestamp
  - Signals 'Successful' only after all data is live in Audience
  - While running: Delta Sync Buffer Queue buffers real-time messages; after completion, buffered messages are replayed
- **Configuration**: 
  - On-demand scaling (creates task instances when needed)
  - CloudWatch metrics monitoring (exact metrics not specified)
- **Integration points**: Full Sync Producer/Consumer, Mappings Manager, Audience, Delta Sync Buffer Queue
- **Gotchas**:
  - ⚠️ Full sync can take hours for large systems (millions of contacts); user must not interrupt or entire sync may be partial
  - If full sync fails partway through, messages already on queue are still processed; partial sync can occur
  - UI shows 'Finalizing' state while waiting for Audience to catch up; can take several minutes for large syncs
  - No documented maximum contact volume; historical baseline 2M contacts; current largest customer 8M (4x) exposes architectural limits
- **Tribal knowledge**: 
  - On-demand stack approach allows horizontal scaling without maintaining always-on Producer/Consumer infrastructure
  - Audience catch-up wait time is non-obvious complexity; must account for Audience's own event processing lag

### Full Sync Producer
- **Purpose**: Fetches paginated contact data from external system using connector library; converts to Justin format; puts batches on Full Sync Queue.
- **Key files**:
  - `full-sync-producer` service (ECS task)
  - `connector libraries` (system-specific)
- **Tech stack**: ECS, HTTP pagination, Connector library
- **Data flow**: External System API (paginated) → Producer → [convert to Justin format] → Full Sync Queue
- **Business rules**:
  - Handles pagination; batches converted to Justin format before queuing
  - Signals completion when no more pages available
  - Uses stored credentials from Mappings Manager
- **Configuration**: Pagination batch size (connector-specific), connector library endpoint config
- **Integration points**: External system API, Full Sync Queue, Full Sync Manager
- **Gotchas**:
  - ⚠️ Contact pagination works fine even for 8M contacts (16,000 pages); consent pagination times out (HTTP 504 Gateway Timeout at 500-entry batches = 22 sec per batch)
  - If single pagination timeout occurs, entire full sync fails (no retry logic for individual page failures)
  - Justin format conversion may lose data if external system has complex nested structures
- **Tribal knowledge**: 
  - Pagination issues are often CRM-side problems (proxy timeouts, query optimization, caching failures) rather than APSIS-side
  - One FSC Enterprise customer with 8M contacts exposed timeout issues; previous testing maximum was 2M (no knowledge of 4x+ scaling limits)

### Full Sync Consumer
- **Purpose**: Consumes messages from Full Sync Queue; queries Mappings Manager for applicable field and consent mappings; sends records to Audience.
- **Key files**:
  - `full-sync-consumer` service (ECS task)
  - `Full Sync Queue` (SQS FIFO)
- **Tech stack**: ECS, Audience profile API
- **Data flow**: Full Sync Queue → Consumer → [query Mappings Manager] → Audience profile ingestion
- **Business rules**:
  - Applies sync conditions (filter rules) configured in Mappings Manager
  - ⚠️ **Currently filters locally after downloading all data** (should be server-side by CRM)
  - Loads entire APSIS audience export into memory for consent comparison (optimization to skip unchanged entries)
  - Compares CRM consents against in-memory export; only sends update messages for changed consent statuses
  - Streams changed consent entries to SQS for asynchronous processing
- **Configuration**: 
  - `audience_export_optimization_enabled` (true, but flagged for removal)
  - Sync condition filtering rules from Mappings Manager
- **Integration points**: Full Sync Queue, Mappings Manager, Audience, SQS (consent stream output)
- **Gotchas**:
  - ⚠️ **CRITICAL**: In-memory audience export comparison causes memory exhaustion crashes with large existing APSIS populations (7-8M profiles with multiple consents)
  - ⚠️ Local filtering after downloading all data is inefficient (8M CRM contacts → filter locally to 1.2M eligible → sync to APSIS)
  - Consent pagination times out on large datasets (500 consents = 22 sec; 8M contacts = 97+ hours estimated)
  - One pagination timeout cascades entire sync failure
- **Tribal knowledge**: 
  - In-memory optimization works for current customer base but is "hidden time bomb" at scale
  - Proper fix is to stream directly to SQS without pre-comparison (removes need for in-memory export)
  - Sync conditions on APSIS side is "theater" — they only work if CRM implements endpoint support

### Profile List Sync
- **Purpose**: Imports static and dynamic lists from external systems; tags matching profiles without creating new profiles; handles recurring imports.
- **Key files**:
  - `profile-list-sync` service (ECS task)
  - `Profile List Sync Queue` (SQS)
  - `Profile List Consumer/Worker`
- **Tech stack**: ECS, CloudWatch Events (Lambda trigger), SQS
- **Data flow**: User selects list → Full Sync Manager routes to Profile List Queue → Profile List Worker → [fetch list members] → Audience tag updates
- **Business rules**:
  - **Profiles not created by list imports; only existing profiles tagged**
  - If profile exists in Audience, add tag named after the list
  - If profile no longer on list in next import, remove the tag
  - Recurring imports triggered by CloudWatch Event at 6:00 AM UTC (hardcoded, not configurable without code change)
  - One worker per import job (not clear if parallel workers supported)
- **Configuration**: 
  - Recurring import schedule: hardcoded 6:00 AM UTC (noted as easy to extend to other schedules)
  - List selection UI in Integration Manager
- **Integration points**: External CRM systems (list APIs), Audience (tag API), Full Sync infrastructure
- **Gotchas**:
  - ⚠️ List imports do NOT create new profiles; only tag existing profiles
  - ⚠️ Memory usage can be high if list is very large (all contacts held in memory during processing); historical failures on large lists
  - If profile removed from list externally, tag removed on next import (no record of deletion in APSIS)
  - Recurring import timing is hardcoded; changing requires code change and deployment
- **Tribal knowledge**: 
  - Profile list sync reuses Full Sync infrastructure but has different semantics (tags, not profile creation)
  - Scheduled trigger via CloudWatch is fragile; would benefit from webhook-based or on-demand triggering

### Outbound Manager
- **Purpose**: Bridges creation of activities (emails, SMS, forms) in APSIS with external system records; coordinates syncing of activity and engagement events back to external systems.
- **Key files**: `outbound-manager` service (ECS task)
- **Tech stack**: ECS, activity submission API
- **Data flow**: User creates activity → Front-end checks Integration Manager: 'Can sync?' → If yes, Outbound Manager notifies external system to create record
- **Business rules**:
  - Checks with Integration Manager before allowing activity sync (feature discovery)
  - Creates corresponding record in external system when activity created in APSIS (if sync enabled)
  - Routes engagement events to All Sub Worker for downstream batching/delivery
- **Configuration**: Feature flags per integration type
- **Integration points**: Front-end (activity creation), Audience (activity/event source), All Sub Worker, Batch Production Worker
- **Gotchas**:
  - ⚠️ Outbound Mappings are account-level, not form-specific (can't have different field mappings per form submission)
  - Activity sync is not available for all activity types (e.g., surveys not supported)
- **Tribal knowledge**: 
  - Activity sync feature is "bidirectional" in that APSIS creates external records, but field sync is "unidirectional" (CRM → APSIS)

### All Sub Worker
- **Purpose**: Consumes Audience Subscription Queue (all engagement events); validates integration installed and activity should sync; routes events to Kafka partition per integration.
- **Key files**:
  - `all-sub-worker` service (ECS long-running task)
  - `Audience Subscription Queue` (SQS FIFO)
  - `Kafka topic` (per integration or mixed partition)
- **Tech stack**: ECS, SQS FIFO, Kafka
- **Data flow**: Audience engagement events → Audience Subscription Queue → All Sub Worker → [validate integration + activity] → Kafka (per integration partition)
- **Business rules**:
  - Validates that integration is installed in this section
  - Validates that activity should be synced to this integration
  - Routes events by integration to Kafka (partitioning by integration ID)
  - Consent events bypass this worker (sent directly to Consent routing, not batched)
- **Configuration**: Integration validation logic (exact validation rules not detailed)
- **Integration points**: Audience (event source), Batch Production Worker, Kafka
- **Gotchas**:
  - ⚠️ Kafka partitioning by integration may create unbalanced load if one integration has vastly more events
  - Consent events NOT routed through this worker; separate consent routing logic exists
- **Tribal knowledge**: 
  - Worker name "All Sub Worker" is historical (possibly "All Subscription Worker"); not intuitive from name

### Batch Production Worker
- **Purpose**: Polls Kafka events until batch > 200KB or > 1 second elapsed; groups events by integration; puts batches on Outbound Queue (SQS FIFO).
- **Key files**:
  - `batch-production-worker` service (ECS long-running task)
  - `Outbound Queue` (SQS FIFO, 256KB limit)
- **Tech stack**: ECS, Kafka consumer, SQS
- **Data flow**: Kafka (mixed events) → Batch Production Worker → [group by integration, batch by size/time] → Outbound Queue (batches per integration)
- **Business rules**:
  - Batches events until either: batch size > 200KB OR 1 second elapsed (whichever comes first)
  - Groups by integration (each batch is for one integration only)
  - Puts batches on Outbound Queue (SQS FIFO, 256KB limit; must fit within SQS message size limit)
  - Consent messages sent individually, NOT batched (bypass this worker)
- **Configuration**: 
  - Batch size threshold: 200KB (hardcoded?)
  - Batch timeout: 1 second (hardcoded?)
- **Integration points**: Kafka, Outbound Worker, Outbound Queue
- **Gotchas**:
  - ⚠️ 1-second batching delay means engagement events are delayed before being sent to external system (by design for efficiency)
  - For 1 million events, could create 1000+ batches; no documented performance issues but potential optimization target
  - Batches must fit within SQS 256KB message limit; large batches may fail
  - Batch ID must be unique for idempotency detection in external systems
- **Tribal knowledge**: 
  - Batching introduces minimal latency (1s) for massive throughput efficiency (vs. sending one-by-one would overwhelm external systems)
  - Kafka consumer lag could indicate bottleneck if worker can't keep up with event rate

### Outbound Worker
- **Purpose**: Consumes Outbound Queue; uses connector library to send batches to external systems; handles retries via DLQ and Retry Driver Lambda.
- **Key files**:
  - `outbound-worker` service (ECS long-running task)
  - `Outbound Queue` (SQS FIFO)
  - `Outbound Dead Letter Queue` (DLQ)
  - `Retry Driver Lambda` (runs daily at 6:00 AM UTC)
- **Tech stack**: ECS, SQS, Connector library, Lambda
- **Data flow**: Outbound Queue → Outbound Worker → [call connector library] → External System API. If fails after retries → DLQ. Daily: Retry Driver Lambda → redrive DLQ to Outbound Queue
- **Business rules**:
  - Consumes batches from Outbound Queue
  - Calls connector library to send batch to external system
  - Implements retry logic (exact retry count/strategy not documented)
  - On final failure: moves to DLQ
  - Daily at 6:00 AM: Retry Driver Lambda pulls from DLQ, redrives to Outbound Queue
  - Messages stay in DLQ/retry loop for up to 14 days, then no automatic retry (improvement noted)
  - Consent routing: ⚠️ **only 'opt-in=false' (opt-outs) are redriven from DLQ** to avoid false positives if consent state changed after initial send
- **Configuration**: 
  - Retry schedule: daily at 6:00 AM UTC
  - DLQ retention: 14 days
  - Consent retry filtering: only opt-outs (opt-in=false)
- **Integration points**: Outbound Queue, External system APIs (via connector), DLQ
- **Gotchas**:
  - ⚠️ Dead Letter Queue messages only retried once per day; failed events can have 14-day delay before being dropped
  - ⚠️ Consent retry logic only redrive opt-outs; opt-ins that failed are NOT retried (safety measure)
  - No documented strategy for 14-day cleanup (messages are dropped silently after 14 days)
- **Tribal knowledge**: 
  - Consent retry filtering is a deliberate safety measure; replaying failed opt-ins could re-subscribe unintended contacts
  - DLQ with daily retry is a workaround; ideally would implement exponential backoff or jitter

### Consent Outbound (Unidirectional)
- **Purpose**: Processes consent changes (opt-in/opt-out) from Audience back to external systems; bypasses Batch Production Worker (sent individually, not batched).
- **Key files**:
  - `Audience Subscription Queue` (FIFO)
  - `Consent routing logic`
  - `Outbound Worker` (shared with engagement events)
- **Tech stack**: SQS FIFO, Outbound Worker, Connector library
- **Data flow**: Consent change in Audience → Subscription Queue → [bypass batching] → Outbound Worker → External System. If fails → DLQ. Daily retry only 'opt-in=false'.
- **Business rules**:
  - Consent messages NOT batched (sent immediately, individual)
  - Only failed opt-outs (opt-in=false) redriven from DLQ (failed opt-ins NOT retried)
  - Unidirectional: CRM → APSIS for consents is inbound (delta sync); APSIS → CRM for consents is outbound (this component)
- **Integration points**: Audience (consent event source), Outbound Worker, External systems
- **Gotchas**:
  - ⚠️ Consent events bypass efficiency gains of batching; each consent sent individually
  - ⚠️ Only opt-outs retried; opt-ins dropped after 14 days if failed (asymmetric retry logic)
- **Tribal knowledge**: 
  - Consent is intentionally bidirectional (unlike field data which is CRM → APSIS only) to maintain consistency in CRM

### Outbound Mappings
- **Purpose**: Defines field mappings in reverse direction (APSIS attributes → External system fields) for form submissions and custom attributes; account-level, not form-specific.
- **Key files**:
  - `Outbound Mappings configuration` (in Mappings Manager DB)
  - `Integration feature flags`
- **Tech stack**: Mappings Manager database, feature flag configuration
- **Data flow**: User enables Outbound Mappings feature → Integration Manager checks: [integration supports it in APSIS?] AND [external system instance supports it?] → All Sub Worker uses Outbound Mappings to construct external payload from profile attributes (not form event data)
- **Business rules**:
  - Account-level only (not form-specific; all forms use same mappings)
  - Two-level feature flag: integration-level flag in APSIS + instance-level capability check via Integration Manager
  - Maps APSIS profile attributes to external system fields (reverse of inbound mappings)
- **Configuration**: 
  - Integration feature flag (APSIS side)
  - Instance capability check (external system side)
- **Integration points**: Integration Manager (feature discovery), Mappings Manager, Form submission pipeline, All Sub Worker
- **Gotchas**:
  - ⚠️ Account-level restriction means can't have per-form custom field mappings
  - ⚠️ Feature availability determined by two-level flag check; if either level disabled, outbound mappings unavailable
- **Tribal knowledge**: 
  - Outbound Mappings feature is relatively new; not all integrations support it
  - Form-specific mappings would require significant pipeline rework (noted as future improvement)

### Unified Data
- **Purpose**: Enables on-demand relational CRM queries during email personalization; streams large result sets via HTTP/2 without holding in memory.
- **Key files**:
  - `Integration Manager HTTP/2 streaming endpoint`
  - `Connector pre-built queries`
  - `Apache Columnar format serialization`
- **Tech stack**: HTTP/2 streaming, Connector libraries, Integration Manager (streaming endpoint), Email Tool
- **Data flow**: Email Tool triggers Audience export + CRM query → Integration Manager streams CRM data (HTTP/2) → Email Tool joins Audience + CRM data on-demand → personalization
- **Business rules**:
  - Queries are pre-built in external system (via connector); no ad-hoc queries supported
  - Email Tool can request named query (e.g., "CEOs of companies with attendees at Event X")
  - Results streamed in columnar format (not flattened)
  - Allows many-to-one joins without flattening APSIS profile model (e.g., attendee → CEO)
- **Configuration**: 
  - Pre-built queries defined in connector (integration-specific)
  - Query parameter schema (not detailed)
- **Integration points**: Email Tool (query initiator), Audience (profile export source), External CRM systems (query execution), Integration Manager, Connector libraries
- **Gotchas**:
  - ⚠️ Ad-hoc queries NOT supported; only pre-built queries available
  - ⚠️ Requires external system to expose relational data in queryable way (not all CRMs support)
  - HTTP/2 streaming adds complexity (connection management, timeout handling)
- **Tribal knowledge**: 
  - Unified Data was designed specifically to avoid flattening CRM relational structure into flat APSIS profile model
  - Streaming architecture prevents memory exhaustion when joining large datasets (e.g., millions of attendees)

### Connector Libraries (System-Specific)
- **Purpose**: Translate between external system APIs and Justin's expected format; isolate all system-specific logic; follow standard contract.
- **Key files**: 
  - `connector-<system-name>` (per integration type, format varies: npm module, Python library, Go plugin, etc.)
  - `Generic Connector API Spec` (latest version)
  - `Generic Connector Implementation Guide`
- **Tech stack**: Language varies (polyglot: Node.js, Python, Go, etc.); HTTP/REST to external system API
- **Data flow**: All Justin workers call connector methods → Connector translates to/from external API → Justin workers receive/send standardized format
- **Business rules**:
  - All system-specific logic isolated in connector (no "CRM-isms" in Justin workers)
  - Follows standard contract (defined by Generic Connector API Spec)
  - Implements standard operations: schema discovery, contact sync, field mapping, consent mapping, activity creation, webhook registration
  - For Unified Data: implements pre-built named queries
- **Configuration**: 
  - Connector endpoint configuration
  - Authentication (CRM-specific)
  - System-specific field mappings (e.g., Silhouette entity for FSC Corporate leads)
- **Integration points**: Integration Manager, Delta Sync Worker, Full Sync Producer/Consumer, Outbound Worker, Unified Data, External system APIs
- **Gotchas**:
  - ⚠️ Connector quality directly impacts integration reliability; weak connector = frequent failures
  - ⚠️ Pagination implementation varies per CRM (some support batch size tuning, some don't)
  - ⚠️ Error messages from CRM may not be standardized; connector must translate to meaningful errors
- **Tribal knowledge**: 
  - Good connector design requires deep CRM system knowledge; "isolate surprises to small connector code"
  - Supplier-owned connectors (expensive hourly billing + SLA costs) were replaced with generic connector partnership model (partners implement, APSIS focuses on infrastructure)

### Generic Connector (Partnership Model)
- **Purpose**: Standard contract that external system partners implement on their side; enables APSIS to interact with system via minimal config.
- **Key files**:
  - `Generic Connector API Spec` (current version, versioned in S3)
  - `Generic Connector Implementation Guide`
  - `Partner agreements` (templates)
- **Tech stack**: REST/HTTP API spec, JSON payload format
- **Data flow**: APSIS publishes spec → Partner implements on their system → APSIS config minimal (just authentication) → All API calls use generic spec contract
- **Business rules**:
  - Partners implement standard endpoints (schema discovery, contact sync, consent mapping, activity creation, webhook registration)
  - No CRM-specific logic on APSIS side for generic connector partners
  - Reduces APSIS maintenance burden; partners own system-specific logic
- **Configuration**: Minimal (authentication credentials, endpoint base URL)
- **Integration points**: External system partners (Sideshop, Intermail, etc.), Integration Manager, All Justin components
- **Gotchas**:
  - ⚠️ Partner-built connectors may have quality variance (some partners more capable than others)
  - ⚠️ Spec versioning and backward compatibility must be managed
  - ⚠️ Partners may delay or refuse to implement optional features
- **Tribal knowledge**: 
  - Generic Connector is strategic shift away from APSIS owning all integration logic
  - Scales APSIS to unlimited external systems without proportional engineering cost

### Clear Integration Endpoint
- **Purpose**: Forcefully delete an integration from Apsis side when normal uninstallation fails due to external system errors or credential changes.
- **Key files**: 
  - `integrations.apsis.one/accounts/{accountId}/sections/{sectionId}/integrations/{integrationId}` (HTTP DELETE endpoint)
  - `Delete Integration Secret Key` (Secrets Manager)
- **Tech stack**: HTTP REST API, RDS, Secrets Manager
- **Data flow**: HTTP DELETE request with Authorization header → Validate Delete Integration Secret Key → Load integration credentials from RDS → Call uninstallation with ignore_external_system_errors flag → Remove DB entries and credentials → Return success regardless of external system response
- **Business rules**:
  - Requires "holy trinity" of parameters: Account ID, Section ID, Integration ID (all must be correct)
  - Uses Delete Integration Secret Key for authorization (not customer credentials)
  - Applies `ignore_external_system_errors` flag to skip external CRM errors
  - Only appropriate when normal uninstallation fails (no users in account, account terminated, CRM misconfigured)
- **Configuration**: 
  - Authorization header: `Authorization: [Delete Integration Secret Key]`
  - URL parameters: account ID, section ID, integration ID
  - External system error handling: `ignore_external_system_errors=true`
- **Integration points**: RDS database, Secrets Manager, Uninstallation flow, External CRM systems
- **Gotchas**:
  - ⚠️ **CRITICAL SECURITY ISSUE**: Delete Integration Secret Key has been exposed in recorded sessions; grants authorization to delete ANY integration
  - ⚠️ Legacy accounts use account names (e.g., 'Meltex Plastics') instead of UIDs for account ID parameter; modern accounts use UID format
  - ⚠️ Integration IDs vary per system (e.g., 'lime', 'fsc_enterprise_12_0' displayed as 'FSC Enterprise 2', 'dynamics')
  - ⚠️ Forced deletion removes Apsis-side data but does NOT delete webhook configurations in customer's CRM; customer must manually clean up (legal/privacy compliance issue)
  - ⚠️ HTTP 401 Unauthorized errors are not well-logged; manual verification of account ID and secret key required
  - ⚠️ External system errors (502 Bad Gateway, timeouts) are expected during forced deletion; proceed anyway with ignore_external_system_errors flag
- **Tribal knowledge**: 
  - Clear Integration endpoint was designed to replace legacy "parameter passing nightmare" where 4-5 flags required through multiple function layers
  - Delete Integration Secret Key should eventually be replaced with delegation key model for better security (not yet prioritized)

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- **Inbound (CRM → APSIS)**: CRM credentials stored in RDS during integration installation; Delta Sync/Full Sync use stored credentials to fetch data
- **Outbound (APSIS → CRM)**: Stored credentials used by Outbound Worker to send batches to external systems
- **Clear Integration**: Uses Delete Integration Secret Key (Secrets Manager) instead of customer credentials; enables forced deletion when customer credentials invalid
- **Future improvement**: Delegation key model (more granular than current secret key approach)

### Error Handling
- **Delta Sync Worker**: Message validation against mappings; errors pushed to DLQ (exact DLQ not specified for delta sync)
- **Full Sync**: If pagination timeout occurs, entire sync fails (no per-page retry); partial data may be processed
- **Outbound Worker**: Retries via DLQ daily; after 14 days, messages silently dropped (no alerting documented)
- **Consent retries**: Only opt-outs (opt-in=false) retried; opt-ins NOT retried (safety measure)
- **Forced deletion**: `ignore_external_system_errors` flag allows deletion even if external system throws errors
- **Lack of alerting**: No documented alert when DLQ messages reach 14-day limit and are about to be dropped

### Logging & Observability
- **Integration logs**: Keyed by account and integration ID format: `[account]_[integration_id]` (e.g., `ACCOUNT123_FSC_ENTERPRISE_2`)
- **Webhook callbacks**: Not logged (privacy concerns); makes post-deletion verification difficult
- **Full Sync progress**: Monitored via CloudWatch metrics (exact metrics not detailed)
- **Kafka consumer lag**: Potential bottleneck indicator if All Sub Worker or Batch Production Worker can't keep up

### Data Privacy & Compliance
- **Outbound mappings**: Can transmit profile attributes to external system; requires customer consent
- **Webhook callbacks post-deletion**: Customer must manually remove webhook configs to prevent data leakage (legal responsibility)
- **DLQ retention**: Messages retained 14 days, then dropped (no audit trail of failures)
- **Webhook logging omission**: By design to avoid logging sensitive customer data, but creates verification gaps

### Scaling & Performance
- **Contact volume baseline**: Historical testing with 2M contacts; current largest customer 8M (4x beyond tested); no known limits documented
- **Pagination**: Contact pagination works (16,000 pages for 8M contacts); consent pagination times out (500-entry batches = 22 sec; 8M = 97+ hours estimated)
- **In-memory optimizations**: Audience export comparison causes memory exhaustion crashes at 7-8M profiles; flagged for removal
- **Batching efficiency**: 200KB batches prevent overwhelming external systems; 1-second timeout prevents excessive latency
- **Kafka partitioning**: By integration; unbalanced load if one integration dominates

### Deployment & Lifecycle
- **On-demand scaling**: Full Sync Manager spins up Producer/Consumer tasks as needed
- **Recurring triggers**: CloudWatch Events for profile list imports (hardcoded 6:00 AM UTC); daily retry at 6:00 AM
- **Secrets rotation**: Delete Integration Secret Key management not documented (rotation policy, audit trail)
- **Database schema**: RDS for relational integration metadata (not DynamoDB)

---

## 5. Business Rules Reference

### Sync Direction & Bidirectionality
| Data Type | Direction | Rationale |
|-----------|-----------|-----------|
| **Field attributes** | Inbound only (CRM → APSIS) | CRM is source of truth for customer data; APSIS avoids overwriting authoritative CRM data |
| **Consent/subscriptions** | Bidirectional (CRM ↔ APSIS) | Opt-in/opt-out changes must sync back to CRM to maintain consistency |
| **Activity creation** | Outbound (APSIS → CRM) | Emails, SMS, forms created in APSIS are synced to CRM as records |
| **Engagement events** | Outbound (APSIS → CRM) | Delivered, opened, clicked events batched and sent back to CRM |

### Sync Conditions (Filtering)
- **Configured**: In APSIS Mappings Manager (e.g., "only sync active contacts")
- **Enforcement**: Ideally server-side by CRM (Generic Connector supports sync_condition_property_enabled flag)
- **Current issue**: ⚠️ FSC Enterprise does NOT implement server-side filtering; APSIS downloads all data and filters locally (performance bottleneck)
- **Reference implementation**: Dynamics Side Shop (Site Shop) successfully filters server-side
- **Impact**: 8M customer with 1.2M eligible contacts after sync conditions must download 8M before filtering

### Profile Creation
- **Inbound sync (Full/Delta)**: Creates new profiles if not found
- **Profile list import**: **Does NOT create profiles; only tags existing profiles**
- **Rationale**: Prevents accidental profile creation from CRM lists; maintains APSIS profile as authoritative

### Integration Lifecycle
- **Install**: User configures mapping, Mappings Manager stores config, subscribes to webhooks
- **Uninstall**: Normal path deletes from external system (requires valid credentials), then from APSIS
- **Forced deletion**: Clear Integration endpoint deletes from APSIS without external system cooperation (using Delete Integration Secret Key)
- **Constraint**: Only one CRM integration per section can be active (due to shared CRM_ID key space)

### Message Ordering & Consistency
- **Delta Sync**: SQS FIFO with CRM_ID as message group ID ensures per-contact ordering (eventual consistency)
- **Buffering during full sync**: Delta messages buffered with visibility timeout; replayed in order after full sync completes
- **Outbound**: SQS FIFO maintains per-integration ordering (via batching) but messages may be processed out of contact order
- **Compromise**: Eventual consistency guaranteed; immediate consistency NOT guaranteed

### Batching & Delivery
- **Engagement events**: Batched by All Sub Worker + Batch Production Worker (200KB or 1 second); sent as single batch to external system
- **Consent events**: NOT batched (sent individually) to ensure immediate delivery
- **Retry on failure**: Engagement batches moved to DLQ if failed after retries; redrive daily by Retry Driver Lambda
- **Consent retries**: Only opt-outs (opt-in=false) retried from DLQ (opt-ins NOT retried to avoid false positives)

### Outbound Mappings
- **Scope**: Account-level only (not form-specific; all form submissions use same mappings)
- **Feature flag**: Two-level check required: APSIS integration-level flag + external system instance-level capability
- **Unavailability**: If either flag disabled, outbound mappings feature unavailable for account
- **Rework needed**: Form-specific mappings would require significant form submission event pipeline rework

### Webhook Lifecycle
- **Auto-subscribe**: When mappings created in Mappings Manager
- **Auto-update**: Webhooks updated whenever mappings change (cache invalidation + subscription update)
- **External system support**: Not all CRM systems support webhooks (legacy systems may require polling)
- **Post-deletion cleanup**: After forced deletion, customer must manually remove webhook configs from their CRM

### Recurring Imports
- **Schedule**: CloudWatch Events trigger at 6:00 AM UTC (hardcoded; not configurable without code change)
- **Scope**: Profile list imports only (static and dynamic lists)
- **Semantics**: Tag existing profiles; does not create new profiles
- **List member removal**: If profile removed from list in CRM, tag removed on next import

### Data Retention & Cleanup
- **DLQ messages**: Retained for 14 days; no automatic cleanup after 14 days (improvement noted)
- **Webhook logs**: NOT logged post-deletion (prevents logging sensitive data, but creates verification gaps)
- **Profile list tags**: Removed on next import if profile no longer on list
- **Audit trail**: Zero-point stories for customer support issues (even quick fixes) provide historical record

### CRM Key Space (⚠️ Architectural Debt)
- **Current**: Single CRM_ID key space for all CRM integrations per section; prevents multiple active CRM integrations
- **Constraint**: "Only one CRM integration per section can be active at a time" (unless using separate key spaces)
- **Proper solution**: System-specific key spaces (Dynamics_ID, Salesforce_ID, FSC_ID, etc.)
- **Historical debt**: Established during "Jonas Super Urgent Release" in early APSIS development; never properly refactored
- **Non-CRM integrations**: Ecommerce (Shopify, WooCommerce) and loyalty use their own key spaces (can be multiple per section)

---

## 6. Known Issues & Workarounds

### Performance & Scaling Issues

| Issue | Severity | Affected Component | Workaround | Status |
|-------|----------|-------------------|-----------|--------|
| **Full sync times out for large contact datasets (8M contacts)** | HIGH | Full Sync Consumer, FSC Enterprise pagination | Engage CRM on pagination/caching optimization; implement server-side sync condition filtering | Open; blocking customer |
| **Consent pagination times out (HTTP 504 Gateway Timeout)** | HIGH | Full Sync Producer, FSC Enterprise proxy | Investigate FSC Enterprise pagination implementation; potentially optimize batch size or timeout settings | Open; impacts sync throughput |
| **Sync conditions filter locally after downloading all CRM data** | HIGH | Sync Conditions (v1), Full Sync Consumer, FSC Enterprise | Implement server-side sync condition filtering (CRM-side); APSIS side ready (sync_condition_property_enabled flag) | Open; awaiting CRM implementation |
| **In-memory audience export comparison causes memory exhaustion crashes** | HIGH | Audience export optimizer, Full Sync Consumer | Remove in-memory audience export optimization; stream all CRM data directly to SQS without pre-comparison | Flagged for removal; needs prioritization |
| **Contact volume baseline 2M; largest customer 8M (4x beyond tested)** | MEDIUM | Platform documentation | Add documentation noting tested/supported with 2M contacts; larger volumes (8M) require server-side sync condition filtering | Open; blocks future large customers |
| **No maximum contact volume limitations documented** | MEDIUM | Platform documentation, customer expectations | Document scaling limits and provide guidance for large customer onboarding | Open |

### Integration Lifecycle Issues

| Issue | Severity | Affected Component | Workaround | Status |
|-------|----------|-------------------|-----------|--------|
| **Clear Integration endpoint error handling (401 Unauthorized)** | MEDIUM | Clear Integration Endpoint, Authorization validation | Manually verify account ID format (legacy name vs UID) and Delete Integration Secret Key before retrying | Open; poor error logging |
| **Forced deletion leaves webhook configs in customer's CRM** | HIGH | Uninstallation Flow, Webhook Management, Legal/Privacy | Customer must be explicitly informed to manually delete webhook configurations in their CRM | Open; requires customer action |
| **Squid proxy configuration edge case: database deletion hangs** | MEDIUM | Uninstallation Flow, RDS Database | Use ignore_external_system_errors flag in Clear Integration endpoint to bypass database deletion hang | Workaround applied; root cause unresolved |
| **No system logging of webhook data post-deletion** | MEDIUM | Webhook Management, Logging, Privacy | Manually check integration logs; propose monitoring enhancement to log callbacks (with privacy considerations) | Open; design trade-off |
| **Delete Integration Secret Key exposed in recorded sessions** | HIGH | Delete Integration Secret Key, Security | Restrict access to recording; consider redaction or re-recording; monitor for unauthorized use | URGENT; mitigation needed |

### Connector & Configuration Issues

| Issue | Severity | Affected Component | Workaround | Status |
|-------|----------|-------------------|-----------|--------|
| **E-Deal folder names show internal name (FSC_CORPORATE) in existing customer accounts** | MEDIUM | Apsis One Folder Creation Engine | Manual folder rename by support or internal migration using delegated keys; ~2-3 customers affected | Open; low-priority cleanup |
| **Integration ID vs. Display Name confusion in logs** | LOW | System-wide logging, Audience domain | Always reference connector configuration to understand identifier context; search logs for integration ID not display name | Ongoing; documentation gap |
| **Not all CRM systems have lead concept** | LOW | Lead Entity Handler, Connector Configuration | Check additional_entities config to know what a connector supports (e.g., Silhouette for FSC Corporate) | Ongoing; per-connector awareness |
| **Postman collections incomplete for legacy connectors** | LOW | Testing/debugging tools | Use Generic Connector collection for isolated testing when legacy collections incomplete | Workaround; improvement needed |

### Customer Support & Process Issues

| Issue | Severity | Affected Component | Workaround | Status |
|-------|----------|-------------------|-----------|--------|
| **Support requests bypass formal ticket routing (Submariner)** | MEDIUM | Support ticket routing process | Enforce unified flow: help channel → product escalation → Submariner formal ticket → R&D pickup; do not monitor alternative channels | Ongoing; discipline required |
| **R&D picks up work outside formal Submariner tickets** | MEDIUM | Support ticket routing process, Visibility | Redirect to official channels; all work must go through formal ticketing process for visibility | Ongoing; precedent management |
| **DLQ messages retained 14 days then silently dropped** | MEDIUM | Outbound Worker, DLQ retention | Implement alerting when messages approach 14-day limit; propose longer retention or manual intervention | Open; improvement needed |

### Data & Metadata Issues

| Issue | Severity | Affected Component | Workaround | Status |
|-------|----------|-------------------|-----------|--------|
| **CRM_ID key space prevents multiple CRM integrations per section** | MEDIUM | Delta Sync Worker, Full Sync Consumer | Use Non-CRM integrations (Shopify, WooCommerce) with separate key spaces; refactor to system-specific key spaces (architectural debt) | Open; architectural redesign needed |
| **Confusing relationship between integration ID and display name** | LOW | Integration Manager, logging | E-Deal (display name) = FSC_CORPORATE (integration ID); both appear in different contexts; not a bug but confusing | Ongoing; documentation |

---

## 7. Glossary

| Term | Definition | Domain |
|------|-----------|--------|
| **All Sub Worker** | ECS task that consumes Audience engagement events, validates integration installed, routes to Kafka per integration | Core component |
| **Audience** | APSIS One's central profile store and analytics platform; receives inbound synced data, sources outbound events | External system |
| **Batch Production Worker** | ECS task that batches Kafka events (200KB or 1s timeout) and puts on Outbound Queue | Core component |
| **Bidirectional consent** | Consent/subscription changes sync CRM ↔ APSIS (unlike field data which is CRM → APSIS only) | Business rule |
| **Clear Integration Endpoint** | HTTP DELETE endpoint that forcefully removes integration from APSIS using Delete Integration Secret Key | Component |
| **Consent mapping** | Configuration linking CRM consent lists to APSIS subscription lists for tracking consent changes | Configuration |
| **Consent Outbound** | Process sending consent changes (opt-in/opt-out) from APSIS back to CRM systems individually (not batched) | Feature |
| **Consent retries** | Only opt-outs (opt-in=false) are retried from DLQ; opt-ins NOT retried to avoid false positives | Business rule |
| **Connector libraries** | Per-CRM translation layer implementing standardized contract (Justin format); isolates system-specific logic | Component |
| **CRM_ID** | Single key space for all CRM integrations per section (architectural debt; should be system-specific) | Architecture |
| **Delta sync** | Real-time synchronization of contact/consent updates via webhooks from external system | Feature |
| **Delta Sync Buffer Queue** | SQS FIFO queue that buffers real-time messages during full sync, replays after completion | Component |
| **Delta Sync Worker** | ECS task consuming real-time webhook messages, applying mappings, sending to Audience | Component |
| **Delete Integration Secret Key** | Secrets Manager key authorizing Clear Integration endpoint requests (without requiring customer credentials) | Security |
| **Display name** | Customer-facing label for connector (e.g., "E-Deal"); mutable, used for folder names | Configuration |
| **E-Deal** | Display name for FSC Corporate connector; integration ID remains FSC_CORPORATE | Naming |
| **Engagement events** | Activity tracking events (delivered, opened, clicked) sent from APSIS to CRM in batches | Data type |
| **Eventually consistent** | Delta sync guarantees per-contact ordering but not immediate consistency across all contacts | Guarantee |
| **Full sync** | Complete synchronization of all contacts/consents from CRM to APSIS, including filtering and comparison | Feature |
| **Full Sync Consumer** | ECS task consuming Full Sync Queue messages, applying mappings, sending to Audience | Component |
| **Full Sync Manager** | ECS task orchestrating full sync operation; spins up Producer/Consumer stacks | Component |
| **Full Sync Producer** | ECS task fetching paginated data from CRM, converting to Justin format, queuing batches | Component |
| **Generic Connector** | Standard REST API contract that partner systems implement; enables APSIS to interact with system via minimal config | Architecture |
| **Holy Trinity** | Three required parameters to uniquely identify an integration: Account ID, Section ID, Integration ID | Concept |
| **Ignore external system errors** | Flag in uninstallation flow that bypasses CRM errors and proceeds with APSIS-side deletion | Feature |
| **Integration ID** | Internal logical name for connector (e.g., FSC_CORPORATE, lime); immutable API contract | Configuration |
| **Integration Manager** | ECS long-running task providing UI/backend interface for schema discovery, connector routing, feature checks | Component |
| **Justin** | Generic integration middleware layer; combines generic infrastructure + system-specific connectors | Architecture |
| **Justin's Laws** | Validation rules enforced by Mappings Manager during mapping creation (exact rules not detailed) | Business rule |
| **Jonas Super Urgent Release** | Historical APSIS development phase during which CRM_ID single key space was established (debt since) | History |
| **Lead entity** | CRM entity representing prospects/leads separate from contacts (not all CRMs support; e.g., Silhouette in FSC Corporate) | Data model |
| **Mapping cache** | In-memory cache of field/consent mappings; invalidated when mappings change | Infrastructure |
| **Mappings Manager** | ECS task managing field/consent mappings, validation, webhook subscription, cache invalidation | Component |
| **Message group ID** | SQS FIFO parameter (CRM_ID) ensuring per-contact ordering while allowing parallel processing | Configuration |
| **Maxo** | Historical development project during which sync conditions architecture decisions were made | History |
| **On-demand scaling** | Full Sync Manager spins up Producer/Consumer ECS tasks as needed (not always-on) | Architecture |
| **Opt-in/Opt-out** | Consent states; only opt-outs (opt-in=false) retried from DLQ (opt-ins NOT retried) | Business rule |
| **Outbound mappings** | Field mappings in reverse direction (APSIS attributes → External system fields); account-level only | Configuration |
| **Outbound Manager** | ECS task coordinating activity creation in APSIS with external system records | Component |
| **Outbound Queue** | SQS FIFO queue holding batches for delivery to external systems | Component |
| **Outbound Worker** | ECS task consuming Outbound Queue, sending batches via connector library, handling DLQ/retries | Component |
| **Pagination** | Mechanism for retrieving large CRM datasets in batches; contact pagination works; consent pagination times out | Infrastructure |
| **Polling** | Alternative to webhooks for systems that don't support real-time updates (less ideal) | Feature |
| **Profile list import** | Importing static/dynamic lists from CRM; tags existing profiles; does NOT create profiles | Feature |
| **Profile List Sync** | ECS task managing recurring profile list imports (triggered daily at 6:00 AM UTC) | Component |
| **RDS** | Relational Database Service storing integration metadata, credentials, configurations, webhooks | Infrastructure |
| **Recurring import** | Profile list imports triggered automatically (hardcoded to 6:00 AM UTC; not configurable without code change) | Configuration |
| **Retry Driver Lambda** | Daily Lambda function (6:00 AM UTC) that redrive DLQ messages to Outbound Queue | Component |
| **Secrets Manager** | AWS secrets storage for Delete Integration Secret Key, API credentials (reference only; actual storage in RDS) | Infrastructure |
| **Silhouette** | Lead entity name in FSC Corporate connector | Data model |
| **Sync condition** | Filter rule (e.g., "only sync active contacts") configured in APSIS, should be enforced server-side by CRM | Business rule |
| **Sync direction** | Field data: CRM → APSIS (inbound only); Consent: bidirectional | Business rule |
| **Unified Data** | HTTP/2 streaming endpoint enabling on-demand relational CRM queries during email personalization | Feature |
| **Visibility timeout** | SQS parameter used to delay buffered delta messages until full sync completes | Configuration |
| **Webhook** | External system callback mechanism for delivering real-time contact/consent updates to APSIS | Integration |
| **Zeero-point story** | Support issue tracked on story board with zero points estimation; provides visibility and historical record | Process |

---

## 8. Confidence Notes

### High Confidence (Multiple Sources, Aligned)
- ✅ Architecture overview (Benjamin + Erik sessions)
- ✅ Integration Manager, Mappings Manager, Delta/Full Sync components (Benjamin)
- ✅ Batch production and outbound delivery (Benjamin)
- ✅ Clear Integration endpoint and deletion flow (Erik)
- ✅ Sync direction business rules (field inbound, consent bidirectional)
- ✅ Full sync buffering strategy (Delta Sync Buffer Queue)
- ✅ SQS FIFO ordering and message group ID usage
- ✅ Display name vs. integration ID distinction (Benjamin + connector PR session)
- ✅ Known performance issues: consent pagination timeouts, in-memory optimization, large dataset scaling

### Medium Confidence (Single Source, Reasonable Detail)
- 🟡 Unified Data HTTP/2 streaming mechanics (Benjamin; high detail but single source)
- 🟡 Sync conditions architecture (local filtering vs. server-side; Efficy Enterprise session describes issue but not comprehensive design)
- 🟡 Kafka partitioning strategy and Batch Production Worker batching logic (Benjamin; specific numbers mentioned but no design rationale)
- 🟡 Profile List Sync implementation (Benjamin; less detail than delta/full sync)
- 🟡 Consent retry filtering logic (only opt-outs retried) — described as safety measure but exact rationale not detailed
- 🟡 On-demand scaling mechanics (Full Sync Manager spin-up) — described but not detailed
- 🟡 Postman collections as dual-purpose debugging tools (Erik; testing tool mentioned but usage not comprehensive)

### Lower Confidence (Incomplete, Contradictions, or Gaps)
- ⚠️ **Exact error handling paths** — Different components describe errors differently; consolidated view incomplete
  - Delta Sync Worker DLQ destination not explicitly stated
  - Full Sync Consumer error cascading not fully documented
  - Authorization validation error logging gaps (401 errors not well-logged)
- ⚠️ **Retry mechanisms** — Exact retry counts, exponential backoff, jitter not specified for any component
- ⚠️ **Cloudwatch metrics** — Full Sync Manager monitors CloudWatch for Audience catch-up, but exact metrics not named
- ⚠️ **Message group ID implications** — SQS FIFO with CRM_ID ensures per-contact ordering but potential parallelization bottleneck not analyzed
- ⚠️ **Kafka consumer lag monitoring** — Mentioned as potential bottleneck indicator but no alerting strategy documented
- ⚠️ **Secrets rotation** — Delete Integration Secret Key rotation policy, audit trail not documented
- ⚠️ **RDS schema design** — Tables, relationships, indexes not detailed

### Known Gaps & Disagreements
| Gap | Source(s) | Impact |
|-----|-----------|--------|
| **Justin's Laws validation rules** | Benjamin mentions "Justin's Laws" enforced by Mappings Manager, but exact rules not detailed | Cannot validate mapping changes without knowing rules |
| **Full Sync timeout handling** | Benjamin describes overall architecture; Efficy Enterprise describes pagination timeout as full sync failure | Unclear if partial data is retained, how recovery works |
| **DLQ cleanup strategy** | Erik mentions 14-day retention then silent drop; no documented alerting or manual intervention process | Data loss not visible; no operational safeguards |
| **Audience catch-up detection** | Benjamin mentions CloudWatch monitoring for Audience to catch up; exact metric/threshold not named | Cannot replicate Full Sync Manager behavior |
| **Mapper cache invalidation timing** | Benjamin describes invalidation on mapping change; exact timing (immediate vs. delayed) unclear | Potential stale mapping window not quantified |
| **Integration Manager conversion from Lambda** | Benjamin mentions Lambda → ECS conversion reason (schema caching); no mention of implementation date or migration details | Historical context incomplete |
| **Display name folder migration for E-Deal** | Benjamin mentions ~2-3 customers; Erik mentions migration strategy with delegated keys | Prioritization, timeline unclear |
| **CRM key space refactoring scope** | Multiple sources acknowledge architectural debt; no proposed timeline or implementation approach | Blocking multiple CRM integrations per section |
| **Performance baselines at scale** | Benjamin: 2M tested; Efficy Enterprise: 8M customer fails; no systematic load testing results | Scaling predictions unreliable |

### Explicit Uncertainties Flagged in Original Data
- ⚠️ FSC Enterprise sync condition enforcement: Does it actually support server-side filtering, or is it planned?
- ⚠️ Profile list memory exhaustion: Was this fixed, or is workaround still in place?
- ⚠️ Dynamics 365 CORS/domain requirement: What does customer actually need?
- ⚠️ CRM implementer issue routing: Integration domain or Audience domain?

### Confidence by Component
| Component | Confidence | Rationale |
|-----------|----------|-----------|
| **Integration Manager** | HIGH | Multiple sessions, consistent across sources |
| **Mappings Manager** | HIGH | Clear purpose and data flow; validation rules less detailed |
| **Delta Sync Worker** | HIGH | Well-described; FIFO ordering well-understood |
| **Full Sync (overall)** | MEDIUM-HIGH | Architecture clear; error handling/recovery gaps |
| **Clear Integration Endpoint** | HIGH | Purpose, authorization, impact well-documented by Erik |
| **Unified Data** | MEDIUM | Concept clear; HTTP/2 streaming mechanics not fully detailed |
| **Profile List Sync** | MEDIUM | Less detail than delta/full; memory issues mentioned but resolution unclear |
| **Error Handling (cross-system)** | LOW-MEDIUM | Fragmented descriptions across components; no unified error strategy |
| **Performance & Scaling** | LOW-MEDIUM | Issues well-documented; solutions/baselines unclear |
| **Security** | MEDIUM | Delete Integration Secret Key exposed and documented; other auth/secrets gaps |

---

## Tribal Knowledge Consolidated

### Gotchas (Common Trip-Ups)
1. **Integration ID vs. Display Name**: "E-Deal" (display name) = "FSC_CORPORATE" (integration ID). Search logs for integration ID. Changing display name does NOT change integration ID (embedded in API contracts).
2. **CRM_ID Key Space Limit**: Only one CRM integration per section can be active (unless refactored to system-specific key spaces). Non-CRM integrations (Shopify, WooCommerce) use separate key spaces.
3. **Sync Conditions Are Theater**: Configured in APSIS but only effective if CRM implements server-side filtering. FSC Enterprise doesn't; APSIS downloads all 8M contacts and filters locally.
4. **Local Filtering Bottleneck**: 8M CRM contacts → filter locally to 1.2M eligible → sync to APSIS. Scales poorly (97+ hours estimated for consent pagination).
5. **In-Memory Optimization Time Bomb**: Works for current customers (2M baseline) but crashes at 7-8M profiles with multiple consents. Remove and stream through SQS instead.
6. **Contact Pagination Works; Consent Pagination Fails**: 16,000 contact pages complete fine; 500-entry consent batches timeout (22 sec each; 8M = 97+ hours). Issue is CRM proxy/caching, not APSIS.
7. **Forced Deletion ≠ CRM Cleanup**: Clear Integration removes Apsis-side data but leaves webhook configs in customer's CRM. Customer must manually delete webhooks (legal/privacy compliance).
8. **Delete Integration Secret Key Exposed**: Security issue in recorded sessions; grants authorization to delete ANY integration. Requires redaction/mitigation.
9. **Legacy Accounts Use Names**: Account ID parameter: legacy accounts use names (e.g., "Meltex Plastics"); modern accounts use UID. Must verify format before Clear Integration.
10. **Integration ID Formats Vary**: 'lime', 'fsc_enterprise_12_0' (internal) displayed as 'FSC Enterprise 2', 'dynamics'. Entering wrong format = 404 error.
11. **Consent Retry Asymmetry**: Only opt-outs (opt-in=false) retried; opt-ins NOT retried (safety measure). Failed opt-ins dropped after 14 days.
12. **Postman Collections as Debugging Tools**: Dual purpose (testing + debugging); Generic Connector collection most flexible for isolated testing.
13. **Kafka Partitioning Imbalance**: Routed by integration; one dominant integration can saturate consumer.
14. **Audience Catch-Up Mystery**: Full Sync Manager waits for Audience to catch up (via CloudWatch metrics) before signaling success. Timing non-obvious.
15. **Message Buffering During Full Sync**: Delta messages buffered with visibility timeout; replayed in order. Visibility timeout expiry before full sync completion = double-processing risk.

### Historical Context
- **CRM_ID Single Key Space (Debt)**: Established during "Jonas Super Urgent Release" in early APSIS dev; never refactored. Prevents multiple CRM integrations per section.
- **Supplier Connectors → Generic Connector Partnership Model**: Supplier-owned connectors were expensive (hourly billing + SLA costs), slow (external company controls iteration), and prone to surprises. Generic Connector model delegates to partners; APSIS focuses on infrastructure.
- **Lambda → ECS for Integration Manager**: Lambda cold starts and timeout constraints were problematic for schema discovery. ECS long-running task with persistent state better.
- **Sync Conditions Compromise**: Stakeholders wanted configuration in CRM (where data lives and relational context exists), but no one invested in CRM-side UI/backend. APSIS compromise: config in APSIS, enforcement in CRM. Side Shop (Dynamics) successfully implements server-side; FSC Enterprise doesn't.
- **Profile List Memory Failures**: Historical failures on large lists before edge case handling. Recent fixes; exact status unclear.
- **Maxo Project Impact**: Architectural decisions on sync conditions and outbound mappings made during Maxo development; some suboptimal compromises (account-level mappings instead of form-specific).
- **Postman Collections Evolution**: FCC Enterprise and Lime collections incomplete/legacy. Generic Connector collection most comprehensive and valuable.
- **E-Deal Folder Name Confusion**: ~2-3 existing customers have folders named "FSC_CORPORATE" (internal) instead of "E-Deal" (display name). Migration strategy: internal delegated keys, not customer credentials.
- **Zero-Point Stories for Visibility**: Small teams (3-4) use rotating support schedule; larger teams must track even 5-minute fixes as zero-point stories for searchability and institutional knowledge.
- **Delete Integration Secret Key Future Plan**: Should be replaced with delegation key model (more granular); not yet prioritized.

### Optimization Opportunities
1. **Server-Side Sync Condition Filtering**: Biggest win. Coordinate with CRM to implement sync_condition_property_enabled endpoint. Reduces data transfer 8M → 1.2M.
2. **Consent Pagination Optimization**: FSC Enterprise proxy/caching issue. Investigate batch size tuning, timeout extension, or query optimization. Quick win potential before full filtering refactor.
3. **Remove In-Memory Audience Export Optimization**: Stream directly to SQS without pre-comparison. Prevents memory exhaustion crashes.
4. **Unified Error Handling Strategy**: Consolidate error paths across components; implement consistent retry, DLQ, and alerting.
5. **Kafka Consumer Lag Alerting**: Monitor All Sub Worker and Batch Production Worker for lag; implement alerting when exceeds threshold.
6. **DLQ Cleanup Strategy**: Implement alerting 1-2 days before 14-day expiry; allow manual intervention or longer retention.
7. **Secrets Rotation Audit Trail**: Document Delete Integration Secret Key rotation policy; implement audit logging.
8. **Load Testing at Scale**: Systematic testing with 4M, 8M, 16M contacts to establish baselines and identify breakpoints.
9. **Documentation of Scaling Limits**: Document tested/supported contact volumes and guidance for large customer onboarding.
10. **CRM Key Space Refactoring**: Implement system-specific key spaces (Dynamics_ID, Salesforce_ID) to enable multiple CRM integrations per section.
11. **Form-Specific Outbound Mappings**: Redesign to support per-form custom field mappings (currently account-level). Medium effort.
12. **Webhook Logging with Privacy**: Enable logging of webhook callbacks post-deletion with PII redaction; improves post-deletion verification.

### Red Flags for Review
- ⚠️ **Delete Integration Secret Key exposed in recorded sessions**: URGENT mitigation required
- ⚠️ **Large customer (8M contacts) hitting hard scaling limits**: No documented path to support; architectural rework needed
- ⚠️ **14-day silent DLQ cleanup**: No alerting; data loss not visible
- ⚠️ **Consent pagination timeouts cascade full sync failures**: Single point of failure in producer step
- ⚠️ **CRM_ID key space architectural debt**: Blocks product feature (multiple CRM integrations)
- ⚠️ **Support ticket bypass precedent**: Lack of discipline in formal ticketing causing visibility loss

---

## Summary & Recommendations

### What You Own
The Integrations domain manages the complete data synchronization lifecycle between APSIS One and external CRM/business systems. You are responsible for:
- **Inbound data ingestion** (Delta Sync, Full Sync, Profile Lists)
- **Outbound event delivery** (Activities, Engagement Events, Consent)
- **Field and consent mapping** enforcement and lifecycle
- **System-agnostic infrastructure** (Justin middleware) + system-specific connectors
- **Error handling, retry, and DLQ management**
- **Integration installation, configuration, and deletion**

### Critical Issues to Address
1. **Performance bottleneck**: Consent pagination timeouts + local filtering of 8M contacts blocking large customer sync
2. **Security**: Delete Integration Secret Key exposure in recorded sessions
3. **Scaling**: No documented path for customers larger than 2M contacts (tested baseline)
4. **Debt**: CRM_ID single key space prevents multiple CRM integrations per section

### Quick Wins (Weeks)
- Engage FSC Enterprise on consent pagination optimization (batch size, timeouts, caching)
- Redact/mitigate Delete Integration Secret Key exposure in recorded content
- Implement DLQ cleanup alerting (before 14-day expiry)
- Document scaling limits (2M tested, 8M at limit, 16M unknown)

### Medium-Term (Months)
- Implement server-side sync condition filtering (coordinate with CRM partners)
- Remove in-memory audience export optimization; stream through SQS
- Consolidate error handling strategy across all components
- Add Kafka consumer lag alerting

### Long-Term (Quarters)
- Refactor CRM_ID to system-specific key spaces (enable multiple CRM integrations)
- Implement form-specific outbound mappings
- Replace Delete Integration Secret Key with delegation key model
- Systematic load testing at scale (4M, 8M, 16M contacts)

---

**Last Updated**: 2026-03-19  
**Sources**: 4 KT sessions (Benjamin, Erik, Efficy Enterprise issue, Connector PR discussion)  
**Confidence Level**: HIGH for architecture; MEDIUM for error handling and scaling specifics  
**Known Gaps**: Exact metrics/thresholds, retry strategies, secrets rotation policy, RDS schema details