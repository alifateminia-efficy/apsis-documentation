---
title: Apsis One - Integrations Domain Knowledge Base
generated: 2026-03-24T10:47:12.052Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-sonnet-4-6
source: KT session transcripts (6 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One - Integrations Domain Knowledge Base](#apsis-one---integrations-domain-knowledge-base)
  - [1. Domain Overview](#1-domain-overview)
  - [2. Architecture Map](#2-architecture-map)
    - [Components](#components)
    - [Infrastructure](#infrastructure)
    - [Data Flow (Inbound)](#data-flow-inbound)
    - [Data Flow (Outbound)](#data-flow-outbound)
    - [Data Flow (Forms → CRM)](#data-flow-forms-crm)
  - [3. Module Reference](#3-module-reference)
    - [Justin (Integration Middleware)](#justin-integration-middleware)
    - [Integration Manager](#integration-manager)
    - [Mappings Manager](#mappings-manager)
    - [Delta Sync Worker](#delta-sync-worker)
    - [Full Sync Manager](#full-sync-manager)
    - [Full Sync Producer](#full-sync-producer)
    - [Full Sync Consumer](#full-sync-consumer)
    - [Profile List Sync Manager](#profile-list-sync-manager)
    - [All Sub Queue (Audience Subscription Queue)](#all-sub-queue-audience-subscription-queue)
    - [All Sub Worker](#all-sub-worker)
    - [Batch Production Worker](#batch-production-worker)
    - [Outbound Worker](#outbound-worker)
    - [Retry Driver Lambda](#retry-driver-lambda)
    - [Connector Libraries](#connector-libraries)
    - [Base Installer](#base-installer)
    - [Installation Manager](#installation-manager)
    - [Profile Merge Worker](#profile-merge-worker)
    - [Broker Service](#broker-service)
    - [Audience (Apsis Profile Store) — External Dependency](#audience-apsis-profile-store-external-dependency)
    - [Athena (Data Warehouse) — External Dependency](#athena-data-warehouse-external-dependency)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication](#authentication)
    - [Error Handling](#error-handling)
    - [Logging / Observability](#logging-observability)
    - [Deployment](#deployment)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Integration Constraints](#integration-constraints)
    - [Field Mappings](#field-mappings)
    - [Keyspace Rules](#keyspace-rules)
    - [Profile Identification](#profile-identification)
    - [Outbound Batching](#outbound-batching)
    - [Full Sync](#full-sync)
    - [Dead Letter Queue](#dead-letter-queue)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence (verified across multiple sessions)](#high-confidence-verified-across-multiple-sessions)
    - [Medium Confidence (single source or partially corroborated)](#medium-confidence-single-source-or-partially-corroborated)
    - [Low Confidence / Unresolved ⚠️](#low-confidence-unresolved-)
    - [Conflicts Resolved](#conflicts-resolved)

---

# Apsis One - Integrations Domain Knowledge Base

## 1. Domain Overview

The Apsis One Integrations domain is responsible for bidirectional data synchronization between Apsis One (its profile store "Audience," marketing automation, forms, and email tools) and external systems (CRMs, e-commerce platforms, loyalty systems, third-party SaaS). The domain encompasses the middleware layer "Justin," which provides generic infrastructure and connector routing; inbound sync pipelines (full sync and real-time delta sync); outbound event batching pipelines; profile keyspace management and lifecycle; consent synchronization; form-submission-to-CRM event flows; and disaster recovery infrastructure. Its boundaries end at the Audience service API (owned by another team) and at external system APIs (owned by partners/suppliers). The domain does NOT own the Audience profile store, Athena data warehouse, or the Maxo email tool—it integrates with them.

---

## 2. Architecture Map

### Components
- **Justin (Integration Middleware)** — Generic infrastructure hub; routes all integration traffic
- **Integration Manager** — Schema fetch + connector routing gateway (ECS task)
- **Mappings Manager** — Stores field/consent/sync-condition mappings with caching
- **Delta Sync Worker** — Real-time webhook consumer → Audience (ECS, always-on)
- **Full Sync Manager** — Orchestrates bulk sync jobs; spins up Producer/Consumer on demand
- **Full Sync Producer** — Paginates external system → Full Sync Queue (SQS FIFO)
- **Full Sync Consumer** — Full Sync Queue → Mappings Manager → Audience
- **Profile List Sync Manager** — Manages static/dynamic list imports; adds tags to profiles
- **All Sub Queue** — Central SQS FIFO collecting all Audience events (sent, opened, consent, MA notes)
- **All Sub Worker** — Validates integration install, routes to Kafka
- **Batch Production Worker** — Kafka consumer; groups by integration; batches into 200KB/1s chunks
- **Outbound Worker** — Sends batches to external systems via connector libraries; DLQ on failure
- **Retry Driver Lambda** — Daily 6:00 AM UTC; redrives DLQ → Outbound Queue
- **Connector Libraries** — Per-system adapters (one per external system type)
- **Base Installer** — Common CRM connector base class; defines keyspace creation and shared functions
- **Installation Manager** — Creates keyspaces on CRM install; registers form event listeners
- **Profile Merge Worker** — Merges profiles across keyspaces (form submissions, CRM merge requests, keyspace bridging)
- **Audience** — External: Apsis profile store; receives inbound synced profiles; emits events to All Sub Queue
- **Athena** — External: data warehouse; joins Audience exports with Integration streams
- **Broker Service** — Decrypts generic connector credentials (KMS) and routes requests via Squid Proxy

### Infrastructure
- SQS FIFO Queues: Delta Sync, Full Sync, Outbound, All Sub
- SQS Standard Queue: Profile List Sync, Dead Letter Queue (DLQ)
- Kafka Partition: buffer between All Sub Worker and Batch Production Worker
- AWS RDS: primary persistent store
- Redis: caching
- AWS KMS: credential encryption for generic connectors
- AWS Secrets Manager + Parameter Store: credentials and shared config
- Squid Proxy: outbound network routing
- CloudFormation + Makefile: 3-phase deployment orchestration
- ECR: Docker image registry

### Data Flow (Inbound)
```
External system webhook → Delta Sync Queue (FIFO, keyed by CRM ID)
  → [if full sync running: visibility timeout buffer]
  → Delta Sync Worker → Mappings Manager → Audience
```
```
Full Sync trigger → Full Sync Manager
  → Full Sync Producer (paginated external API) → Full Sync Queue (FIFO)
  → Full Sync Consumer → Mappings Manager → Audience
  → [delta buffer drains after completion]
```

### Data Flow (Outbound)
```
Audience event → All Sub Queue → All Sub Worker
  → Kafka → Batch Production Worker (200KB or 1s batch)
  → Outbound Queue (FIFO) → Outbound Worker
  → Connector Library → External system
  [on failure] → DLQ → Retry Driver (6:00 AM) → Outbound Queue
```

### Data Flow (Forms → CRM)
```
Form submission → Audience Subscription Worker
  → Kafka → Form Event Batching Worker (5-min window)
  → Outbound Worker → CRM
  ← CRM response (ignore/match/create with entity ID)
  → [if entity ID returned] Profile Merge Worker
```

---

## 3. Module Reference

### Justin (Integration Middleware)
- **Purpose**: Generic infrastructure layer between Apsis One and all external systems; handles schema fetching, connector routing, and data standardization.
- **Key files**: Integration Manager service, Connector libraries (per system type)
- **Tech stack**: ECS tasks (formerly Lambda), HTTP endpoints
- **Data flow**: In: requests from frontend or internal services. Out: translated data to/from external systems in standard format.
- **Business rules**: One connector library per external system type; generic infrastructure never modified to add new integrations.
- **Configuration**: ECS task configuration per service; environment variables via Parameter Store
- **Integration points**: Audience, Athena, all external CRM/e-commerce systems
- **Gotchas**: "Justin" = "It's Just IN Sync" — the name is an acronym, not a person.
- **Tribal knowledge**: New integrations require only a new connector library; Justin core logic is untouched.

---

### Integration Manager
- **Purpose**: Gateway ECS service that fetches schema from external systems and routes requests to the appropriate connector library.
- **Key files**: Integration Manager service code; schema fetch logic; connector routing logic
- **Tech stack**: ECS task (formerly Lambda)
- **Data flow**: In: frontend requests for schema or feature support. Out: schema/feature response from external system via connector library.
- **Business rules**: Feature visibility requires BOTH Apsis-side support AND external system instance support to pass. Different instances of the same CRM may support different features.
- **Configuration**: Connector routing table; feature flag matrix per CRM type
- **Integration points**: All connector libraries; frontend; external CRM APIs
- **Gotchas**: On-premise CRM instances (e.g., eDeal) may support features that the cloud version does not, or vice versa.

---

### Mappings Manager
- **Purpose**: Stores and validates field mappings, consent mappings, and sync conditions; enforces that mappings don't create circular dependencies.
- **Key files**: Mappings database; caching layer; validation logic
- **Tech stack**: Dedicated database with caching layer
- **Data flow**: In: mapping queries from Delta Sync Worker, Full Sync Consumer, Outbound Worker. Out: applicable mappings. Cache invalidated on user mapping update.
- **Business rules**: Field mappings are **inbound only** (CRM → Apsis). Consent mappings are bidirectional. Sync conditions are boolean filters; failing inbound condition = record not synced.
- **Configuration**: Cache invalidation on update; cache hit avoids repeated DB queries
- **Integration points**: Delta Sync Worker, Full Sync Consumer, Outbound Worker, frontend mapping UI
- **Gotchas**: Recently updated mappings may not apply immediately due to caching. Cache TTL not specified.

---

### Delta Sync Worker
- **Purpose**: Consumes real-time webhook messages from external systems and writes mapped profiles to Audience.
- **Key files**: Delta Sync Worker service code; queue consumption logic
- **Tech stack**: ECS task (always-on), SQS FIFO consumer
- **Data flow**: In: Delta Sync Queue (FIFO, message group key = CRM ID). Out: mapped profiles to Audience via Mappings Manager.
- **Business rules**: FIFO ordering per CRM ID ensures eventual consistency per contact. During full sync, messages receive extended visibility timeout (buffering, not rejection).
- **Configuration**: SQS FIFO message group key = CRM ID
- **Integration points**: Delta Sync Queue, Mappings Manager, Audience
- **Gotchas**: **Do NOT change delta buffering to rejection** — this was attempted previously and abandoned. It would require re-architecting the entire queue system.

---

### Full Sync Manager
- **Purpose**: Orchestrates bulk sync jobs; spins up Full Sync Producer and Consumer ECS tasks on demand; tracks job state including "Finalizing" phase.
- **Key files**: Full Sync Manager service code; job state tracking; infrastructure spinup logic
- **Tech stack**: ECS task (formerly Lambda), ad-hoc spinup/teardown
- **Data flow**: In: user/scheduled trigger. Out: spins up Producer and Consumer; monitors progress; signals delta buffer drain after completion.
- **Business rules**: Full sync buffers all delta messages (visibility timeout) until completion. "Finalizing" state persists until Audience ingestion delay catches up — this is intentional; do not remove it.
- **Configuration**: Ad-hoc ECS task spinup (considered overkill but required due to historical queuing issues)
- **Integration points**: Full Sync Producer, Full Sync Consumer, Delta Sync Queue (to set buffer), Audience
- **Gotchas**: Ad-hoc spinup is wasteful but necessary. If the job fails mid-run, infrastructure must be torn down manually before a new job can start.

---

### Full Sync Producer
- **Purpose**: Fetches all records from external system via paginated API and places them on the Full Sync Queue.
- **Key files**: Full Sync Producer service code; pagination logic
- **Tech stack**: ECS task (spun up on demand)
- **Data flow**: In: trigger from Full Sync Manager. Out: paginated batches to Full Sync Queue (SQS FIFO).
- **Business rules**: Uses connector library for pagination. Translates to Justin standard format before queuing.
- **Configuration**: SQS FIFO message group key = integration ID; page size varies by connector (50–1000 records/page ⚠️)
- **Integration points**: Connector libraries, Full Sync Queue, external system APIs

---

### Full Sync Consumer
- **Purpose**: Consumes messages from Full Sync Queue, applies field/consent mappings, and sends profiles to Audience.
- **Key files**: Full Sync Consumer service code; batch processing logic
- **Tech stack**: ECS task (spun up on demand)
- **Data flow**: In: Full Sync Queue (1 or 10 messages per poll). Out: mapped profiles to Audience.
- **Configuration**: Consumer poll batch size: 1 or 10 (configurable ⚠️ — exact default unconfirmed)
- **Integration points**: Full Sync Queue, Mappings Manager, Audience

---

### Profile List Sync Manager
- **Purpose**: Manages imports of static and dynamic profile lists from external systems; adds/removes tags on existing Apsis profiles.
- **Key files**: Profile List Sync Manager service code; list import logic; tag management logic
- **Tech stack**: ECS task; CloudWatch Events for scheduling
- **Data flow**: In: user import request or scheduled trigger. Out: tag add/remove requests to Audience via sequential workers (one per integration at a time).
- **Business rules**: List import does **NOT** create profiles — only tags existing ones. Workers run sequentially per integration to avoid DDOSing customer systems. Lists are held in memory.
- **Configuration**: Default schedule: 6:00 AM via CloudWatch Events (optional per integration); one worker per integration at a time
- **Integration points**: Profile List Sync Queue, Audience, connector libraries
- **Gotchas**: Past failures from parallel workers and in-memory list holding. Do NOT parallelize without stress-testing large lists.
- **Tribal knowledge**: "Profiles" in FC Enterprise = static lists, not Apsis person records. Terminology was never reconciled.

---

### All Sub Queue (Audience Subscription Queue)
- **Purpose**: Central SQS FIFO queue collecting all Audience events: sent, delivered, opened, clicked, consent changes, MA notes.
- **Tech stack**: AWS SQS FIFO
- **Data flow**: In: events from Audience. Out: consumed by All Sub Worker.
- **Configuration**: FIFO; message ordering by source
- **Integration points**: Audience (producer), All Sub Worker (consumer)

---

### All Sub Worker
- **Purpose**: Validates integration installation and activity sync configuration; routes valid messages to Kafka.
- **Key files**: All Sub Worker service code; installation verification logic
- **Tech stack**: ECS task, SQS consumer, Kafka producer
- **Data flow**: In: All Sub Queue. Out: Kafka partition (for batching) or direct to Outbound Queue for consent (not batched).
- **Business rules**: Consent messages are processed one-by-one, not batched. All other event types go through Kafka for batching.
- **Integration points**: All Sub Queue, Kafka, integration configuration store

---

### Batch Production Worker
- **Purpose**: Polls Kafka partition; groups messages by target integration; batches into 200KB chunks with 1-second timeout.
- **Key files**: Batch Production Worker service code; batching algorithm
- **Tech stack**: ECS task, Kafka consumer, SQS FIFO producer
- **Data flow**: In: Kafka partition (all installations mixed). Out: ready batches to Outbound Queue (SQS FIFO).
- **Business rules**: Batch size limit = **200KB** (hard ceiling below SQS 256KB limit). Timeout = **1 second**. Whichever is reached first triggers batch send.
- **Configuration**: 200KB batch size; 1s timeout; SQS FIFO Outbound Queue
- **Integration points**: Kafka, Outbound Queue
- **Gotchas**: **Do NOT increase batch size** — SQS 256KB is a hard limit. Switching queue types is on the improvement list but not done. Kafka partition is single, not ordered globally; messages from all installations are mixed.

---

### Outbound Worker
- **Purpose**: Consumes batches from Outbound Queue and delivers them to external systems via connector libraries; routes failures to DLQ.
- **Key files**: Outbound Worker service code; batch sending logic; error handling
- **Tech stack**: ECS task, SQS FIFO consumer, connector library caller
- **Data flow**: In: Outbound Queue (FIFO). Out: external system (via connector library) or DLQ on failure.
- **Business rules**: Each outbound batch has a unique **batch ID** — external systems are contractually required to deduplicate on batch ID during redrives.
- **Configuration**: SQS FIFO message group key = target integration ID
- **Integration points**: Outbound Queue, connector libraries, Dead Letter Queue, external CRM/SaaS systems

---

### Retry Driver Lambda
- **Purpose**: Daily scheduled Lambda that redrives all DLQ messages back to Outbound Queue.
- **Key files**: Retry Driver Lambda function code; redrive logic
- **Tech stack**: AWS Lambda + CloudWatch Events
- **Data flow**: In: Dead Letter Queue (daily 6:00 AM UTC). Out: Outbound Queue (with consent filtering).
- **Business rules**: **Only opt-out (false) consent messages are redriven. Opt-in (true) consent messages are NEVER redriven.** Reason: redrives lose eventual consistency; resetting opt-in status incorrectly is a compliance risk.
- **Configuration**: CloudWatch Events schedule: daily 6:00 AM UTC; no maximum retry limit
- **Integration points**: Dead Letter Queue, Outbound Queue
- **Gotchas**: No maximum retry limit — infinite daily retries. No DLQ size alerting; must be reviewed manually.

---

### Connector Libraries
- **Purpose**: System-specific adapters translating between external system APIs and Justin's standard format.
- **Key files**: `<system>_connector.ts` (or equivalent per language); API endpoint mappings; field translation logic
- **Tech stack**: Varies by system; HTTP client; API authentication
- **Data flow**: In: Justin-format requests from Integration Manager, Full Sync Producer, Outbound Worker. Out: translated responses from/to external system APIs.
- **Business rules**: One connector library per external system type. Partners implement their own via the Generic Connector Interface spec (S3 bucket, domain-named folder).
- **Configuration**: Generic Connector spec: S3 folder with domain name. Implementation guide: last major update 2022; revised 2024 (⚠️ 2024 version not yet merged into main doc).
- **Integration points**: All internal consumers; all external system APIs
- **Gotchas**: 2024 Generic Connector Implementation Guide updates must be merged before onboarding new partners.

---

### Base Installer
- **Purpose**: Foundation class defining common functions (especially keyspace creation) shared across all CRM connectors; prevents code duplication.
- **Key files**: Base Installer module (exact path ⚠️ not provided in KT)
- **Tech stack**: Object-oriented inheritance pattern
- **Data flow**: In: connector configuration + section discriminator + integration info. Out: keyspace definitions with hashed discriminators.
- **Business rules**: All CRM connectors inherit from Base Installer. New common functions must be added here, not in individual connectors.
- **Integration points**: All CRM connector implementations (Dynamics, E-deal, Salesforce, FSC Enterprise, Sideshop, Corporate, Episerver)
- **Tribal knowledge**: Connector hierarchy: Base Installer → Generic Connector → individual CRM connectors (some connectors skip Generic Connector and inherit directly from Base Installer).

---

### Installation Manager
- **Purpose**: Orchestrates CRM integration setup: creates keyspaces and entity-specific keyspaces; stores discriminators; registers form event listeners.
- **Key files**: Installation Manager module (exact path ⚠️ not provided in KT)
- **Tech stack**: Not specified
- **Data flow**: In: CRM install trigger with section discriminator + connector config. Out: keyspace records in DB; event listener registrations.
- **Business rules**: Keyspace creation is deterministic and automatic. Keyspaces are **never deleted on uninstall** (GDPR + win-back). Same CRM reinstalled on same section **reuses existing keyspace**. Form event listeners (`start_viewed`, `submit`) only registered if `can_sync_form_activities = true`.
- **Configuration**: Keyspace discriminator format: `integrations.keyspaces.<8-char-hash>.<crm-logical-name>[.<entity-name>]`
- **Integration points**: Keyspace manager, DB, form event system, connector libraries

---

### Profile Merge Worker
- **Purpose**: Merges profiles across different keyspaces; used for form submissions, CRM-initiated duplicate resolution, and keyspace bridging.
- **Key files**: Profile merge handler (exact path ⚠️ not provided in KT)
- **Tech stack**: Async message queue (type ⚠️ not specified)
- **Data flow**: In: merge message `{source_keyspace, source_key, dest_keyspace, dest_key}`. Out: consolidated profile record; source marked for consolidation.
- **Business rules**: Merges may **only** be initiated from the CRM system — never from Apsis unilaterally. CRM-sourced contacts must **never** be merged with the email keyspace or non-CRM keyspaces outside of a CRM merge request. Merges from Apsis without CRM knowing cause out-of-sync state.
- **Integration points**: Profile Update Service, keyspace manager, form submission handler, CRM connector
- **Gotchas**: Merge is asynchronous — stale data window exists. Profiles without email attributes are silently not merged in Option 2 workaround.

---

### Broker Service
- **Purpose**: Decrypts generic connector customer API credentials (via KMS) and routes outbound requests through Squid Proxy to customer systems.
- **Key files**: Broker Service module (exact path ⚠️ not provided in KT)
- **Tech stack**: Microservice, AWS KMS client, Squid Proxy
- **Data flow**: In: message from queue with encrypted credentials reference. Out: decrypted API request → Squid Proxy → customer system.
- **Business rules**: KMS key is **single-region only** — credentials cannot be decrypted in disaster recovery (backup) accounts. This is known technical debt.
- **Configuration**: KMS key ARN (single-region); Squid Proxy endpoint
- **Integration points**: KMS, Squid Proxy, generic connector credential DB, customer external APIs
- **Gotchas**: Multi-region KMS key cannot be retrofitted to existing key — requires new key + full DB migration of all encrypted credentials.

---

### Audience (Apsis Profile Store) — External Dependency
- **Purpose**: Central Apsis profile store; receives mapped inbound data; emits events for outbound processing; supports HTTP/2 streaming for unified data queries.
- **Tech stack**: Data storage with ingestion endpoints; HTTP/2 streaming
- **Data flow**: In: mapped profiles from Delta/Full Sync. Out: events to All Sub Queue; HTTP/2 streams to Integrations for unified data queries.
- **Business rules**: Audience ingestion has a real delay — Full Sync "Finalizing" state intentionally waits for this.
- **Gotchas**: Audience service takes 30+ minutes to fully initialize after deployment (even after CloudFormation shows COMPLETE). All integration operations requiring profile data block on Audience availability.

---

### Athena (Data Warehouse) — External Dependency
- **Purpose**: Joins Audience profile exports with Integration-streamed external CRM relational data for advanced queries.
- **Tech stack**: Columnar data warehouse; Apache Columnar format input
- **Data flow**: In: Audience profile exports + Integrations HTTP/2 streams (translated to Apache Columnar). Out: joined query results.
- **Gotchas**: Unified data / Athena join path not proven at scale in production — one beta customer only. ⚠️ Do not rely on it for high-volume production use without further stress testing.

---

## 4. Cross-Cutting Concerns

### Authentication
- **External systems**: API key + endpoint URL (provided by customer at install time)
- **Microsoft Dynamics**: OAuth 2.0; Client ID must come from **Apsis International AB** Azure tenant, NOT FSC account. Using wrong account will authenticate but fail at runtime.
- **Generic connector credentials**: Encrypted with AWS KMS (single-region) before storage in RDS; decrypted by Broker Service at runtime.
- **Internal service-to-service**: Method ⚠️ not fully specified in KT sessions.
- **Legacy Enterprise 12.0 connectors**: Credentials stored unencrypted (legacy model).

### Error Handling
- **Outbound failures**: Failed messages → DLQ → Retry Driver at 6:00 AM UTC daily (infinite retries, no maximum).
- **Consent redrive exception**: Only opt-out (false) consent redriven. Opt-in never redriven.
- **Batch ID deduplication**: External systems contractually required to deduplicate redrives by batch ID.
- **DLQ monitoring**: No automated alerting — manual review required (Monday operational review recommended).
- **Form submission failures**: 504 errors from CRM → retry logic ⚠️ (backoff strategy not specified in KT).

### Logging / Observability
- **CloudWatch Logs**: Per-service runtime logs
- **SNS topics**: Alerts (created in base infrastructure phase)
- **DLQ depth**: No automated alert — manual review only ⚠️
- **Operational review**: Weekly Monday meeting recommended (was historically active, then stopped; should resume if DLQ accumulation occurs)

### Deployment
Three-phase CloudFormation + Makefile orchestration:
1. **Phase 1 (Base)**: ECR, Lambda buckets, IAM policies, SNS topics — all SQS queues created here (enables Phase 3 parallelization)
2. **Phase 2 (Persistent)**: RDS cluster (~30 min), Redis (~30 min), Kafka, Squid Proxy — longest phase; deploy separately with `make deploy-db` if concerned about timeout
3. **Phase 3 (Microservices)**: 40+ services in parallel batches of 4–5; enabled by Phase 1 queue pre-deployment

Command: `make deploy ABS_PROFILE=<profile>` (e.g., `dr`, `prod`, `staging`)

**Key pre-deployment steps for DR:**
1. Manually create secrets in AWS Secrets Manager (Dynamics OAuth Client ID from Apsis International AB Azure tenant; DB encryption key)
2. Create `nv-<ABS_PROFILE>.conf` with all shared service URLs (Audience, CRM endpoints, Folder service)
3. Create AVS env config with ECR host: `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com`

**Critical DR limitation**: Generic connector credentials cannot be decrypted in backup accounts (single-region KMS key). Plan for customer credential reinstallation or test only with legacy Enterprise 12.0 connectors.

---

## 5. Business Rules Reference

### Integration Constraints
- Only **one CRM integration active per section** at a time (blocked by CRM_ID single field space, pre-2020 integrations). This is the single largest piece of technical debt.
- Non-CRM integrations (Playable, third-party SaaS) are not subject to this limitation — they use custom key spaces.
- Post-2020 integrations use per-entity ID fields (proper design); pre-2020 use shared CRM_ID.

### Field Mappings
- **Inbound only**: CRM → Apsis. Attributes do not sync back to CRM.
- **Consent is the exception**: bidirectional between Apsis and CRM (legal requirement).
- Opt-in messages are **never redriven** from DLQ; only opt-out messages are redriven.
- Sync conditions are boolean filters on inbound records; failed condition = record not synced.
- Outbound form field mappings are **not fully supported** — only profile attributes can be synced from form submissions; form-specific field data stays in Apsis only.

### Keyspace Rules
- Keyspace discriminator format: `integrations.keyspaces.<8-char-hash>.<crm-logical-name>[.<entity-name>]`
- Keyspaces are **never deleted on CRM uninstall** (historical data, GDPR, win-back).
- Reinstalling same CRM on same section **reuses existing keyspace**.
- **Merges must originate from the CRM** — never from Apsis. Merging CRM contacts with email keyspace breaks sync.
- CRM ID attribute must **never** be modified via form or any Apsis mechanism.
- Lead/silhouette entities are stored in **separate entity-specific keyspaces**; never pre-emptively merged with contact keyspace.
- Form sync (`can_sync_form_activities`) only available if CRM integration is installed and flag is true on connector.

### Profile Identification
- Form submission requires: CRM ID OR email OR phone (SMS). First name + last name alone are insufficient.
- CRM is the **master data source** for all contact attributes. Apsis does not push attribute changes back to CRM (only consent and events).
- Enrichment attributes (data collected in Apsis not from CRM) are Apsis-only and not pushed to CRM.

### Outbound Batching
- Batch max size: **200KB** (SQS 256KB hard limit with safety margin).
- Batch timeout: **1 second**.
- External systems are contractually required to deduplicate redrives via **batch ID**.
- Form submission events to CRM are batched over a **5-minute window**.

### Full Sync
- Delta messages during full sync: **buffered** (visibility timeout), never rejected.
- Full sync "Finalizing" state waits for Audience ingestion delay to catch up before reporting success.
- Infrastructure is spun up on demand and torn down after completion.

### Dead Letter Queue
- Daily redrive at **6:00 AM UTC** via Retry Driver Lambda.
- **No maximum retry limit** (infinite).
- Opt-in consent messages: **never redriven**.
- Opt-out consent messages: redriven.

---

## 6. Known Issues & Workarounds

| Issue | Severity | Workaround | Component |
|-------|----------|-----------|-----------|
| Only one CRM integration per section (CRM_ID single key space) | High | Bootstrapped unique key spaces per integration (FSC_Enterprise_ID, Dynamics_ID, etc.) that internally map to CRM_ID | All inbound/outbound sync |
| Pre-filled form profile resolution may create duplicates across keyspaces | High | Testing required to verify; implement cross-keyspace lookup and merge before GA | Form Tool, Profile Merge Worker |
| Pre-filled forms do not display consent checkboxes | High | Manual consent update required separately | Form Tool |
| Form submission can overwrite CRM-sourced attributes | High | Product decision needed: block overwrites or allow with reset on next sync; Mappings Manager must be queried to identify protected fields | Form Tool |
| CRM ID attribute exposed in form editor when integration active | High | Hide CRM ID attribute from form editor when section has active CRM integration | Form Tool |
| Contacts merged in Apsis without CRM initiation break sync | High | Enforce: merges must always originate from CRM | Profile Merger, Data Governance |
| Single-region KMS key prevents credential decryption in DR account | High | Plan for customer credential reinstallation; or create new multi-region KMS key + full DB migration (non-trivial) | Broker Service, Generic Connector |
| Acceptance criteria (form/CRM) expects attribute updates sent to CRM — architecture does not support this | High | Clarify with product: only events are sent, not attribute values | Integration, Outbound Worker |
| Tribe CRM entity selection via dynamic dropdown incompatible with static Apsis entity config | High | Create virtual "lead" keyspace mapping to Tribe's current dropdown entity | Tribe Connector |
| Outbound form field mappings not fully supported (only profile attributes, not form field data) | Medium | Document limitation; potential future feature project | Forms integration, Outbound Worker |
| Unified data HTTP/2 streaming not proven at scale | Medium | One beta customer only; do not rely on for high-volume production use | Unified Data pipeline |
| Profile list import held in memory; past failures with no recovery | Medium | Sequential workers (one per integration); now more stable but do not parallelize | Profile List Sync |
| MA webhook node sends one-by-one; frequent "too many requests" errors | Medium | No current fix; could reuse outbound event batching infrastructure | MA webhook node |
| Orphaned profiles when keyspace mapping changed mid-stream (Option 3) | Medium | Manual merge work or customer full resync | Integration Installation Mapping |
| Duplicate profiles when same customer uses JoinCX sync (CRM ID) + direct API (email) | Medium | Option 2 (Profile Merge Worker automation, 3–5 days) or Option 3 (DB config change, immediate) | Keyspace Manager |
| Option 3 DB workaround reverts on integration reinstall | Medium | Reapply DB change post-reinstall; document in runbook | Integration Installation Mapping |
| DLQ infinite retries with no maximum limit | Low | Monday operational reviews; deprioritize known-bad integrations | Retry Driver Lambda |
| Ad-hoc full sync infrastructure considered overkill | Low | Necessary due to historical queuing issues; redesign if queuing architecture improves | Full Sync Manager |
| SQS 256KB message limit constrains batch size | Low | 200KB batch size is permanent constraint unless queue type is changed | Batch Production Worker |
| Generic Connector Implementation Guide 2024 updates not merged | Low | Must merge before onboarding new partners | Connector documentation |
| Undiscovered interdependencies in DR setup (2-year gap since last from-scratch deploy) | Medium | Expect trial-and-error; CloudFormation errors are self-documenting | CloudFormation pipeline |
| Audience service takes 30+ min to fully initialize after deployment | Medium | Plan extended wait; test non-Audience-dependent components first | Audience dependency |
| 'Jonas' era CRM_ID single field space — never fixed | High | Unique key space workaround; migration project never prioritized | All CRM integrations |
| 2022–2024 Generic Connector Implementation Guide not merged | Low | Benjamin committed to merging; prioritize before new partner onboarding | Partner documentation |

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Justin** | Integration middleware ("Just IN Sync"); generic infrastructure layer between Apsis One and external systems |
| **Connector Library** | System-specific adapter translating between an external system's API and Justin's standard format |
| **Integration Manager** | ECS service that fetches schemas from external systems and routes to connector libraries |
| **Mappings Manager** | Service storing/validating field mappings, consent mappings, sync conditions |
| **Delta Sync** | Real-time sync of individual record changes via webhooks |
| **Full Sync** | Bulk sync of all records from external system (initial setup or manual trigger) |
| **Profile List Sync** | Import of static/dynamic lists from external system; adds tags to existing Apsis profiles |
| **All Sub Queue** | Central SQS FIFO collecting all Audience events for outbound processing |
| **Batch Production Worker** | Groups outbound messages by integration into 200KB/1s batches |
| **Outbound Worker** | Sends batches to external systems via connector libraries |
| **Retry Driver Lambda** | Daily 6:00 AM Lambda that redrives DLQ messages to Outbound Queue |
| **Dead Letter Queue (DLQ)** | Holds failed outbound messages; redriven daily |
| **CRM_ID** | Single shared field space in Audience for all pre-2020 CRM integrations; root cause of one-CRM-per-section limitation |
| **Sync Condition** | Boolean filter on inbound records (e.g., only sync if `active == true`) |
| **Unified Data** | Advanced query capability joining external CRM relational data (HTTP/2 stream) with Apsis profile data in Athena |
| **Keyspace** | Profile identity namespace using a specific identifier type (email, CRM ID, etc.); enforces its own validation and uniqueness |
| **Keyspace Discriminator** | Unique string identifier: `integrations.keyspaces.<8-char-hash>.<crm-logical-name>[.<entity-name>]` |
| **Base Installer** | Foundation class providing common CRM connector functions (especially keyspace creation) |
| **Installation Manager** | Orchestrates CRM integration install: creates keyspaces, registers form event listeners |
| **Silhouette** | E-deal terminology for a lead entity (form submission-generated prospect); assigned silhouette ID (not person ID); stored in separate keyspace; only receives deletion events |
| **Person Entity** | Known CRM contact; receives full profile updates |
| **Lead Entity** | Preliminary CRM prospect (used in Dynamics Sitecore version); separate keyspace from contacts |
| **Profile Merge Worker** | Merges profiles across keyspaces; must be initiated by CRM, not Apsis |
| **Bootstrapped Key Space** | Automatically created keyspace when CRM integration is installed; uses CRM ID as primary key |
| **can_sync_form_activities** | Boolean flag on CRM connector; if false, form sync option is hidden and no event listeners registered |
| **Finalization State** | Full sync state awaiting Audience ingestion delay catchup; shows "Finalizing" until data confirmed written |
| **Batch ID** | Unique identifier per outbound batch; external systems must deduplicate by this on redrives |
| **Supplier** | External agency paid hourly by Apsis to build integrations (old model; no longer active; e.g., CRM Consultana) |
| **Partner** | External system provider who implements Apsis's Generic Connector Interface and bills customers directly (current model; e.g., Sideshop, Intermail) |
| **Generic Connector Interface** | API spec/contract that external systems fulfill; Apsis provides spec, partners implement |
| **Definition of Done (DoD)** | Supplier delivery checklist: 75% test coverage, installation instructions, roles required, file/table documentation |
| **Abandoned Cart** | Standardized e-commerce event; runs hourly, fetches carts from all connected systems, emits events to Audience |
| **Visibility Timeout** | SQS mechanism to delay message visibility; used to buffer delta messages during full sync |
| **Columnar Format** | Apache Columnar format used for unified data HTTP/2 streaming; efficient for large result sets |
| **Virtual Consent** | Consent represented as fields on contact record (legacy approach) |
| **Native Consent** | Consent as dedicated resource in external system (modern approach) |
| **ABS_PROFILE** | Deployment environment identifier (e.g., `prod`, `dr`, `staging`); used to construct config file names |
| **Broker Service** | Decrypts generic connector credentials (KMS) and routes requests via Squid Proxy |
| **Squid Proxy** | HTTP/HTTPS proxy managing outbound network traffic from integration services |
| **Win-Back Scenario** | Customer re-enables a previously uninstalled integration; keyspace persistence ensures historical data is available |
| **GDPR Cleanup** | Profile data deletion on customer request; justifies keeping keyspaces even after integration uninstall |
| **Message Group Key** | SQS FIFO parameter ensuring ordering; Delta Sync uses CRM ID; Full Sync uses integration ID; Outbound uses target integration ID |
| **Ritual Fields** | Fields added to events by Integration for context: activity ID, timestamp, event type, profile key |
| **Form Event Batching** | Collection of form events over 5-minute window before CRM transmission |
| **Enrichment Attribute** | Apsis attribute not mapped from CRM; freely updatable via forms; not pushed to CRM |
| **Mapped Attribute** | Attribute sourced from CRM via active mapping; should be protected from form overwrites |
| **Option 2 (Merge Worker)** | Workaround for keyspace mismatch: automate profile merge after JoinCX sync; 3–5 days effort; no customer API changes |
| **Option 3 (DB Config Change)** | Workaround for keyspace mismatch: update Integration Installation Mapping table to route to email keyspace; ~5 seconds; customer must send emails in CRM ID field; reverts on reinstall |
| **Apsis International AB** | Parent Apsis Azure tenant (not FSC); correct account for Dynamics OAuth Client ID |

---

## 8. Confidence Notes

### High Confidence (verified across multiple sessions)
- CRM_ID single field space limitation and its impact on one-CRM-per-section rule
- Delta sync buffering during full sync (not rejection) — confirmed as intentional, do not change
- 200KB batch size / 1-second timeout for outbound batching
- Do NOT redrive opt-in consent messages from DLQ
- Full sync "Finalizing" state is intentional (Audience ingestion delay)
- Keyspaces never deleted on uninstall
- Merges must originate from CRM, not Apsis
- Consent is bidirectional; all other attributes are inbound-only
- Base Installer inheritance pattern for CRM connectors

### Medium Confidence (single source or partially corroborated)
- Full Sync Consumer poll batch size (1 or 10) — exact default ⚠️ unconfirmed
- Full Sync Producer page size (50–1000 records) — varies by connector ⚠️
- KMS single-region rationale — described as implementation constraint, not intentional decision
- Option 2 vs Option 3 recommendation — Erik's preference; pending internal alignment meeting with Alexander Lindström and Opri
- Unified data HTTP/2 streaming stability — one beta customer only ⚠️

### Low Confidence / Unresolved ⚠️
- Exact hashing algorithm for keyspace discriminator generation ("SHA-1 or something" per transcript)
- Exact service-to-service authentication method between integration services and Audience
- Whether pre-filled form profile resolution correctly handles cross-keyspace lookup (untested — high-priority before GA)
- Whether the 2024 Generic Connector Implementation Guide updates have been merged (Benjamin committed to this; status unknown)
- Current status of CRM Consultana (Dynamics) supplier contract — active or discontinued?
- Legacy open-source CMS integrations (Drupal, FP Server, Sitecore) — active maintenance or deprecated?
- Maximum retry limit for DLQ — described as "infinite" but no system-enforced cap confirmed in code
- `allow_override_data` form option behavior — described as unclear even by Erik; requires code investigation

### Conflicts Resolved
- **"Justin" definition**: One session named it "Integration Middleware" generically; confirmed in overview session as acronym "Just IN Sync." Kept acronym version.
- **Outbound Queue type**: Described as "SQS FIFO" in overview (Benjamin) and confirmed by Erik's queue discussion. No conflict.
- **Consent directionality**: Both Benjamin and Erik confirm bidirectional for consent only; all other attributes inbound-only. Consistent.
- **Form batching window**: Erik specifies 5-minute window for form event batching; Benjamin's session does not mention this specific value. Kept Erik's value.