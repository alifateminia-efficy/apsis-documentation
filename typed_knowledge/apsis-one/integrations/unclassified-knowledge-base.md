---
title: Apsis One Integrations — unclassified Knowledge Base
subdomain: unclassified
generated: 2026-03-25T12:40:46.849Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-haiku-4-5-20251001
source: KT session transcripts (3 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One Integrations — unclassified Knowledge Base](#apsis-one-integrations-unclassified-knowledge-base)
  - [1. Subdomain Overview](#1-subdomain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Justin (Core Integration Middleware)](#justin-core-integration-middleware)
    - [Integration Manager (Service)](#integration-manager-service)
    - [Mappings Manager (Service)](#mappings-manager-service)
    - [Delta Sync Worker (ECS Task)](#delta-sync-worker-ecs-task)
    - [Full Sync Producer (ECS Task)](#full-sync-producer-ecs-task)
    - [Full Sync Consumer (ECS Task)](#full-sync-consumer-ecs-task)
    - [Full Sync Manager (ECS Task/Service)](#full-sync-manager-ecs-taskservice)
    - [Outbound Manager (Service)](#outbound-manager-service)
    - [Audience Subscription Queue (ASQ) (SQS FIFO)](#audience-subscription-queue-asq-sqs-fifo)
    - [Audience Subscription Queue Worker (ASQ Worker) (ECS Task)](#audience-subscription-queue-worker-asq-worker-ecs-task)
    - [Batch Production Worker (ECS Task)](#batch-production-worker-ecs-task)
    - [Outbound Worker (ECS Task)](#outbound-worker-ecs-task)
    - [Dead Letter Queue (DLQ) (SQS)](#dead-letter-queue-dlq-sqs)
    - [Retry Driver Lambda (AWS Lambda)](#retry-driver-lambda-aws-lambda)
    - [Consent Message Processing (Outbound)](#consent-message-processing-outbound)
    - [Connector Library (Generic)](#connector-library-generic)
    - [Connector Library (Microsoft Dynamics Legacy)](#connector-library-microsoft-dynamics-legacy)
    - [Connector Library (Efficy Enterprise)](#connector-library-efficy-enterprise)
    - [Profile Lists Import (Static and Dynamic)](#profile-lists-import-static-and-dynamic)
    - [Sync Conditions Filtering](#sync-conditions-filtering)
    - [Unified Data Export Service](#unified-data-export-service)
    - [Audience (Profile Store & CDP)](#audience-profile-store-cdp)
    - [React (Real-time Profile Store)](#react-real-time-profile-store)
    - [Athena (Data Warehouse)](#athena-data-warehouse)
    - [CloudFormation Deployment Infrastructure](#cloudformation-deployment-infrastructure)
    - [Parameter Store (AWS Systems Manager)](#parameter-store-aws-systems-manager)
    - [Secrets Manager (AWS)](#secrets-manager-aws)
    - [KMS (Key Management Service)](#kms-key-management-service)
    - [Broker Service](#broker-service)
    - [Squid Proxy](#squid-proxy)
    - [Integration Mapping Table (Database)](#integration-mapping-table-database)
    - [Profile Merge Worker Service](#profile-merge-worker-service)
    - [Keyspace System (CRM ID, Email, and Other Variants)](#keyspace-system-crm-id-email-and-other-variants)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Authorization](#authentication-authorization)
    - [Error Handling & Logging](#error-handling-logging)
    - [Deployment & Configuration](#deployment-configuration)
    - [Monitoring & Alerting](#monitoring-alerting)
    - [Disaster Recovery](#disaster-recovery)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Sync Direction & Scope](#sync-direction-scope)
    - [Record Filtering & Consistency](#record-filtering-consistency)
    - [Integration Installation & Configuration](#integration-installation-configuration)
    - [Mapping Constraints (Justin's Laws)](#mapping-constraints-justins-laws)
    - [Outbound Activity Flow](#outbound-activity-flow)
    - [Profile List Import](#profile-list-import)
    - [Keyspace Management](#keyspace-management)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
    - [Critical Issues](#critical-issues)
    - [High-Impact Issues](#high-impact-issues)
    - [Medium-Impact Issues](#medium-impact-issues)
    - [Low-Impact Issues](#low-impact-issues)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence (Sourced from multiple sessions, corroborated)](#high-confidence-sourced-from-multiple-sessions-corroborated)
    - [Medium Confidence (Sourced from single session, logical inferences)](#medium-confidence-sourced-from-single-session-logical-inferences)
    - [Low Confidence (Sourced from single session, incomplete details)](#low-confidence-sourced-from-single-session-incomplete-details)
    - [Gaps & Uncertainties](#gaps-uncertainties)
    - [Discrepancies & Contradictions](#discrepancies-contradictions)
    - [Recommendations for KB Curation](#recommendations-for-kb-curation)
  - [9. Reference Artifacts](#9-reference-artifacts)
    - [Integration Types & Connectors Supported](#integration-types-connectors-supported)
    - [CloudFormation Deployment Checklist](#cloudformation-deployment-checklist)
    - [Disaster Recovery Known Issues & Responses](#disaster-recovery-known-issues-responses)

---

# Apsis One Integrations — unclassified Knowledge Base

## 1. Subdomain Overview

The **unclassified** subdomain encompasses Apsis One's integration infrastructure for synchronizing customer data bidirectionally between Apsis's unified profile store (Audience) and external systems (CRMs, loyalty platforms, ecommerce systems). This subdomain covers the architecture, deployment, and operational management of ~15+ system-specific and generic connectors; real-time (delta) and batch (full) synchronization workflows; outbound activity sync (emails, forms, tasks); consent management; profile merging across identifier keyspaces; and disaster recovery procedures. It excludes domain-specific integrations (Marketing Automation, CMS, reporting) and focuses on the core infrastructure serving CRM/ERP/loyalty integrations across ~100+ customer accounts.

---

## 2. Architecture Map

```
INBOUND FLOW (CRM → Audience):
  External CRM → Connector Library → Justin format → Mappings Manager → Delta/Full Sync Workers → Audience profile store
  ├─ Real-time: Webhook → Delta Sync Worker → (cached) Mappings Manager → update Audience
  └─ Batch: Full Sync Producer (paginate CRM) → Full Sync Consumer (apply mappings) → Audience

OUTBOUND FLOW (Audience → CRM):
  Audience event → Audience Subscription Queue (ASQ) 
  → ASQ Worker (validate installation) → Kafka (partition by integration)
  → Batch Production Worker (batch 200KB or 1s timeout) → SQS FIFO queue
  → Outbound Worker (send via Connector, retry via DLQ) → External CRM
  
CONSENT BIDIRECTIONAL:
  Audience consent change ↔ (bypass batching) → SQS → Outbound Worker → CRM
  CRM consent webhook → Delta Sync Worker → Audience

PROFILE MERGE (multi-keyspace):
  Profile in CRM_ID keyspace + Email keyspace → Profile Merge Worker → Unified profile queryable via both identifiers

INFRASTRUCTURE LAYERS:
  ├─ Config: Parameter Store (URLs, endpoints), Secrets Manager (credentials)
  ├─ Storage: RDS (mappings, config), Redis (cache), Kafka (message queue)
  ├─ Compute: Lambda/ECS (services), CloudFormation (deployment)
  ├─ Security: KMS (credential encryption), IAM (permissions)
  └─ Monitoring: SNS (alerts), CloudWatch (logs)

KEY SERVICES (4-5 deployed per parallel batch):
  Manager Services (Integration Manager, Mappings Manager, Outbound Manager, Full Sync Manager)
  Worker Services (Delta Sync Worker, Full Sync Producer/Consumer, ASQ Worker, Batch Production Worker, Outbound Worker, Retry Driver Lambda)
  Support Services (Broker Service for credential injection, Sluice Worker for message routing)
```

---

## 3. Module Reference

### Justin (Core Integration Middleware)
- **Purpose**: Generic infrastructure layer enabling sync operations and keeping external systems synchronized with Audience profile store. Named after Justin Timberlake.
- **Key files**: Integration specs (code/config), Mappings table (database)
- **Tech stack**: Modular connector library architecture, event-driven microservices
- **Data flow**: External system → Connector library → Justin format → Mappings translation → Audience attributes/consent
- **Business rules**: 
  - Attributes unidirectional (external→Apsis only); consent bidirectional
  - Enforces Mappings Manager consistency rules (no loops)
  - Respects sync conditions filtering
- **Configuration**: Integration specs define capabilities per system; runtime feature flag checks
- **Integration points**: All connectors, Audience, Mappings Manager
- **Gotchas**: 
  - Circular dependencies between services prevented via intentional queue decoupling
  - Service deployment must respect Phase ordering (Base → Database → Supporting → Microservices)
- **Tribal knowledge**: Shift from supplier-based (Geran Kosnokana) to partner-based (Sideshop) model is strategic; commercial org hasn't completed migration

### Integration Manager (Service)
- **Purpose**: Serves field mapping UI and fetches external system schemas via connector libraries; formerly Lambda, now ECS task with caching layer
- **Key files**: Connector library calls, Mappings Manager queries
- **Tech stack**: ECS task, HTTP/REST API, connector abstraction
- **Data flow**: User navigates /integrations → Integration Manager queries connector → schema returned → UI renders dropdowns → user maps fields → stored in Mappings Manager
- **Business rules**: 
  - Fetches schema from external system at request time
  - Queries Mappings Manager for active mappings
  - Instance-level capability checks verify actual external system support
- **Configuration**: Connector type (dynamics, fc_enterprise, generic, etc.) selected during installation
- **Integration points**: Frontend UI, Connector libraries, Mappings Manager, Audience service
- **Gotchas**: 
  - Schema fetch depends on connector library working correctly; broken connector = broken schema fetch
  - Webhook subscription updates on mapping save can fail silently; check logs
  - Multiple CRM integrations per section will collide on CRM ID field (current limitation)
- **Tribal knowledge**: Mappings Manager cache in Delta Sync Worker can become stale if mapping save webhook fails; monitor cache logs

### Mappings Manager (Service)
- **Purpose**: Manages field and consent mappings between external systems and Audience attributes; enforces Justin's laws (consistency rules preventing loops); provides cached lookup for Delta Sync Worker
- **Key files**: Mappings table (database)
- **Tech stack**: Microservice with database user access, caching layer
- **Data flow**: Field/consent mapping created → validated against Justin's laws → stored in mappings table → cached for Delta Sync lookups → applied during sync operations
- **Business rules**: 
  - Enforces consistency rules (no loops, no invalid configurations)
  - Mappings are installation-specific and entity-type-specific
  - Consent mapping translates external topics to Audience topics
  - Attributes unidirectional; consent bidirectional only
- **Configuration**: Field mappings (source field → target attribute), Consent mappings (external topic → Audience topic), Sync conditions (field expressions)
- **Integration points**: Integration Manager (mapping CRUD), Delta Sync Worker (cached reads), Full Sync Consumer (mapping lookups), Consent mapping system
- **Gotchas**: 
  - Cache invalidation critical; stale cache causes old mappings to apply to new records
  - Consent resources terminology varies (virtual vs native) by system; confusing
  - Mapping save must trigger webhook to update external system webhooks; silent failures possible
- **Tribal knowledge**: Mappings persist across reinstalls unless manually changed; reinstall always reverts to default configuration

### Delta Sync Worker (ECS Task)
- **Purpose**: Consumes real-time webhook updates from external systems and applies mappings to sync changed records to Audience in real-time
- **Key files**: Webhook endpoints (custom or official), Mappings Manager cache queries
- **Tech stack**: ECS task, asynchronous message consumer, cache-heavy
- **Data flow**: External system webhook → inbound queue → Delta Sync Worker queries Mappings Manager (cached) → mapped data sent to Audience; messages buffered if full sync in progress
- **Business rules**: 
  - Respects Mappings Manager (cached) for field translation
  - Evaluates sync conditions; ignores/deletes records failing conditions
  - Buffers messages during full sync to maintain eventual consistency (processed after full sync completes)
  - Deduplication: relies on external system behavior (no built-in deduplication)
- **Configuration**: Cache TTL (not explicitly specified; assumed short), Full Sync buffering queue name
- **Integration points**: External system webhooks, Mappings Manager (cached), Audience, Full Sync buffering queue
- **Gotchas**: 
  - Webhook subscription can fail silently; monitor connector logs
  - Large webhook payloads cause queue processing delays
  - No deduplication; duplicate webhooks → duplicate processing
  - Sync conditions evaluated only on delta; existing records not re-evaluated unless profile updated
  - Messages buffered during full sync may create noticeable real-time update delay
- **Tribal knowledge**: Buffering ensures eventual consistency but introduces visible delay in real-time sync; communicate to customers

### Full Sync Producer (ECS Task)
- **Purpose**: Paginates all contacts from external system using connector library and places them in Justin format on Full Sync Queue
- **Key files**: Connector library pagination calls
- **Tech stack**: ECS task, stateless paginated API client
- **Data flow**: User clicks 'Start new sync' → infrastructure spun up → Producer paginates all contacts from external system → converts to Justin format → queues for consumer
- **Business rules**: 
  - Respects connector library pagination interface
  - Fetches ALL records (no filtering; filtering happens at consumer)
  - Converts external system format → Justin canonical format
- **Configuration**: Connector type, external system API endpoint (from Parameter Store)
- **Integration points**: External system API (paginated), Connector library, Full Sync Queue
- **Gotchas**: 
  - Holds all records in processing queue; very large systems may timeout/memory-exhaust
  - Pagination token issues in connector library can cause hang or infinite loop
  - No automatic timeout; full sync can hang indefinitely if connector broken
  - No memory monitoring; large systems silently fail
- **Tribal knowledge**: Test pagination with large datasets (>1 million records); add timeout to producer

### Full Sync Consumer (ECS Task)
- **Purpose**: Processes queued full sync records, queries Mappings Manager for active mappings, sends records to Audience in batches
- **Key files**: Full Sync Queue consumer code, Mappings Manager queries
- **Tech stack**: ECS task, queue consumer, batch sender
- **Data flow**: Queued records (Justin format) → query Mappings Manager for mappings → evaluate sync conditions → batch send to Audience; delta sync messages buffered on separate queue during this process
- **Business rules**: 
  - Applies Mappings Manager mappings (non-cached, fresh lookup per record)
  - Evaluates sync conditions; ignores records failing conditions
  - Buffers delta sync messages on separate queue (processed after full sync completes)
  - Batch sends to Audience (reduces API calls)
- **Configuration**: Batch size (not user-configurable), queue name, Audience endpoint (from Parameter Store)
- **Integration points**: Full Sync Queue, Mappings Manager, Audience, delta sync buffering queue
- **Gotchas**: 
  - Full sync overwrite: subsequent full syncs overwrite existing profile data; old attributes may not be cleaned up if mappings changed
  - Buffered delta messages create noticeable delay; must communicate to customer
  - Queue visibility timeout must be long enough for full sync to complete
  - Audience ingestion delay tracking required; UI state depends on this
- **Tribal knowledge**: Subsequent full syncs are safe but data migration risk if mappings change; test with staging environment first

### Full Sync Manager (ECS Task/Service)
- **Purpose**: Manages full sync lifecycle and UI state transitions (Downloaded → Finalizing → Successful)
- **Key files**: Not specified
- **Tech stack**: ECS task/service, state machine
- **Data flow**: Monitors Producer/Consumer progress → tracks Audience ingestion delay → updates UI state through lifecycle
- **Business rules**: 
  - Transitions UI state based on Producer/Consumer completion
  - Waits for Audience ingestion completion before marking sync successful
  - Triggers delta sync message release after full sync completes
- **Configuration**: UI state update endpoints, Audience ingestion monitoring
- **Integration points**: Full Sync Producer/Consumer, Audience, UI state updates
- **Gotchas**: 
  - UI shows 'Successful' only after Audience confirms all records ingested; can take hours on large systems
  - No visibility into Audience ingestion delays; UI may appear stuck
- **Tribal knowledge**: Ingestion delay tracking is critical for user experience; communicate during full sync if delays expected

### Outbound Manager (Service)
- **Purpose**: Bridges frontend and external systems for outbound activity sync; handles 'Sync to [CRM]' option and coordinates outbound flow
- **Key files**: Feature flag checks, Integration Manager capability checks
- **Tech stack**: Microservice, capability query interface
- **Data flow**: User creates activity (email, form, etc.) with sync option → Outbound Manager queries Integration Manager for capability → sends to Audience Subscription Queue
- **Business rules**: 
  - Queries Integration Manager to verify integration supports email/form/task outbound
  - Instance-level checks verify actual capability (feature flags + connector spec)
  - Validates integration installed for this account
- **Configuration**: Feature flags (enable_outbound_email, enable_outbound_forms, etc.), Integration Manager endpoint
- **Integration points**: Frontend (email tool, forms), Integration Manager, Audience Subscription Queue
- **Gotchas**: 
  - Capability checks at feature selection time; if capability breaks, already-scheduled activities fail silently
  - Form outbound uses pre-defined outbound mappings, not form submission data directly; fields not in mappings not synced
- **Tribal knowledge**: Form outbound mapping was a political/organizational decision in 2022; customer preference against bidirectional attributes

### Audience Subscription Queue (ASQ) (SQS FIFO)
- **Purpose**: SQS FIFO queue collecting real-time events from Audience for outbound sync; maintains ordering per integration
- **Key files**: Queue configuration (not specified)
- **Tech stack**: AWS SQS FIFO
- **Data flow**: Audience fires event → placed on ASQ → ASQ Worker validates and routes → Kafka
- **Business rules**: 
  - FIFO ordering guaranteed per partition
  - Consent messages bypass batching (direct to SQS)
  - Activity events routed to batching pipeline
- **Configuration**: Queue name pattern (asq-<account_id>.fifo), visibility timeout
- **Integration points**: Audience (event source), ASQ Worker (consumer), Kafka (secondary queue)
- **Gotchas**: 
  - SQS message size limit 256 KB; batch production worker keeps batches <200 KB to stay safe
  - Dead letter queue required for failed messages
- **Tribal knowledge**: FIFO ordering critical for multi-partition integrations; understand Kafka partition key strategy

### Audience Subscription Queue Worker (ASQ Worker) (ECS Task)
- **Purpose**: Validates whether integration is installed and whether activity should be synced; routes valid messages to Kafka
- **Key files**: Integration installation checks, activity sync flags
- **Tech stack**: ECS task, queue consumer, router
- **Data flow**: ASQ message → validate integration installed → validate activity type synced → route to Kafka partition (partitioned by integration/installation)
- **Business rules**: 
  - Validates integration_installed(account_id, integration_type)
  - Checks feature flags for activity type support
  - Routes to Kafka partition keyed by (integration_id, installation_id)
- **Configuration**: Integration registry (installation check source), Kafka broker addresses, partition key strategy
- **Integration points**: ASQ (message source), Kafka (routed messages), Integration registry
- **Gotchas**: 
  - Installation validation failures cause messages to be dropped silently (no retry)
  - Kafka partition strategy critical for ordering; misconfiguration breaks message ordering
  - Multiple integrations on same account create multiple Kafka partitions (parallelism)
- **Tribal knowledge**: Partition key must remain consistent across ASQ Worker and Batch Production Worker; coordinate if changing

### Batch Production Worker (ECS Task)
- **Purpose**: Consumes Kafka partitions and creates batches of outbound events; sends batches to SQS FIFO queue
- **Key files**: Batching logic (200 KB threshold, 1 sec timeout)
- **Tech stack**: ECS task, Kafka consumer, batch producer
- **Data flow**: Kafka partition → group messages by integration → batch when 200 KB exceeded OR 1 second elapsed → send to SQS FIFO → Outbound Worker
- **Business rules**: 
  - Batches created when 200 KB exceeded OR 1 second timeout elapsed (whichever first)
  - 200 KB chosen to stay well below SQS 256 KB limit (safety margin)
  - One partition consumer per batch worker instance
- **Configuration**: Batch size threshold (200 KB, not user-configurable), batch timeout (1 sec, not user-configurable), SQS queue name pattern
- **Integration points**: Kafka (batches grouped by partition), SQS FIFO queue (batch destination), Outbound Worker (downstream consumer)
- **Gotchas**: 
  - 1 second batching timeout introduces ~1 second delay before external system sees activity (acceptable but important for real-time workflows)
  - If single message >200 KB, not batched (sent alone); connector must handle
  - Batching introduces exactly-once delivery challenge; relies on connector batch ID tracking
- **Tribal knowledge**: ~1 second delay is acceptable trade-off for efficiency; communicate to customers concerned about real-time

### Outbound Worker (ECS Task)
- **Purpose**: Picks batches from SQS FIFO queue, uses connector library to send to external system, handles retries via DLQ
- **Key files**: Connector library outbound calls, batch retry logic, DLQ handling
- **Tech stack**: ECS task, SQS consumer, retry engine
- **Data flow**: Batch from SQS FIFO → use connector library to send to external system → on success, delete from queue → on failure, retry with exponential backoff → on exhaustion, move to DLQ
- **Business rules**: 
  - Retries failed batches with exponential backoff (strategy not documented)
  - Tracks batch ID to prevent duplicates on retry (external system must do same)
  - Only opt-out consent messages (opt_in=false) retried on DLQ redrive; opt-in messages NOT retried
  - Consent messages sent individually (not batched)
- **Configuration**: Retry backoff parameters (not documented), DLQ name, Connector library selection based on integration type
- **Integration points**: SQS FIFO queue, Connector library, external CRM system, DLQ, Retry Driver Lambda
- **Gotchas**: 
  - Batch retry consistency loss: external system MUST track batch IDs to prevent duplicate activity creation
  - Failed batches can take hours before reaching DLQ (exponential backoff)
  - Opt-in consent messages left in DLQ if retry exhausted; no automatic redrive for these
  - Connector library errors not always recoverable; may require external system fix
  - No timeout mechanism documented; messages may accumulate indefinitely before retry
- **Tribal knowledge**: Batch ID tracking is contractual requirement with suppliers/partners; validate in acceptance tests

### Dead Letter Queue (DLQ) (SQS)
- **Purpose**: Collects failed outbound batches and consent messages for operational review and redrive
- **Key files**: None
- **Tech stack**: AWS SQS, monitoring/alerting system
- **Data flow**: Failed messages after retry exhaustion → DLQ → monitored via alerts → redriven at 6:00 AM daily by Retry Driver Lambda
- **Business rules**: 
  - Messages stay in DLQ indefinitely (infinite retry model, limitation)
  - Only opt-out consent and batch messages (with batch ID tracking) eligible for scheduled redrive
  - Opt-in consent messages manually handled (no redrive)
- **Configuration**: DLQ queue name, redrive schedule (6:00 AM daily), alerting thresholds
- **Integration points**: Outbound Worker (failed message source), Retry Driver Lambda (redrive mechanism), monitoring/alerting
- **Gotchas**: 
  - Infinite accumulation: messages never aged out; DLQ can grow unbounded
  - No manual redrive mechanism documented; only scheduled redrive at 6:00 AM
  - Batch failure patterns not automatically analyzed; must be manually reviewed
  - Permanent failures stay in DLQ indefinitely (e.g., broken external system API)
- **Tribal knowledge**: DLQ monitoring via alerts is critical; undiscovered DLQ messages cause integration health issues

### Retry Driver Lambda (AWS Lambda)
- **Purpose**: Redrive DLQ messages back to SQS FIFO queue at 6:00 AM daily; maintains infinite retry loop
- **Key files**: None
- **Tech stack**: AWS Lambda, CloudWatch Events trigger
- **Data flow**: 6:00 AM trigger → pull all messages from DLQ → re-queue to SQS FIFO → Outbound Worker retries
- **Business rules**: 
  - Triggered daily at fixed time (6:00 AM)
  - Redrive logic:
    - Batch messages: eligible if batch ID present (enables deduplication in external system)
    - Opt-out consent: eligible (opt_in=false)
    - Opt-in consent: NOT eligible (stays in DLQ)
- **Configuration**: CloudWatch Events schedule (cron: 0 6 * * ?), DLQ/SQS queue names
- **Integration points**: DLQ (message source), SQS FIFO queue (redrive destination), Outbound Worker
- **Gotchas**: 
  - Only 6:00 AM daily; if immediate redrive needed, manual intervention required
  - Opt-in messages never redriven; customer must manually handle if redrive needed
  - No idempotency check; same message can be redriven multiple times if visibility timeout issues occur
  - External system must handle batch ID tracking to prevent duplicates
- **Tribal knowledge**: Infinite retry model is intentional design; failures eventually resolved when external system fixes issues

### Consent Message Processing (Outbound)
- **Purpose**: Handles bidirectional consent changes between Audience and external systems; single-message routing (not batched)
- **Key files**: Consent message routing logic, opt-in/opt-out rules
- **Tech stack**: Microservice, SQS consumer
- **Data flow**: Consent change in Audience → bypass Audience Subscription Queue and Batch Production Worker → direct to SQS FIFO → Outbound Worker sends individual message → retries until successful
- **Business rules**: 
  - Consent messages sent individually (NOT batched, unlike activities)
  - Only opt-out messages (opt_in=false) retried on DLQ redrive
  - Opt-in messages NOT retried (safety: redriving opt-in without user action unsafe)
  - Bidirectional: Audience → CRM, CRM → Audience via webhooks
- **Configuration**: SQS queue name (consent-specific), retry strategy (infinite until external system confirms)
- **Integration points**: Audience (consent change source), SQS (singular message queue), Outbound Worker
- **Gotchas**: 
  - Asymmetry in retry: opt-out retried, opt-in not; confusing and poorly documented
  - No deduplication of consent changes; multiple changes queued as separate messages
  - CRM must respond to consent updates (no feedback loop documented)
  - Redrive operation loses eventual consistency guarantees; opt-out assumed safer direction
- **Tribal knowledge**: Asymmetry is intentional for compliance safety; document clearly in support runbooks

### Connector Library (Generic)
- **Purpose**: Generic interface specification for external systems to implement; defines fields, events, pagination, error handling, idempotency contracts
- **Key files**: S3 folder: generic connector API specification, S3 folder: implementation guide (last updated 2022, 2024 updates pending merge)
- **Tech stack**: API specification, code examples (language-agnostic)
- **Data flow**: Partner implements spec → Integration Manager queries via connector abstraction → schema/data returned in Justin format
- **Business rules**: 
  - Partners must implement complete spec (no cherry-picking features)
  - Batch ID tracking mandatory (no duplicates on retry)
  - Idempotency required for all operations
  - Error handling and timeout strategies specified
- **Configuration**: Partner provides endpoint URLs, auth credentials (encrypted via KMS)
- **Integration points**: Integration Manager (schema fetch), Full Sync Producer (pagination), Delta Sync Worker (webhooks), Batch Production Worker (batch receive), Outbound Worker (event send), external partners
- **Gotchas**: 
  - 2024 updates not yet merged; partners seeing outdated documentation
  - Implementation guide ambiguities lead to partner misunderstandings
  - Partners sometimes miss batch ID tracking requirement; causes duplicates in production
  - Spec validation at partner hand-off (75% test coverage required)
- **Tribal knowledge**: Spec changes require careful coordination with active partners; merge PRs before onboarding new partners

### Connector Library (Microsoft Dynamics Legacy)
- **Purpose**: Legacy system-specific connector for Microsoft Dynamics CRM built by supplier CRM Consultana; includes plugin/webhook solution
- **Key files**: Supplier: CRM Consultana plugin solution (S3), Dynamics webhook subscription configuration
- **Tech stack**: Dynamics plugin solution, webhooks, REST API
- **Data flow**: Legacy supplier plugin sends webhooks on contact changes → Delta Sync Worker processes → Full Sync Producer paginates contacts via supplier API
- **Business rules**: 
  - Supplier maintains plugin and webhook infrastructure
  - Supplier responsible for webhook reliability
  - Supplier escalations per SLA template
- **Configuration**: Supplier API endpoint (Parameter Store), webhook callback URL, supplier credentials (Secrets Manager)
- **Integration points**: Dynamics CRM, webhook subscriptions, supplier infrastructure (CRM Consultana)
- **Gotchas**: 
  - Supplier model is inflexible; feature additions require supplier engagement (delays)
  - Plugin solution proprietary; if supplier goes out of business, support issues
  - Commercial organization hasn't fully migrated to partner model (Sideshop); both coexist
- **Tribal knowledge**: Being phased out in favor of Sideshop partner model; understand which customers still use legacy

### Connector Library (Efficy Enterprise)
- **Purpose**: Connector for Efficy Enterprise CRM versions 12.0 and 12.1
- **Key files**: Not specified
- **Tech stack**: Generic or system-specific connector (TBD)
- **Data flow**: Similar to Dynamics/generic flow
- **Business rules**: 
  - Supports profile lists import (static and dynamic)
  - Terminology collision: 'query' = dynamic list, 'profile' = static list (opposite of Apsis terminology)
- **Configuration**: Efficy Enterprise API endpoint, credentials
- **Integration points**: Integration Manager, Full Sync Producer, Delta Sync Worker, Profile List Import
- **Gotchas**: 
  - Naming confusion: use Efficy-specific terminology when discussing this system
  - Profile list import holds all contacts in memory; no queue/retry logic (causes failures at scale)
- **Tribal knowledge**: Legacy system; one of the few without full modern connector implementation

### Profile Lists Import (Static and Dynamic)
- **Purpose**: Imports static or dynamic profile lists from external systems; tags existing profiles with list membership; supports one-time or recurring (6:00 AM daily via CloudWatch Lambda)
- **Key files**: Lambda function code (location not specified)
- **Tech stack**: Lambda, CloudWatch Events, in-memory processing
- **Data flow**: Profile list from CRM → Lambda fetches all list members (in-memory, no queue) → tags existing Audience profiles → subsequent import removes tags if contact no longer in list
- **Business rules**: 
  - Does NOT create profiles; only tags existing ones
  - Tracks which contacts received tags per operation (for clean removal on next import)
  - One-time or recurring (6:00 AM daily via CloudWatch Lambda)
- **Configuration**: CloudWatch Events schedule (6:00 AM for recurring imports), Lambda function timeout, maximum in-memory size
- **Integration points**: External CRM (list source), Audience (profile tagging), Profile list tracking (state storage)
- **Gotchas**: 
  - **CRITICAL: Holds all contacts in memory with no queue/retry logic**
  - Very large lists (>1 million) cause Lambda memory exhaustion or timeout
  - No retry logic; if Lambda times out mid-import, state is inconsistent (some tags added, others not)
  - Subsequent import tracking relies on previous state; if tracking lost, duplicate tags or missed removals
  - Gotcha: Efficy Enterprise naming collision: 'query' = dynamic list, 'profile' = static list (opposite of Apsis)
- **Tribal knowledge**: This has caused outages before; break large lists into smaller batches or implement queue-based approach

### Sync Conditions Filtering
- **Purpose**: Filters which records synced based on field conditions (e.g., active=true); applies during full/delta sync; supports optional deletion if condition no longer met
- **Key files**: Condition evaluation logic (in sync workers)
- **Tech stack**: Field expression evaluator
- **Data flow**: Record processed → evaluate condition expression → if false, skip (or delete if feature-flag enabled and message indicates condition failed)
- **Business rules**: 
  - Conditions are field expressions (e.g., active=true, status=eligible)
  - Applied during full sync and delta sync
  - Records failing conditions ignored or optionally deleted (feature-flag controlled)
  - Feature flag: Enable consent sync on condition failure (default: disabled)
- **Configuration**: Condition expression (stored per integration), feature flags (enable_delete_on_condition_failure)
- **Integration points**: Full Sync Consumer, Delta Sync Worker, Audience (optional deletion)
- **Gotchas**: 
  - Conditions evaluated only on delta; existing records not re-evaluated unless profile updated
  - Feature flag for deletion default off; enable carefully (data loss risk)
  - Condition logic must match external system semantics or records silently filtered
- **Tribal knowledge**: Deletion feature-flag is powerful but dangerous; test thoroughly before enabling on production customers

### Unified Data Export Service
- **Purpose**: Real-time CRM query result export via HTTP/2 streaming to support complex joins (e.g., CEOs of companies with event attendees)
- **Key files**: Unified Data export API implementation, Apache Columnar format conversion
- **Tech stack**: HTTP/2 streaming, Apache Columnar format, integration middleware
- **Data flow**: User creates CRM query (e.g., 'CEOs of companies with event attendees') → Email tool joins with segment → Audience connects to Integrations via HTTP/2 → Integrations streams CRM query results → Audience joins with profile data → Email personalized
- **Business rules**: 
  - CRM queries are user-defined (partners create them in CRM)
  - Results streamed progressively without loading all data into memory
  - Joins with Audience profile data in real-time
- **Configuration**: Feature flag: Enable Unified Data personalization (limited rollout)
- **Integration points**: External CRM (query source), Audience (stream destination, join operation), Email tool (personalization use), Athena (data warehouse source), React (real-time profile store)
- **Gotchas**: 
  - **NOT production-validated at scale; tested on one customer only**
  - Very large query results (millions of rows) can cause stream delays
  - If Audience-side join operation slow, delays email send (no async fallback)
  - If CRM query changes after email scheduled, personalization may reference stale data (no re-query on send)
  - No timeout mechanism documented
- **Tribal knowledge**: Feature not widely deployed; monitor closely on first large-scale rollout and have rollback plan ready

### Audience (Profile Store & CDP)
- **Purpose**: Unified profile store and CDP in Apsis One; stores customer profiles, attributes, consent, segment membership
- **Key files**: Not specified in integration domain context
- **Tech stack**: CDP platform (internal to Apsis One)
- **Data flow**: Inbound: External data → integration services → Audience updates. Outbound: Audience events → Subscription Queue → External systems
- **Business rules**: 
  - System of truth for customer attributes (except when external system explicitly designated)
  - Bidirectional consent (opt-in, opt-out)
  - Unidirectional attributes (external→Apsis only)
- **Configuration**: Audience endpoints (Parameter Store), integration authentication (internal, not specified)
- **Integration points**: Justin (inbound/outbound), Delta Sync Worker, Full Sync Consumer, Audience Subscription Queue, React, Athena
- **Gotchas**: 
  - Heavy dependency: nearly all integration services require Audience operational
  - Outage in Audience prevents inbound and outbound sync
  - Profile merging across keyspaces requires coordination with Audience
- **Tribal knowledge**: Biggest external dependency bottleneck during disaster recovery; coordinate activation first

### React (Real-time Profile Store)
- **Purpose**: Real-time profile lookup store; provides fast profile queries (used instead of Athena for real-time needs)
- **Key files**: Not specified
- **Tech stack**: Real-time data store
- **Data flow**: Profile updated in Audience → React updated in real-time → available for fast queries
- **Business rules**: 
  - Real-time consistency with Audience
  - Lower latency than Athena (batch)
- **Configuration**: Not specified in transcript
- **Integration points**: Audience (source), Athena (alternative)
- **Gotchas**: 
  - Latency is lower but SLA not documented
  - Not used directly by integrations; primarily for email tool personalization
- **Tribal knowledge**: Understand difference between React (real-time) and Athena (batch); choose based on latency needs

### Athena (Data Warehouse)
- **Purpose**: Data warehouse for batch exports and complex queries; ingests from React via congestion pipeline (slow, ~1 day latency)
- **Key files**: Not specified
- **Tech stack**: Data warehouse (SQL querying)
- **Data flow**: React data → congestion pipeline (1 day latency) → Athena → available for batch/complex queries
- **Business rules**: 
  - Batch-oriented; not suitable for real-time needs
  - ~1 day ingestion latency
- **Configuration**: Query timeouts (not specified)
- **Integration points**: React (source via congestion pipeline), Unified Data export (query data source)
- **Gotchas**: 
  - 1 day latency makes Athena unsuitable for real-time integrations
  - Congestion pipeline: data flow is slow and unpredictable
- **Tribal knowledge**: Understand latency trade-off; use React for real-time, Athena for batch

### CloudFormation Deployment Infrastructure
- **Purpose**: Infrastructure-as-Code templates for deploying integration services in phases (Base Infrastructure → Database → Supporting Infrastructure → Microservices)
- **Key files**: cloudformation/base-infrastructure.yaml, cloudformation/database.yaml (inferred), service templates
- **Tech stack**: AWS CloudFormation, YAML, Phase-based orchestration
- **Data flow**: Configuration → Phase 1 deployment → Phase 2 (database) → Phase 3 (supporting) → Phase 4 (microservices parallel batches)
- **Business rules**: 
  - Phase sequencing must be respected (Base must complete before Database, etc.)
  - Phase 4 microservices deploy in parallel batches (4-5 per batch) due to service independence
  - Queue decoupling in Phase 3 enables Phase 4 parallelism
- **Configuration**: AVS_PROFILE variable (determines nv-*.conf and .env.* files), snapshot identifier (database restore), secrets references
- **Integration points**: Parameter Store, Secrets Manager, ECR, RDS, Lambda, SQS, Kafka, Redis, SNS
- **Gotchas**: 
  - Database phase (Phase 2) is longest bottleneck (~30+ minutes); deploy separately if avoiding late blocking
  - Snapshot identifier parameter must be set BEFORE deployment or new empty database created instead of restore
  - Phase 4 parallelism ONLY safe due to intentional queue decoupling; modifying architecture breaks this
- **Tribal knowledge**: Deploy database first in isolation during disaster recovery; avoids discovering database issues after 40+ minutes

### Parameter Store (AWS Systems Manager)
- **Purpose**: Centralized storage of non-sensitive environment variables reused across CloudFormation templates and services
- **Key files**: cloudformation/environment variables/nv-<AVS_PROFILE>.conf
- **Tech stack**: AWS Systems Manager Parameter Store
- **Data flow**: Configuration file → Parameter Store → referenced by CloudFormation templates and services
- **Business rules**: 
  - Non-sensitive values only (credentials go in Secrets Manager)
  - Shared across all services (eliminates duplication)
  - Profile-specific (different values per environment)
- **Configuration**: File naming convention: nv-<PROFILE_NAME>.conf (exact match required), contains Audience URLs, CRM endpoints, service discovery endpoints
- **Integration points**: CloudFormation (template references), all microservices (environment variable source)
- **Gotchas**: 
  - File name must EXACTLY match profile name used in make deploy command (typos cause silent failures)
  - Discovering all required values is 'most annoying part' of disaster recovery (requires contacting multiple teams)
  - Legacy system required updating values in 15+ different locations; Parameter Store eliminates this
- **Tribal knowledge**: Automate endpoint discovery in future to reduce manual team contacts; implement service registry if possible

### Secrets Manager (AWS)
- **Purpose**: Store sensitive credentials required for integration infrastructure
- **Key files**: None (manual creation required)
- **Tech stack**: AWS Secrets Manager
- **Data flow**: Manual creation → CloudFormation references → runtime retrieval by services
- **Business rules**: 
  - Sensitive credentials only (OAuth IDs, database keys, API keys)
  - Cannot be defined in CloudFormation code (security best practice)
  - Manual creation required (no automation)
- **Configuration**: Microsoft Dynamics OAuth Client ID, Database key, service-specific credentials (discovered via trial-and-error during deployment)
- **Integration points**: CloudFormation (secret references), Broker Service (credential retrieval), KMS (encryption)
- **Gotchas**: 
  - Cannot be defined in CloudFormation; discovery is intentional trial-and-error (deploy → error → create secret → retry)
  - Missing secrets only surface after CloudFormation deployment attempts; no upfront validation
  - Secrets location (source of truth) scattered or in personal memory; document in Confluence
- **Tribal knowledge**: Suggested improvement: document secret structure in code (without values) to reduce discovery iterations

### KMS (Key Management Service)
- **Purpose**: Encrypt customer API credentials for generic connector integrations
- **Key files**: None
- **Tech stack**: AWS KMS single-region key
- **Data flow**: Customer credentials → encrypted via KMS → stored in database → decrypted by Broker Service at request time
- **Business rules**: 
  - Single-region key (current limitation)
  - Cannot decrypt in backup/disaster recovery accounts
  - Multi-region migration not possible for existing keys (new key must be created)
- **Configuration**: KMS key ID (single-region), IAM role permissions for Broker Service
- **Integration points**: Generic Connector (credential encryption), Broker Service (decryption), Database (credential storage), Disaster recovery account (limitation)
- **Gotchas**: 
  - **Single-region limitation: generic connector credentials cannot be decrypted in backup accounts**
  - **Workaround: customers must re-enter credentials after disaster recovery for generic connectors**
  - Legacy connectors (Efficy Enterprise) store credentials without KMS encryption and CAN be verified in backup accounts
  - AWS does not allow converting existing key from single-region to multi-region; new key must be created with multi-region enabled
  - Multi-region migration would require: (1) create new multi-region key, (2) replicate to backup accounts, (3) modify ALL database records to reference new key
- **Tribal knowledge**: KMS single-region limitation has been deprioritized; should be addressed before next disaster recovery exercise

### Broker Service
- **Purpose**: Make requests to customer CRM systems with decrypted API credentials
- **Key files**: Not specified
- **Tech stack**: Microservice (Lambda or managed service)
- **Data flow**: Retrieve encrypted credentials → decrypt via KMS → add to request headers → send to customer CRM
- **Business rules**: 
  - Decrypts credentials at request time (not at startup)
  - Adds credentials to outbound requests
  - Credentials never logged or cached in plain text
- **Configuration**: KMS key ID, CRM system endpoints (from Parameter Store)
- **Integration points**: KMS (decryption), Generic Connector (credential source), Customer CRM systems (outbound requests), Squid Proxy (request routing)
- **Gotchas**: 
  - KMS failures not discovered until credentials actually used (not until service deployment)
  - Request-time decryption may add latency; SLA impact not documented
- **Tribal knowledge**: Decryption-at-request-time avoids startup complexity but delays error discovery

### Squid Proxy
- **Purpose**: Proxy service for outbound integration requests
- **Key files**: Not specified
- **Tech stack**: Squid (proxy server)
- **Data flow**: Integration requests routed through Squid → customer CRM
- **Business rules**: 
  - All outbound CRM requests routed through proxy
  - Enables request inspection/filtering (if needed)
- **Configuration**: Proxy authentication (if configured), upstream CRM URLs
- **Integration points**: Broker Service (request source), Customer CRM systems (proxied destination)
- **Gotchas**: 
  - Proxy latency impact not documented
  - Proxy failures affect all outbound integrations
- **Tribal knowledge**: Used for request routing and inspection; understand proxy configuration for troubleshooting

### Integration Mapping Table (Database)
- **Purpose**: Stores configuration for which keyspace should receive data for each entity type in an integration installation
- **Key files**: Database table (location not specified)
- **Tech stack**: Relational database (RDS)
- **Data flow**: On data received from CRM: service queries mapping table (installation_id + entity_type) → returns keyspace_id → profile created/updated in that keyspace
- **Business rules**: 
  - Mapping is installation-specific and entity-type-specific
  - Default keyspace set during installation based on integration type
  - Reinstalling integration reverts any manual mapping changes to default
  - Email keyspace requires valid email format; CRM ID keyspace accepts any string
- **Configuration**: installation_id, entity_type, keyspace_id, default keyspace per integration type
- **Integration points**: Profile Update Service (mapping lookup), Profile Merge Worker (keyspace specification)
- **Gotchas**: 
  - Reinstallation resets mapping to default (breaking Solution 3 workarounds)
  - Manual database changes (Solution 3) are non-persistent across reinstalls
  - Email keyspace enforcement: if customer sends non-email value in email keyspace, lookup fails
- **Tribal knowledge**: Solution 3 (database hack) is single UPDATE statement but not persistent; requires reapplication if integration reinstalled

### Profile Merge Worker Service
- **Purpose**: Merges profiles from one keyspace into another to prevent duplicates and enable multi-identifier access to same profile
- **Key files**: Not specified
- **Tech stack**: Worker service (integration platform)
- **Data flow**: Receives merge request (source_keyspace_id, source_key, destination_keyspace_id, destination_key) → creates empty profile in destination if needed → merges into unified profile accessible via both identifiers
- **Business rules**: 
  - Merge operation is directional but final result is bi-directional queryable
  - Creates empty profile in destination keyspace if it doesn't exist
  - Merge is idempotent; merging already-merged profiles safe
  - Reuses proven infrastructure built for form submission workflows
- **Configuration**: Source/destination keyspace IDs, source/destination keys (email, CRM ID, etc.)
- **Integration points**: Profile Update Service, Form Submission Handler, Integration configuration system
- **Gotchas**: 
  - If destination profile doesn't exist, merge worker creates empty shell (expected, not error)
  - If email address null/invalid, merge fails; must handle gracefully
  - Automatic merge after every update (Solution 2) requires evaluation of performance impact
  - Rate-limiting may be needed if merge called too frequently
- **Tribal knowledge**: Already-built infrastructure; reusing for new use cases is cost-efficient and low-risk

### Keyspace System (CRM ID, Email, and Other Variants)
- **Purpose**: Logical data partitions that store profiles using different primary identifiers
- **Key files**: Not specified
- **Tech stack**: Apsis One core data storage (keyspace abstraction layer)
- **Data flow**: Each keyspace independently queryable by its identifier type. Profiles can be merged across keyspaces.
- **Business rules**: 
  - CRM ID keyspace: accepts any string value, no format validation, multiple profiles can have same email (as secondary attribute)
  - Email keyspace: requires valid email format, enforces one-to-one uniqueness (one email per profile)
  - Email keyspace is more restrictive; data quality must be high before using
  - Merge operation unifies profiles across keyspaces
- **Configuration**: Keyspace ID, keyspace type (CRM_ID, EMAIL, JOIN_CX, SILHOUETTE, EDEAL, ENTERPRISE, etc.), Primary identifier field
- **Integration points**: Integration Mapping Table (mapping specification), Profile Merge Worker (merge operations), Profile Update Service
- **Gotchas**: 
  - Email keyspace constraint: one email per profile (hard uniqueness); cannot have multiple profiles with same email
  - Switching keyspaces mid-stream creates duplicates; must coordinate with customer data migration
  - Integration installation determines default keyspace; reinstallation reverts to default
- **Tribal knowledge**: Keyspace selection is implicit during installation; no general UI for changing keyspace (only targeted solutions like Solution 2/3)

---

## 4. Cross-Cutting Concerns

### Authentication & Authorization
- **Intra-service**: Services authenticate to each other via IAM roles (AWS) or internal credentials (not fully specified)
- **External systems**: Credentials stored in Secrets Manager (sensitive) or Parameter Store (non-sensitive); KMS encrypts generic connector credentials
- **Supplier/Partner**: OAuth, API keys, or custom auth (connector-specific)
- **Database access**: Database user account with permissions to mappings table and configuration tables

### Error Handling & Logging
- **Inbound sync failures** (webhook processing): Messages logged to CloudWatch; optional dead-letter-queue-like buffering during full sync
- **Outbound sync failures**: Exponential backoff retries → DLQ after exhaustion → scheduled redrive at 6:00 AM
- **Schema fetch failures**: Integration Manager logs errors; customer sees broken schema dropdown
- **Mapping save failures**: Silent webhook failures possible; check logs for mapping sync issues
- **Consent update failures**: Messages to DLQ; opt-out messages retried, opt-in messages manually handled
- **Full sync producer hangs**: No timeout mechanism; requires manual intervention or Lambda timeout (kills task)
- **Disaster recovery**: Secrets Manager and Parameter Store failures discovered via trial-and-error deployment attempts

### Deployment & Configuration
- **CloudFormation phases**: Base Infrastructure → Database (30+ minutes) → Supporting Infrastructure → Microservices (parallel batches)
- **Service independence**: Enabled via decoupled SQS queues (Phase 3) and parallel deployment (Phase 4)
- **Configuration files**:
  - `nv-<AVS_PROFILE>.conf` (Parameter Store values): Audience URLs, CRM endpoints, service discovery
  - `.env.<AVS_PROFILE>` (make file env): ECR_HOST, optional database parameters
  - `Secrets Manager` (manual): OAuth credentials, encryption keys, service secrets
- **Make commands**: `make deploy AVS_PROFILE=<name>` (full deployment), `make deploy-db` (database only)
- **Rollback**: CloudFormation stack deletion or rollback to previous version (not fully documented)

### Monitoring & Alerting
- **DLQ monitoring**: SNS topics for failed batch alerts
- **Full sync progress**: UI state tracking (Downloaded → Finalizing → Successful)
- **Real-time health**: Integration service logs in CloudWatch
- **Operational reviews**: Formerly Monday meetings reviewing DLQ contents and integration health

### Disaster Recovery
- **Backup**: RDS database snapshots (stored in backup account)
- **Recovery procedure**:
  1. Obtain snapshot identifier from backup account
  2. Manually create Secrets Manager secrets
  3. Create nv-<PROFILE>.conf with service URLs (discovered via team contact)
  4. Create .env.<PROFILE> with ECR_HOST
  5. Execute `make deploy AVS_PROFILE=<profile>`
  6. Wait for Phase 2 (database restore ~30+ minutes)
  7. Wait for Phase 4 (microservices deployment)
  8. Verify Audience service operational (external dependency)
  9. Test integration workflows
- **Limitation**: Generic connector credentials cannot be decrypted in backup account due to single-region KMS key; customers must re-enter credentials
- **Last full simulation**: ~2 years ago; expect undiscovered interdependencies to surface
- **Documentation gaps**: Parameter Store values scattered; undocumented service dependencies; no consolidated runbook

---

## 5. Business Rules Reference

### Sync Direction & Scope
- **Attributes**: Unidirectional external→Apsis only (customers want CRM to be system of truth)
- **Consent**: Bidirectional (Audience↔CRM for compliance)
- **Activities (email, form, task)**: Outbound only (Audience→CRM)
- **Form submissions**: Create profiles in email keyspace; trigger CRM entity creation if integration supports outbound

### Record Filtering & Consistency
- **Sync conditions**: Field expressions (e.g., active=true) filter records during full/delta sync; non-matching records ignored or optionally deleted
- **Delta buffering**: During full sync, real-time webhook messages buffered on separate queue and processed after full sync completes (maintains eventual consistency)
- **Deduplication**: Relies on external system behavior; delta sync processes all webhooks (no built-in deduplication)

### Integration Installation & Configuration
- **One CRM integration per section** (due to CRM ID field collision); third-party integrations don't use CRM ID field and can coexist
- **Default keyspace**: Determined by integration type (CRM ID for Join CX, Efficy; email for web forms)
- **Reinstallation**: Reverts all manual configuration changes to default keyspace
- **Feature flags**: Specify capability support per integration; checked at runtime

### Mapping Constraints (Justin's Laws)
- **No loops**: Mappings cannot create circular dependencies
- **Consistency rules**: Enforced by Mappings Manager at mapping save time
- **Entity-type specificity**: Same CRM system can map different entity types to different keyspaces

### Outbound Activity Flow
- **Batching**: Activities batched when 200 KB exceeded OR 1 second elapsed (200 KB chosen to stay below SQS 256 KB limit)
- **Consent bypass batching**: Consent messages sent individually, not batched
- **Batch ID tracking**: External systems contractually required to track batch IDs to prevent duplicates on retry
- **Retry strategy**: Exponential backoff (strategy not documented) → DLQ after exhaustion → scheduled redrive at 6:00 AM daily
- **Consent redrive asymmetry**: Only opt-out messages (opt_in=false) retried; opt-in messages NOT retried (safety)

### Profile List Import
- **Tag existing profiles only**: Does not create new profiles, only tags existing ones
- **No queue/retry logic**: Holds all contacts in memory; large lists cause failures
- **One-time or recurring**: Can be run on-demand or scheduled daily at 6:00 AM
- **Subsequent imports**: Clean up tags from profiles no longer in list

### Keyspace Management
- **CRM ID keyspace**: Accepts any string identifier; multiple profiles can have same email (as secondary attribute)
- **Email keyspace**: Requires valid email format; enforces one-to-one uniqueness (one email per profile)
- **Profile merge**: Unifies profiles from source and destination keyspaces; result accessible via both identifiers
- **Switching keyspaces**: Creates duplicates if existing profiles in old keyspace; customer must handle manually

---

## 6. Known Issues & Workarounds

### Critical Issues

| Issue | Severity | Workaround | Status |
|-------|----------|-----------|--------|
| **CRM ID field collision prevents multiple CRM integrations per section** | HIGH | Install only one CRM integration per section; use third-party integrations for additional systems; post-2020 multi-entity systems use separate ID attributes (lead_id vs contact_id) | Design limitation; fall project for full refactor pending |
| **Profile list import holds all contacts in memory; fails on large lists and lacks retry logic** | HIGH | Break large lists into smaller batches (<1M contacts); manually retry failed imports; monitor logs for timeouts | Pending queue-based redesign |
| **KMS single-region key prevents decryption in backup accounts** | HIGH | Customers must re-enter generic connector credentials during disaster recovery; legacy connectors (Efficy Enterprise) unaffected | Deprioritized; create new multi-region key for future deployments |
| **Service dependencies on Audience not fully documented; integrations cannot verify functionality until Audience operational** | HIGH | Coordinate with Audience team to ensure deployment/operational status before verifying integration workflows | Disaster recovery bottleneck; need service dependency mapping |

### High-Impact Issues

| Issue | Severity | Workaround | Status |
|-------|----------|-----------|--------|
| **Infinite retries in DLQ; messages never aged out** | MEDIUM | Monitor DLQ via alerts; investigate root causes manually; consider implementing age-based purge policy | Design limitation; considered acceptable for low-volume DLQ scenarios |
| **Database snapshot identifier parameter must be set before deployment; if not set, new database created instead of restore** | MEDIUM | Set snapshot identifier BEFORE deploying database template; verify in CloudFormation parameters | Deployment procedure risk; no safeguard to prevent accidental new database creation |
| **Generic connector spec 2024 updates not yet merged; partners seeing outdated documentation** | MEDIUM | Merge pending PR immediately before onboarding new partners; notify active partners of updates | Pending merge; blocks new partner integrations |
| **File naming critical for deployment configuration; exact match required (nv-*.conf and .env.*); typos cause silent failures** | MEDIUM | Triple-check file names before deployment; implement validation in Make file | Deployment usability issue; could be caught by file existence checks |
| **Undocumented service interdependencies; last full disaster recovery simulation ~2 years ago** | MEDIUM | Conduct annual disaster recovery simulations; document all interdependencies as they surface | Operational risk; could paralyze recovery in actual crisis |
| **Parameter Store values must be manually discovered from multiple teams; no automated service endpoint discovery** | MEDIUM | Create comprehensive contact list for all dependent services; implement service registry for endpoint discovery | Disaster recovery bottleneck; described as 'most annoying part' of recovery process |

### Medium-Impact Issues

| Issue | Severity | Workaround | Status |
|-------|----------|-----------|--------|
| **Form outbound mapping uses pre-defined mappings, not form submission data directly** | MEDIUM | Ensure form data written to profile attributes first; use outbound mappings feature to control which attributes sync | Feature limitation; 2022 organizational decision; may be addressed in future |
| **Supplier model is inflexible; feature additions require supplier engagement causing delays** | MEDIUM | Migrate to partner-based model (Sideshop for Dynamics); build feature in generic connector spec for future integrations | Architectural shift underway; legacy supplier contracts complicate transition |
| **Consent opt-in messages never retried on redrive; only opt-out messages retried** | MEDIUM | Ensure opt-in changes sent initially; manually handle redrive for opt-in if needed | Design decision for safety; asymmetry poorly documented |
| **Pagination token issues in connector library can cause full sync to hang or infinite loop** | MEDIUM | Test connector pagination with large datasets (>1M records); add timeout to Full Sync Producer | Connector library quality assurance issue |
| **Email keyspace enforces one-to-one uniqueness; cannot use if duplicate emails in source system** | MEDIUM | Validate/deduplicate customer data before switching to email keyspace; handle existing duplicates manually | Data quality prerequisite not always discovered upfront |
| **Solution 3 (database hack for keyspace switching) non-persistent across reinstalls** | MEDIUM | Document workaround with installation ID and reapplication steps; create runbook for reinstall handling | Workaround limitation; not a supported feature |
| **Batch retry consistency loss on redrive; external system must track batch IDs** | MEDIUM | Ensure external system implements batch ID tracking per supplier/partner SLA; validate in acceptance tests | Contractual requirement; not guaranteed; causes duplicates if missed |

### Low-Impact Issues

| Issue | Severity | Workaround | Status |
|-------|----------|-----------|--------|
| **Unified Data export not production-validated at scale; tested on one customer only** | MEDIUM | Monitor closely on first large-scale deployment; have rollback plan ready | Feature flag controlled; limited rollout until validation complete |
| **Mappings Manager cache can become stale if mapping save webhook fails** | LOW | Monitor cache invalidation logs; redeploy Integration Manager if cache stale | Cache invalidation logic improvement needed |
| **Webhook node in Marketing Automation sends events one-by-one instead of batching** | LOW | Use standard outbound sync for batch operations; webhook node suitable for low-volume custom events | Alternative exists; performance suboptimal but acceptable |
| **Supplier contact information scattered or in personal memory** | LOW | Create comprehensive spreadsheet with supplier/partner contacts, active integrations, SLA status | Documentation/knowledge transfer issue |
| **Commercial organization hasn't fully executed migration from legacy Dynamics supplier to Sideshop partner model** | LOW | Coordinate with commercial team; maintain both connectors until migration complete | Organizational/commercial issue; technical support ongoing for both models |
| **Efficy Enterprise naming collision: 'query' = dynamic list vs. Apsis convention** | LOW | Use Efficy-specific terminology when discussing that system; document terminology difference | Terminology education/documentation issue |

---

## 7. Glossary

**Apsis One**: Parent marketing/CDP platform containing Audience, email tool, forms, events, marketing automation, integrations.

**Athena**: Data warehouse for batch exports and complex queries; ingests from React via congestion pipeline (1 day latency).

**ASQ Worker**: ECS task validating integration installation and routing valid messages to Kafka partition.

**Audience**: Unified profile store and CDP; central destination for inbound data, source of outbound events.

**Audience Subscription Queue (ASQ)**: SQS FIFO queue collecting real-time events from Audience for outbound sync.

**Attribute**: Customer data field (email, name, phone, custom fields); unidirectional external→Apsis only.

**Batch Production Worker**: ECS task grouping outbound events into batches (200 KB threshold, 1 sec timeout) and sending to SQS FIFO queue.

**Batch ID**: Unique identifier for outbound batch message; external system must track to prevent duplicates on retry.

**Bidirectional**: Data flow in both directions (Audience ↔ CRM); applies to consent only.

**Broker Service**: Makes requests to customer CRM systems with decrypted API credentials (via KMS).

**Connector Library**: System-specific (Dynamics, Efficy) or generic API abstraction layer for external system communication.

**Consent**: Customer preference (opt-in, opt-out) for marketing communications; bidirectional (Audience ↔ CRM).

**CRM ID keyspace**: Keyspace indexing profiles by CRM system's native ID; accepts any string identifier.

**CRM ID field collision**: All inbound mappings historically map to single CRM ID field; causes key space collision when multiple CRM integrations attempted on same section.

**Dead Letter Queue (DLQ)**: SQS queue collecting failed outbound batches/consent messages after retry exhaustion; monitored and redriven daily at 6:00 AM.

**Delta Sync**: Real-time synchronization of profile changes via webhooks.

**Disaster Recovery Account (DR Account)**: Separate AWS account for infrastructure restoration from backups.

**Duplicate Profile**: Same customer existing as multiple separate records in different keyspaces due to identifier mismatch.

**Email keyspace**: Keyspace indexing profiles by email address; requires valid email format and enforces one-to-one uniqueness.

**Entity Type**: Classification of objects in CRM system (person, lead, contact, member); each entity type can map to different keyspace.

**Eventual Consistency**: Data consistency model where all updates reach final consistent state eventually; achieved in delta sync via buffering during full sync.

**Feature Flag**: Runtime control enabling/disabling capabilities (e.g., outbound email, Unified Data personalization).

**Field Mapping**: Definition mapping external system field to Audience profile attribute.

**Full Sync**: Batch synchronization downloading all records from external system and syncing per current mappings.

**Full Sync Consumer**: ECS task processing queued records, applying mappings, batch-sending to Audience.

**Full Sync Manager**: ECS task managing full sync lifecycle and UI state transitions.

**Full Sync Producer**: ECS task paginating all contacts from external system and placing in Justin format on queue.

**Generic Connector**: Integration connector type allowing customers to connect any CRM system using encrypted API credentials; implements standardized spec.

**GolfAmore**: End customer (golf company) experiencing identifier mismatch problem with Join CX integration.

**Identity/Identifier**: Unique value for profile lookup within keyspace (email, CRM ID, etc.).

**Integration Mapping Table**: Database table storing (installation_id, entity_type) → keyspace_id configuration.

**Integration Manager**: Service fetching external system schemas and serving field mapping UI.

**Integration**: Configured connector instance (e.g., Join CX, Efficy) for specific customer.

**Intermail**: Third-party company operating Join CX customer loyalty system.

**Join CX**: Third-party customer loyalty/rewards system by Intermail; maintains customer data indexed by CRM ID.

**Justin**: Core integration middleware (named after Justin Timberlake); generic infrastructure for sync operations.

**Justin's Laws**: Consistency rules enforced by Mappings Manager preventing loops and invalid configurations.

**Keyspace**: Logical data partition storing profiles indexed by specific identifier type (email, CRM ID, etc.).

**KMS (Key Management Service)**: AWS encryption service for generic connector credentials; currently single-region only.

**Legacy Connector**: Older integration connector type (e.g., Efficy Enterprise 12.0) storing credentials without KMS encryption.

**Mappings Manager**: Service managing field/consent mappings and enforcing consistency rules.

**Multi-Entity Systems (post-2020)**: CRM systems using separate ID attributes per entity (lead_id, contact_id) avoiding CRM ID field collision.

**Multi-Region Key**: KMS key existing in multiple AWS regions; not current but required for disaster recovery scenarios.

**Native Consent**: Modern terminology for dedicated consent resource in external system (not stored as contact fields).

**Opt-In**: Customer preference to receive marketing communications.

**Opt-Out**: Customer preference to NOT receive marketing communications; safer direction for redrive.

**Outbound Manager**: Service coordinating 'Sync to [CRM]' option and outbound flow coordination.

**Outbound Worker**: ECS task sending outbound batches/consent messages to external CRM; handles retries and DLQ.

**Partner Model**: Engagement where external party implements generic connector spec independently; owns support/revenue.

**Parameter Store**: AWS Systems Manager service storing non-sensitive environment variables across CloudFormation templates.

**Profile Cloud**: Legacy name for Audience; originally positioned as middleware for bidirectional CRM sync.

**Profile List Import**: Lambda function importing static/dynamic lists from CRM and tagging existing Audience profiles.

**Profile Merge**: Operation unifying profiles from different keyspaces; result accessible via both identifiers.

**Profile Merge Worker Service**: Service processing merge requests and unifying profiles across keyspaces.

**React**: Real-time profile lookup store; faster than Athena for real-time queries.

**Record ID**: Identifier in consent/subscription update messages specifying which profile's preferences updated.

**Retry Driver Lambda**: Lambda function redriving DLQ messages at 6:00 AM daily; eligible: batch messages (with batch ID), opt-out consent.

**Secrets Manager**: AWS service for storing sensitive credentials (OAuth IDs, encryption keys, API keys).

**Service Independence**: Architectural principle enabling services to deploy in any order; enabled via decoupled SQS queues.

**Silhouette Keyspace / Edeal Keyspace / Enterprise Keyspace**: Domain-specific CRM ID keyspace variants for particular CRM systems.

**Single-Region Key**: KMS key existing in one AWS region only; cannot decrypt in backup accounts (current limitation).

**Sluice Worker**: Worker service processing and routing integration messages.

**Solution 1 (Keyspace Configuration UI)**: Rejected approach for full keyspace selection UI (too complex, over-engineered).

**Solution 2 (Automatic Profile Merge)**: Recommended long-term approach; automatically merge CRM-ID-keyed profiles into email keyspace after sync.

**Solution 3 (Database Configuration Hack)**: Immediate workaround for customer identifier mismatch; manual database UPDATE changing keyspace mapping.

**Solution 4 (No Change)**: Maintain current design; unsatisfactory to customer with identifier mismatch problem.

**Squid Proxy**: Proxy service routing outbound integration requests.

**Supplier Model**: Engagement where Apsis pays contractor to build/maintain system-specific integration (legacy, being phased out).

**Sync Condition**: Field expression (e.g., active=true) filtering which records synchronized; optionally delete if condition fails.

**Unidirectional**: Data flow in one direction only (external→Apsis for attributes); opposite of bidirectional.

**Unified Data Export Service**: HTTP/2 streaming service exporting complex CRM query results for email personalization.

**Virtual Consent**: Legacy terminology for consent stored as contact fields (not dedicated resource).

**Webhook**: HTTP callback sent by external system to Apsis when record created/updated; triggers real-time delta sync.

---

## 8. Confidence Notes

### High Confidence (Sourced from multiple sessions, corroborated)
- ✅ Architecture overview (Justin, sync workers, outbound flow, keyspace system)
- ✅ Integration Manager, Mappings Manager, Delta/Full Sync Worker purposes and data flows
- ✅ Outbound batching (200 KB, 1 sec timeout) and DLQ/Retry Driver Lambda operations
- ✅ CRM ID field collision and one-CRM-per-section limitation
- ✅ Consent bidirectionality, attribute unidirectionality
- ✅ Profile merge architecture and existing use for form submissions
- ✅ CloudFormation deployment phases and Phase 4 parallelism via queue decoupling
- ✅ Parameter Store vs. Secrets Manager distinction and configuration file naming
- ✅ KMS single-region limitation preventing disaster recovery decryption for generic connectors
- ✅ Full sync buffering delta messages during full sync for eventual consistency
- ✅ Opt-in consent messages NOT retried on redrive (only opt-out)

### Medium Confidence (Sourced from single session, logical inferences)
- ⚠️ Exact exponential backoff strategy for Outbound Worker retries (not documented; algorithm unspecified)
- ⚠️ Mappings Manager cache TTL value (assumed short but not explicitly stated)
- ⚠️ Full Sync visibility timeout duration for buffered delta queue
- ⚠️ Prometheus/monitoring query specifics for integration health (assumed CloudWatch)
- ⚠️ Exact database schema for integration_mapping_table (structure inferred from usage)
- ⚠️ Audience service multi-region deployment and failover behavior (not specified in integration context)
- ⚠️ Performance impact of Solution 2 (automatic profile merge) scaling characteristics
- ⚠️ Athena ingestion SLA and congestion pipeline detailed timing

### Low Confidence (Sourced from single session, incomplete details)
- ❓ Full Sync timeout mechanism existence and configuration
- ❓ Profile list import tracking state storage mechanism (database table structure)
- ❓ Sideshop vs. legacy Dynamics supplier commercial transition timeline
- ❓ Efficy Enterprise 12.0 connector implementation status (generic or system-specific)
- ❓ Marketing Automation webhook node architecture and relation to outbound pipeline
- ❓ CMS integration deprecation timeline and ongoing support status
- ❓ Exact file paths for S3 folders containing generic connector spec and implementation guide
- ❓ Specific Lime CRM relationship-removal-to-opt-out mapping issue details

### Gaps & Uncertainties
- **🚩 Disputed**: Opt-in consent redrive safety rationale (assumed but not definitively proven safer than alternative)
- **🚩 Missing**: Complete runbook for disaster recovery account setup from scratch (procedure documented but gaps in troubleshooting)
- **🚩 Missing**: Supplier/partner contact inventory and SLA status documentation
- **🚩 Missing**: Service dependency mapping (which services require which dependent services)
- **🚩 Outdated**: Generic connector implementation guide (2024 updates not yet merged; partners seeing stale docs)
- **🚩 Deprioritized**: Multi-region KMS key migration (known limitation; no timeline provided)
- **🚩 Non-persistent**: Solution 3 database hack workaround (breaks on integration reinstall; requires reapplication)

### Discrepancies & Contradictions
- **None identified** across three sessions; all architectural descriptions consistent

### Recommendations for KB Curation
1. Merge generic connector spec 2024 updates into primary documentation immediately
2. Create consolidated disaster recovery runbook with all discovered dependencies and troubleshooting steps
3. Document complete Secrets Manager secret list with discovery procedure
4. Create service dependency graph (which services require which external services operational)
5. Establish annual disaster recovery simulation schedule to uncover undocumented interdependencies
6. Implement service endpoint registry to eliminate manual team contact during recovery
7. Document supplier/partner contact information in Confluence (mitigate knowledge loss)
8. Add file existence checks to Make file to catch configuration file naming typos

---

## 9. Reference Artifacts

### Integration Types & Connectors Supported
- Microsoft Dynamics 365 (legacy supplier: CRM Consultana; new partner: Sideshop)
- Efficy Enterprise 12.0 / 12.1 (legacy; terminology quirks)
- Salesforce (assumed supported; details minimal)
- Shopify (legacy/deprecated; ecommerce focus)
- Lime CRM (supplier-maintained; relationship-removal issue)
- Generic Connector (partner-implemented; standard spec)
- Playable (third-party; no CRM ID dependency)
- Extramanial (loyalty/CRM partner; generic connector)
- Join CX / Intermail (third-party loyalty platform; key use case)

### CloudFormation Deployment Checklist
- [ ] Obtain database snapshot identifier from backup account
- [ ] Manually create Secrets Manager secrets:
  - [ ] Microsoft Dynamics OAuth Client ID (from Azure portal, Apsis Int'l account)
  - [ ] Database key
  - [ ] Service-specific credentials (discovered via trial-and-error)
- [ ] Create `nv-<PROFILE>.conf` in `cloudformation/environment variables/`
  - [ ] Audience endpoints (Parameter Store)
  - [ ] CRM API URLs (Parameter Store)
  - [ ] API gateway URLs (Parameter Store)
- [ ] Create `.env.<PROFILE>` file with:
  - [ ] **MANDATORY**: `ECR_HOST=<ACCOUNT>.dkr.ecr.<REGION>.amazonaws.com`
  - [ ] OPTIONAL: Database connection parameters
- [ ] Verify file names EXACTLY match profile name
- [ ] Execute: `make deploy AVS_PROFILE=<PROFILE>`
- [ ] Monitor Phase 1 (Base Infrastructure) - quick
- [ ] Monitor Phase 2 (Database) - 30+ minutes
- [ ] Monitor Phase 3 (Supporting) - Kafka, Redis, SQS, etc.
- [ ] Monitor Phase 4 (Microservices parallel batches)
- [ ] Coordinate with Audience team for operational status
- [ ] Test integration workflows

### Disaster Recovery Known Issues & Responses
| Symptom | Cause | Resolution |
|---------|-------|-----------|
| CloudFormation deployment fails immediately | Missing Secrets Manager secret | Create missing secret in Secrets Manager console; redeploy |
| Parameter Store values not found | nv-*.conf file name mismatch or location wrong | Check file name matches profile exactly; verify location is `cloudformation/environment variables/` |
| Phase 2 (database) takes >45 min without progress | Snapshot identifier not set; creating new database | Cancel deployment; set snapshot identifier; redeploy |
| Generic connector credentials can't be decrypted | Single-region KMS key in backup account | Inform customer they must re-enter credentials; legacy connectors unaffected |
| Services deploy but integration workflows fail | Audience service not operational | Coordinate with Audience team; defer integration testing |
| Services hang during Phase 4 | Service circular dependency (queue not decoupled) | Verify queue deployment isolated in Phase 3; check architecture changes |
| DLQ messages never retried | Opt-in consent messages not in redrive logic | Manually handle opt-in messages; batch/opt-out messages retried at 6:00 AM |

---

This knowledge base provides a comprehensive reference for engineering teams maintaining and extending the Apsis One Integrations unclassified subdomain. It synthesizes architectural decisions, component interactions, operational procedures, and tribal knowledge to enable effective onboarding, troubleshooting, and feature development.