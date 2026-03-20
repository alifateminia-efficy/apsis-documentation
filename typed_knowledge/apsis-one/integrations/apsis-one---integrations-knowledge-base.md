---
title: Apsis One - Integrations Domain Knowledge Base
generated: 2026-03-20T09:52:54.691Z
generated_by: n8n KT Refinement Pipeline v2
model_used: claude-sonnet-4-6
source: KT session transcripts (38 sessions)
purpose: AI coding agent context
---

# Table of Contents

- [Apsis One - Integrations Domain Knowledge Base](#apsis-one---integrations-domain-knowledge-base)
  - [1. Domain Overview](#1-domain-overview)
  - [2. Architecture Map](#2-architecture-map)
  - [3. Module Reference](#3-module-reference)
    - [Integration Manager (IM)](#integration-manager-im)
    - [Mappings Manager (MM)](#mappings-manager-mm)
    - [Delta Sync Manager (DSM)](#delta-sync-manager-dsm)
    - [Sluice Worker (SLW)](#sluice-worker-slw)
    - [Delta Sync Worker (DSW)](#delta-sync-worker-dsw)
    - [Full Sync Manager (FSM)](#full-sync-manager-fsm)
    - [Full Sync Producer](#full-sync-producer)
    - [Full Sync Consumer](#full-sync-consumer)
    - [Audience Subscription Worker](#audience-subscription-worker)
    - [Batch Production Worker](#batch-production-worker)
    - [Outbound Worker](#outbound-worker)
    - [Retry Driver Lambda (Dead Letter Queue Redrive)](#retry-driver-lambda-dead-letter-queue-redrive)
    - [Broker Service](#broker-service)
    - [Squid Proxy](#squid-proxy)
    - [Generic Connector](#generic-connector)
    - [Connector Libraries (CRM-specific)](#connector-libraries-crm-specific)
    - [Installer Interface](#installer-interface)
    - [Keyspace System](#keyspace-system)
    - [Profile List Sync Manager (Queries and Profiles)](#profile-list-sync-manager-queries-and-profiles)
    - [Unified Data (Data Provider Pattern)](#unified-data-data-provider-pattern)
    - [Consent Management](#consent-management)
    - [Clear Integration Endpoint](#clear-integration-endpoint)
    - [Disaster Recovery (DR) / Deployment](#disaster-recovery-dr-deployment)
  - [4. Cross-Cutting Concerns](#4-cross-cutting-concerns)
    - [Authentication & Secrets](#authentication-secrets)
    - [Error Handling](#error-handling)
    - [Logging](#logging)
    - [Deployment](#deployment)
    - [Support Workflow](#support-workflow)
  - [5. Business Rules Reference](#5-business-rules-reference)
    - [Data Direction](#data-direction)
    - [CRM as Master](#crm-as-master)
    - [Sync Conditions](#sync-conditions)
    - [Integration Limits](#integration-limits)
    - [Batch & Queue](#batch-queue)
    - [Idempotency](#idempotency)
    - [Form Submissions](#form-submissions)
    - [Profile Merges](#profile-merges)
    - [Keyspaces](#keyspaces)
  - [6. Known Issues & Workarounds](#6-known-issues-workarounds)
  - [7. Glossary](#7-glossary)
  - [8. Confidence Notes](#8-confidence-notes)
    - [High Confidence](#high-confidence)
    - [Medium Confidence / Verify Before Relying On](#medium-confidence-verify-before-relying-on)
    - [Gaps / Unresolved](#gaps-unresolved)
    - [Conflicts / Discrepancies](#conflicts-discrepancies)

---

# Apsis One - Integrations Domain Knowledge Base

## 1. Domain Overview

The Integrations domain ("Justin") connects Apsis One's marketing automation platform to external CRM systems (Microsoft Dynamics, FSC Enterprise, Lime, E-Deal/FSC Corporate, Tribe, Web CRM, and others) and third-party platforms (WooCommerce, Sleeknote, Zapier, I am Loyalty, Join CX). It is responsible for bidirectional synchronization of customer contact profiles, consent/subscription states, outbound marketing activity events, and form submissions. The domain owns both the real-time (delta sync via webhooks) and batch (full sync) data pipelines, manages field/consent mapping configurations, enforces data integrity rules (CRM as master for attributes; consent is the only bidirectional exception), and provides the Broker Service + Squid Proxy security layer for all outbound CRM traffic. It is architecturally organized as ~28–30 independent Go microservices deployed as ECS tasks on AWS ARM64 (Graviton) infrastructure.

---

## 2. Architecture Map

**Core middleware:** Justin (branding for the entire integration middleware platform)

**Inbound pipeline (CRM → Apsis):**
```
External CRM → webhook → Delta Sync Manager (DSM) → Sluice Queue (SQS)
  → Sluice Worker (SLW) [gate: blocks during full sync] → DSW Queue (SQS)
  → Delta Sync Worker (DSW) → Mappings Manager (MM) → Apsis HLS/Consent APIs
```

**Full sync pipeline (batch import):**
```
User trigger → Full Sync Manager (FSM) → spawns Producer + Consumer ECS tasks
  → Producer: CRM API (paginated) → temp SQS queue
  → Consumer: temp SQS → MM → Apsis HLS/Consent APIs
```

**Outbound pipeline (Apsis → CRM):**
```
Audience events → SNS → aud-sub-queue (SQS) → Audience Subscription Worker
  → Kafka (by event type: email/SMS/form/event-tool/MA-flow)
  → Batch Production Worker (one per Kafka topic, ~6 total)
  → outbound-worker-queue (SQS) → Outbound Worker
  → Broker Service (decrypt KMS creds) → Squid Proxy (domain whitelist)
  → External CRM

Consent changes → bypass Kafka → direct to outbound-worker-queue
```

**Security layer:** Broker Service + Squid Proxy in front of all outbound CRM calls.

**Data model:**
- Integration Manager (IM): CRUD for installations, credential storage, webhook registration
- Mappings Manager (MM): field mappings, consent mappings, sync conditions
- Aurora RDS (PostgreSQL): central state store
- Redis: caching
- Kafka: event distribution by type

**CI/CD:**
- GitHub Actions: unit tests on PRs
- AWS CodeBuild: Docker builds (ARM64 native) + CloudFormation deployment
- Branches: `develop` → staging, `beta` → beta (EU West only), `master` → prod EU West + APAC simultaneous

---

## 3. Module Reference

### Integration Manager (IM)
- **Purpose:** Handles integration installation, uninstallation, API key generation, credential storage, webhook registration with external CRM, schema fetching.
- **Key files:** `app/local/managers/main.go` (local dev entry point); `server/install` handler; `lib_config_connectors.go` (connector bootstrap config); `con_FSC` / `con_<connector>` database tables
- **Tech stack:** Go, ECS task (formerly Lambda), Aurora RDS
- **Data flow:** Frontend wizard → IM → validates CRM credentials → registers webhooks → stores `con_FSC`/credentials in DB → returns success. On uninstall: attempts CRM webhook deletion → if succeeds OR `ignore_external_system_errors=true`: deletes all DB records.
- **Business rules:**
  - Uninstallation is atomic (DB transaction). Fails entire flow if any step fails unless `ignore_external_system_errors=true`.
  - Webhooks subscription IDs must be stored before attempting deletion; never delete local IDs before confirming CRM deletion.
  - Operations Bus account deletion always passes `ignore_external_system_errors=true`.
  - Force-uninstall endpoint available for customer support (e.g., when customer rotated API key): `POST /integration-manager/force-uninstall`.
- **Configuration:** `ignore_external_system_errors` (boolean flag in request)
- **Integration points:** CRM APIs, Audience API, delegation tokens, Operations Bus (internal event bus)
- **Gotchas:** Installation sequence must be: create keyspace → bootstrap_attributes() → bootstrap_events() → register_webhooks() → store_credentials(). Reversing order breaks consent management. `lib_config_connectors.go` must include each connector for its bootstrap events to fire—omission causes silent consent sync failures (Dynamics consent timeline bug).
- **Tribal knowledge:** Refactoring in progress (epic created) to flatten nested function architecture. Current 4–5 level nesting makes `ignore_external_system_errors` threading unmaintainable.

---

### Mappings Manager (MM)
- **Purpose:** Stores and serves field mappings, consent mappings, and sync conditions for each integration; enforces Justin's Laws (no circular mappings).
- **Key files:** `apps/mappings-manager/`, `apps/mappings-manager/cloudformation.yaml`
- **Tech stack:** Go, ECS task, Aurora RDS
- **Data flow:** User configures → MM stores in DB; queried by Delta Sync Worker and Full Sync Consumer during message processing via `GET /mappings?account=X&section=Y&integration_id=Z`
- **Business rules:**
  - Only mapped fields are synced and included in webhook callbacks.
  - Field mappings are unidirectional (CRM → Apsis). Consent mappings are bidirectional.
  - Sync conditions: AND logic only (V1 current production). V2 (AND/OR, multiple operators) backend implemented but not deployed to all CRM systems.
- **Configuration:** Returns `apsis_field_id`, `crm_field_name`, direction, subscription IDs.
- **Gotchas:** Mappings Manager caching means changes may lag. Workers poll with caches; cache invalidation must propagate. Adding a field mapping post-installation does NOT retroactively sync that field for existing profiles.

---

### Delta Sync Manager (DSM)
- **Purpose:** Receives incoming webhook callbacks from CRM; validates webhook secrets/signatures; formats payloads; places messages on Sluice Queue.
- **Key files:** `apps/delta-sync-manager/`, `apps/delta-sync-manager/cloudformation.yaml`
- **Tech stack:** Go, ECS task, receives HTTP webhooks
- **Data flow:** CRM webhook → DSM validates (secret/HMAC/OAuth depending on connector type) → converts payload → places on Sluice Queue (SQS) → returns HTTP 200 immediately
- **Business rules:** Returns 200 to CRM immediately (async pattern); downstream success is decoupled. Incoming traffic bypasses Squid proxy; validated via webhook hash/secrets.
- **Configuration:** Webhook callback URL format: `https://integration.[env].apsis.cloud/dsm/v1/accounts/{accountId}/sections/{sectionId}/integrations/{integrationId}/contact-updates`
- **Integration points:** External CRM systems (inbound webhooks), Sluice Queue (SQS)
- **Gotchas:** FCC Enterprise 12.0 sends only changed fields (not full record). Sluice Worker handles this by calling back to CRM to fetch full contact—cannot be done in DSM due to synchronous nature and scale (FCC can send 6MB batches of 5,000 contacts at once).

---

### Sluice Worker (SLW)
- **Purpose:** Consumes Sluice Queue; gates real-time updates during active full sync; handles FCC 12.0 special case requiring full contact fetch.
- **Key files:** `apps/sluice-worker/` (inferred)
- **Tech stack:** Go, ECS task, SQS consumer
- **Data flow:** Message from Sluice Queue → check if full sync active (DB query) → if active: set visibility timeout (few minutes) → message returns to queue for retry → if inactive: [for FCC 12.0: callback to CRM for full record, evaluate sync conditions] → forward to DSW Queue
- **Business rules:**
  - Full sync blocking is mandatory for eventual consistency. Real-time updates during full sync are held, not dropped.
  - FCC 12.0 callback goes through Squid proxy.
  - Entity type in URL path is for business logic; query parameter (e.g., `?entity=lead`) is for debugging filtering only.
- **Gotchas:** This is intentional but acknowledged as "a solution we wish we didn't have." Without it, full sync historical data can overwrite newer real-time changes.

---

### Delta Sync Worker (DSW)
- **Purpose:** Consumes DSW Queue; applies field/consent mappings; calls Apsis HLS and Consent APIs to update profiles.
- **Key files:** `apps/delta-sync-worker/` (inferred)
- **Tech stack:** Go, ECS task, SQS consumer
- **Data flow:** Message from DSW Queue → queries MM for mappings → transforms CRM data → calls Apsis HLS API (attributes) → calls Apsis Consent API (subscriptions) → on 429: message returned to queue for retry
- **Business rules:** Phone number format validation: invalid formats silently skipped (not fatal). FCC Enterprise 12.1 returns HTTP 200 for business logic errors—must parse response body.
- **Integration points:** Mappings Manager, Apsis HLS API, Apsis Consent API (outbound through Squid proxy)
- **Gotchas:** Sync conditions evaluated here (client-side in Apsis, not CRM-side). All records retrieved from CRM, then filtered. This causes crashes for 2M+ contact customers—fix requires pushing sync conditions server-side to CRM.

---

### Full Sync Manager (FSM)
- **Purpose:** Customer-facing service; manages full sync lifecycle; spawns dedicated Producer and Consumer ECS tasks on-demand per job.
- **Key files:** `apps/full-sync-manager/`, `apps/full-sync-manager/cloudformation.yaml`; `integration/fs/worker`
- **Tech stack:** Go, ECS task, Aurora RDS
- **Data flow:** User clicks Start → FSM creates sync record in DB → spawns Producer ECS task + Consumer ECS task → monitors completion via DB flags → transitions states: Running → Finalizing → Successful
- **Business rules:**
  - Full sync spins up dedicated stack per job (avoids FQSOA queuing issues). ⚠️ Benjamin acknowledges "a bit overkill."
  - Full sync completion: Consumer detects empty queue AND `total_is_known=1` flag set by Producer. If Consumer times out (~8 hours) without Producer completing, marks as timed out.
  - Audience consent export (optimization to skip unchanged consents): 15-minute polling timeout. If timeout, sync continues without optimization.
- **Gotchas:** No callback architecture for Audience exports—only timeout detection. Consumer can have a race condition (concurrent map writes) that manifests as exit code 2; unchanged for 2+ years but has reappeared. Exit code 2 = crash (not startup failure). Check Consumer CloudWatch logs, not just ECS exit code.

---

### Full Sync Producer
- **Purpose:** Temporary ECS task; downloads all CRM contacts paginated; pushes to temp SQS queue; sets `total_is_known=1` on completion.
- **Key files:** `integration/fs/worker` (same module as FSM)
- **Tech stack:** Go, ECS task (temporary)
- **Data flow:** FSM spawns → paginates CRM API (5 concurrent threads by default) → pushes to temp SQS queue → sets DB flag on completion
- **Business rules:** Generic connectors: data arrives in required format directly. Legacy connectors (Dynamics, FSC): data goes through `mutate_data_to_match_justin_formats()` transformation.
- **Gotchas:** Do NOT cache contacts in-memory for comparison (crashes at 2M+ contacts). Must stream messages to SQS immediately. In-memory cache removal is pending implementation.

---

### Full Sync Consumer
- **Purpose:** Temporary ECS task; consumes temp SQS queue; applies mappings; calls Apsis APIs; triggers Audience consent export optimization.
- **Key files:** `integration/fs/worker`
- **Tech stack:** Go, ECS task (temporary), Redis
- **Data flow:** Temp SQS queue → queries MM for mappings → transforms → calls Apsis HLS/Consent APIs → on 429: message returned to queue → detects empty queue + `total_is_known=1` → signals FSM to cleanup
- **Gotchas:** Consumer stops if Producer fails; unknown if Producer stops if Consumer fails. Concurrent map write race condition is known (exit code 2). Audience export optimization failure (15-minute timeout) is not a sync failure—just proceeds less efficiently.

---

### Audience Subscription Worker
- **Purpose:** Consumes aud-sub-queue (SQS); validates events (requires profile CRM ID + active installation + feature released); enriches with outbound mapped_fields; routes to Kafka or direct to outbound-worker-queue for consent.
- **Key files:** `audience-subscription-worker` (service directory)
- **Tech stack:** Go, ECS task, SQS consumer, Kafka producer
- **Data flow:** SNS → aud-sub-queue → worker validates 3 conditions → transforms to standard message → consent events: direct to outbound-worker-queue → other events: Kafka topic (by type: email/SMS/form/event-tool/MA-flow)
- **Business rules:** Three validation checks before any processing: (1) profile has CRM ID, (2) integration is installed on section, (3) feature is released to production. All three required.
- **Gotchas:** File imports often corrupt CRM IDs (Excel converts large integers to floats: `123,500,000` becomes invalid). Results in mass discard of events. Consent events bypass Kafka batching—sent directly.

---

### Batch Production Worker
- **Purpose:** One instance per Kafka topic (6 total); accumulates events into batches; flushes at 200KB or 30-second timeout; routes to outbound-worker-queue (SQS).
- **Key files:** `batch-production-worker` (same Docker image for all 6 instances, differentiated by Kafka topic config)
- **Tech stack:** Go, ECS task, Kafka consumer, SQS producer
- **Data flow:** Kafka topic → accumulate events by installation (account+section+integration ID) → batch ID (GUID) generated → SQS → Outbound Worker
- **Business rules:** Batch size ≤ 200KB (SQS limit is 256KB). Time threshold: 30 seconds. Kafka partitioned by integration ID (one partition per integration)—single partition per integration is a known scaling bottleneck.
- **Gotchas:** Kafka messages not visible in AWS Console. Batch ID is the primary debugging tool—always get it to trace payload + outcome in CloudWatch.

---

### Outbound Worker
- **Purpose:** Consumes outbound-worker-queue; routes to CRM via Broker Service + Squid Proxy; handles retries; routes failures to Dead Letter Queue.
- **Key files:** `outbound-worker` (service directory); `lib/sqs.go` (backoff logic)
- **Tech stack:** Go, ECS task, SQS consumer
- **Data flow:** Batch from SQS → Broker Service (credential decryption + header injection) → Squid Proxy (domain whitelist check) → CRM API
- **Business rules:**
  - CRM IDs and identifying fields per system: E-Deal: `per_mail` (email), `person_id`; Dynamics: `email1`; FSC Enterprise 12.0: profile entity (workaround—no native consent concept).
  - SQS message group ID: `account_section + integration_id + crm_id` (prevents single customer from blocking others).
  - Backoff sequence (current, ~18 hours total): 15s, 15s, 1m, 5m, 15m, 1h, 4h, 6h. Refactoring to ~5 hours using `ApproximateReceiveCount` (switch-case, max 75 retries).
- **Gotchas:** SQS FIFO 20,000-message scan limit: if first 20K messages all fail from one customer, messages beyond position 20K from other customers are blocked. Emergency fix: blacklist customer in `processMessage()` (log + return nil), notify SOC, customer performs full sync.

---

### Retry Driver Lambda (Dead Letter Queue Redrive)
- **Purpose:** Daily 6:00 AM UTC—redrives failed messages from DLQ back to processing queue, with consent opt-in filtering.
- **Tech stack:** AWS Lambda, CloudWatch Events (cron)
- **Business rules:**
  - **NEVER** redrive consent opt-in messages (would double-opt-in).
  - Safe to redrive: consent opt-out, batch activity messages (external system must handle idempotency via batch ID).
  - No upper limit on retry count (only 14-day SQS retention hard cap).
- **Gotchas:** 14-day SQS retention is the hard cap. Messages lost after 14 days.

---

### Broker Service
- **Purpose:** Decrypts customer CRM API credentials (KMS for Generic Connector; plaintext for legacy) and injects into `X-API-Key` header before forwarding to Squid proxy.
- **Key files:** `backend/internal-broker/` (inferred); stored in Justin repository
- **Tech stack:** Go, ECS task, AWS KMS
- **Data flow:** Service request → Broker looks up encrypted credential in DB → KMS decrypt (Generic Connector only) → adds `Authorization`/`X-API-Key` header → passes to Squid proxy
- **Configuration:** Management key: `integration.internal_management_key` stored in Secrets Manager (loaded at startup, requires restart to rotate). Broker accessible internally at `broker-internal.apsis.io`. Parameters: `X-Account-ID`, `X-Section`, `X-Integration`, `X-Justin-Url`, `Authorization` (management key).
- **Business rules:**
  - Generic Connector: KMS-encrypted credentials (developers cannot read plaintext).
  - Legacy connectors (FCC Enterprise, Lime): credentials in plaintext in `connections_fsc` table—accessible for debug.
  - Management key is the same key used for force-uninstall.
- **Gotchas:** Rotation requires broker restart (~20 seconds downtime). FCC Enterprise legacy stores credentials plaintext; Generic Connector always uses KMS.

---

### Squid Proxy
- **Purpose:** HTTP-layer egress control; enforces domain whitelisting for all outbound integration traffic; prevents code injection attacks from exfiltrating data to unauthorized domains.
- **Key files:** `cloud-formation/squid-proxy/Dockerfile`, `cloud-formation/squid-proxy/squid.conf`, `cloud-formation/squid-proxy/allowed_staging.txt`, `cloud-formation/squid-proxy/allowed_prod.txt`, `cloud-formation/squid-proxy/squid_helper`, `cloud-formation/squid-proxy/Makefile`
- **Tech stack:** Squid proxy, Docker, ECS, CloudFormation
- **Data flow:** Service request (routed via `HTTP_PROXY`/`HTTPS_PROXY=localhost:3128` env vars) → Squid evaluates ACLs: static allowlist (AWS, Apsis internal, OAuth providers) → dynamic helper script queries `allowed_domains` DB table per customer → allow (TCP_TUNNEL/200) or block (TCP_DENIED/403)
- **Configuration:**
  - Squid port: 3128
  - `AWS_PROFILE` env var during `make build` selects allowlist file: `allowed_staging.txt`, `allowed_prod.txt`, `allowed_dr.txt`
  - Static allowlist: baked into Docker image at build time. Dynamic: `allowed_domains` DB table populated during integration installation.
  - Cache TTL: 5 minutes (both hits and misses).
  - Wildcard matching: single level only (`*.apsis.cloud` matches `folder.apsis.cloud` but NOT `sub.folder.apsis.cloud`).
- **Gotchas:**
  - HTTP 403 from Squid is the most common integration debugging failure point. Always check Squid logs (`TCP_DENIED`) first.
  - Domain format in allowlist must exclude scheme, port, path (`login.salesforce.com` not `https://login.salesforce.com:443`)—violations silently ignored.
  - Allowlist changes require Docker rebuild + `make deploy` (baked in at build time).
  - `--force-new-deployment` flag is mandatory in `make deploy`; ECS sometimes doesn't pick up latest image without it.
  - Microsoft Entra/Azure AD auth servers must be in static allowlist for Dynamics integrations.
  - Incoming CRM webhooks bypass Squid entirely (Delta Sync Manager handles inbound directly).

---

### Generic Connector
- **Purpose:** Standardized REST API contract that CRM systems implement; eliminates need for per-CRM custom adapters; all conforming systems share same Apsis code path.
- **Key files:** `lib/connectors/generic/assets/generic-api-spec.yaml`, `lib/connectors/generic/assets/justin-generic-webhook.yaml`; distributed at `https://integration-files.appsis.one/`
- **Tech stack:** OpenAPI 3.0 spec (YAML), Go interfaces generated from spec
- **Data flow:** API-first: spec → generated interfaces → implementation → deploy → publish to S3
- **Business rules:**
  - Webhook auth: HMAC-SHA256 (`X-Signature` header). CRM hashes request body + secret.
  - All CRM systems must return proper HTTP status codes (404 for not found, etc.—not always 200 with error in body like FCC 12.0 does).
  - Feature capability declarations via `/integrations` endpoint (per-instance, not per-account).
  - Adding optional properties: no version bump needed. Removing required properties: requires new API version.
  - Supports: `/records` (paginated, field filtering, sync conditions), `/consents`, `/integrations` (capabilities), webhook registration.
- **CRMs using generic connector:** Dynamics (via Sideshop), E-Deal (FSC Corporate), Magento, FSC Enterprise 12.1, Tribe, Webserum 2
- **Gotchas:** S3 bucket contains only DEPLOYED spec, not development-branch version. For in-progress features, send spec manually to partners. Stale e-commerce endpoints exist in `api_spec.json`—ignore them.

---

### Connector Libraries (CRM-specific)
- **Purpose:** CRM-specific adapters that translate between Justin's expected format and each CRM's API.
- **Key files:** `lib/connectors/FSC_Enterprise_2/installer.go`, `lib/connectors/FSC_Corporate/installer.go`, `lib/connectors/Dynamics/installer.go`, `lib/connectors/[connector_name]/installer.go`
- **Tech stack:** Go
- **Business rules:**
  - `integration_id` (e.g., `FSC Corporate`) = internal identifier used in API contracts, logs. Never changes.
  - `display_name` (e.g., `E-Deal`) = customer-visible name used in folder creation, UI. Easy to change.
  - `additional_entities` section defines lead/silhouette support per connector.
- **Gotchas:** FSC Enterprise 12.0 default date returns `1899-12-31` for empty dates—triggers MA flows for "100-year-old" contacts. FSC Enterprise 12.0 returns integers as stringified `'123'` instead of `123`. Always check type handling.

---

### Installer Interface
- **Purpose:** Go interface defining required functions all connectors must implement: `register_webhooks()`, `unregister_webhooks()`, `store_credentials()`, `remove_credentials()`, `create_folder()`, `bootstrap_attributes()`, `bootstrap_events()`.
- **Key files:** `libraries/connectors/types`
- **Gotchas:** Adding a function to the interface appears trivial (3 minutes) but requires implementation in all 4 connector types (FSC Enterprise, Dynamics, Lime, Generic)—2+ days total work.

---

### Keyspace System
- **Purpose:** Logical namespaces for storing profiles by identifier type; each CRM integration gets its own keyspace(s).
- **Tech stack:** Profile storage layer (Apsis One platform)
- **Data flow:** Installation → creates main keyspace → loops additional entity types → creates entity keyspaces; stores discriminators in DB
- **Business rules:**
  - Keyspace discriminator format: `integrations:keyspaces:<8-char-hash>:<crm-logical-name>[:<entity-name>]` (8-char hash = first 8 chars of section discriminator hash)
  - Keyspaces are **never deleted** on uninstallation (preserves historical events; reused on reinstall).
  - Cross-keyspace merges: only allowed within same CRM installation's entity types (lead ↔ contact). Never merge CRM keyspace with email keyspace.
  - Merges must originate from CRM, not Apsis, to maintain CRM consistency.
  - Email keyspace: requires valid email address format; single-email-per-profile uniqueness constraint.
  - CRM ID keyspace: accepts any string.
- **Supported entity types per CRM:** E-Deal: contact + silhouettes; FSC Corporate: contact + silhouettes; FSC Enterprise 12.1: contact only; Tribe: contact + leads; Dynamics (legacy): contact; Dynamics (Sideshop): contact + leads
- **Gotchas:** Keyspace mapping table (`integration_mappings`) is reset on reinstallation. Non-standard keyspace configurations (e.g., remapping to email keyspace for Golfamore/Intermail) must be documented in Confluence and must be re-applied after reinstalls.

---

### Profile List Sync Manager (Queries and Profiles)
- **Purpose:** Imports external CRM saved queries/segments into Apsis as tagged profile groups; supports one-time and recurring imports.
- **Tech stack:** Go, ECS task, CloudWatch scheduling
- **Data flow:** Trigger → fetch list from CRM → hold in memory → tag matching profiles in Apsis
- **Business rules:**
  - List import does NOT create new profiles—only tags existing profiles.
  - One worker per job per integration (serial, no parallelization)—intentional to avoid overwhelming customer infrastructure.
  - Tag name = integration name (e.g., `FEC Enterprise`).
- **Gotchas:** Memory-hold during import can be problematic for very large lists. Pre-existing CRM queries required; custom query creation is complex.

---

### Unified Data (Data Provider Pattern)
- **Purpose:** Enables cross-system complex queries by joining Audience profile data with external CRM data at query time via HTTP/2 streaming.
- **Tech stack:** HTTP/2 streaming, Apache columnar format (Arrow/Parquet), Athena
- **Data flow:** Audience export + external query → Integrations Data Provider HTTP/2 endpoint → external CRM paginated results → translated to columnar format → streamed back → Audience joins on-the-fly
- **Business rules:** Avoids flattening external system data into profiles (Athena storage cost). Joins at query time.
- **Gotchas:** Production-ready but only one real customer used it (Maxo project). Not battle-tested at scale. Partners must implement HTTP/2 endpoint separately from basic generic connector.

---

### Consent Management
- **Purpose:** Bidirectional consent synchronization between Apsis and CRM systems.
- **Business rules:**
  - Consent is the ONLY bidirectional data type. All attributes are unidirectional (CRM → Apsis).
  - `virtual consent` = boolean fields on contact card (legacy terminology). `native consent` = dedicated consent resource (modern).
  - Consent outbound bypasses Kafka/batching—sent directly from Audience Subscription Worker.
  - **Never redrive consent opt-in messages from DLQ.** Only opt-out and batch activity messages safe to redrive.
  - Auto-mapping (`automatic_consent_mappings = true`): generates virtual mappings dynamically for Magento (only current user). Sets both `update_email=true` AND `update_sms=true`. Bug in `message_processor` line 426: AND NOT clause fails when both flags true—fix is to remove AND NOT clause.
  - FCC Enterprise processes consent outbound in batches (couple-minute delay).
- **Gotchas:** Consent checkboxes in pre-filled forms not currently pre-filled with existing consent values (known gap). Only mapped subscriptions trigger CRM updates.

---

### Clear Integration Endpoint
- **Purpose:** Forced deletion of integrations when CRM-side uninstallation fails.
- **Configuration:**
  - URL: `https://integrations.apsis.one/accounts/{account_id}/sections/{section_id}/integrations/{integration_id}`
  - Auth: `Authorization: Bearer <Delete Integration Secret Key>` (from Secrets Manager)
  - Params: `?ignore_external_system_errors=true` and/or `?ignore_audience_errors=true`
- **"Holy Trinity":** Account ID (or name for pre-UID migration), Section ID, Integration ID (exact format e.g., `FSC Enterprise 12.0` not `FSC Enterprise`)
- **Business rules:** Webhooks on CRM side NOT deleted. Must contact customer to manually remove CRM webhooks after forced deletion.
- **Gotchas:**
  - Account name/ID typos = silent 401 error with no detail.
  - Integration ID must match exact stored format.
  - Pre-UID migration customers use account names instead of IDs.
  - Never log or expose Delete Integration Secret Key. Never include in Postman collections.
  - Squid proxy config entries may hang during deletion—`ignore_external_system_errors=true` handles this.

---

### Disaster Recovery (DR) / Deployment
- **Purpose:** Infrastructure-as-code for complete environment reconstruction.
- **Key files:** `env.<PROFILE_NAME>.conf` (parameter store values), `<PROFILE_NAME>.env` (AWS env, contains `ECR_HOST`), `Makefile` (root)
- **Configuration:**
  - `ECR_HOST` format: `<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com`
  - `AWS_PROFILE` must exactly match filenames (e.g., `staging` → `env.staging.conf` + `staging.env`)
  - `make deploy` from root = rebuilds ALL 28-30 services (~40 min). Always deploy from service directory for individual service.
  - `make deploy db` = database only (test DB restore independently before committing to full deploy).
- **Known issues:** KMS single-region key prevents Generic Connector credential decryption in DR account. Legacy connectors (plaintext creds) CAN be verified in DR. Fix: migrate to multi-region KMS key (deprioritized).
- **Gotchas:**
  - System hasn't been deployed from scratch in 2+ years—expect undocumented dependencies.
  - Audience service is ALWAYS a blocker for validation—must deploy Audience before testing integrations.
  - Dynamics O Client ID must come from Apsis International AB Azure account (NOT FSC account).
  - Secrets must be created manually in Secrets Manager before deployment. Deployment tells you which are missing.

---

## 4. Cross-Cutting Concerns

### Authentication & Secrets
- Customer CRM credentials: KMS-encrypted for Generic Connectors (Broker Service decrypts); plaintext for legacy connectors (FCC Enterprise, Lime)
- Webhook secrets: per-installation random UID generated by Apsis; HMAC-SHA256 for Generic Connector; API key header for legacy
- Delete Integration Secret Key: in Secrets Manager only; never in code, Postman, or logs
- GitHub CICD token (`integrations-ma-cicd`): in AWS Parameter Store AND CodeBuild source provider credentials—both must be updated on rotation
- OAuth apps (Microsoft Dynamics): `Apsis 1 EU Prod`, `Apsis 1 APAC Prod` in `Apsis@on.microsoft.com` Azure tenant; Application ID is public (in frontend config), Client Secret is backend-only (Secrets Manager/AVS)

### Error Handling
- FCC Enterprise 12.1 returns HTTP 200 for business-logic failures—always parse response body
- Exit code 2 = ECS task crash (not startup failure); check CloudWatch Consumer logs for actual error
- HTTP 403 from Squid = domain whitelist issue (most common cause)
- HTTP 400 from CRM = usually CRM-side configuration, not Apsis bug (downgrade to warning)
- SQS exponential backoff: ~2s, 1m, 5m, 15m, 1h, 3h, 5-6h per message

### Logging
- CloudWatch log groups per service (one per service type)
- Integration key format for log search: `{account_id}/{section_id}/{integration_id}`
- Batch ID (GUID): primary debugging tool for outbound pipeline
- Do NOT log webhook payload data from rejected/orphaned requests (GDPR compliance)
- Old Lambda log groups from pre-ECS era still exist in CloudWatch—ignore them

### Deployment
- Environments: staging (develop branch), beta (beta branch, EU West only), prod EU West + prod APAC (master branch, simultaneous)
- ECS rolling deployment: ~3-4 minutes (image pull + container start + LB registration + old container drain)
- `make deploy` from root = all services sequential; `cd apps/SERVICE && make deploy` = single service
- CloudFormation stack name must match service name exactly to update (not create new)
- GitHub Actions: PR checks (unit tests, Sigrid code quality)
- AWS CodeBuild: Docker builds (ARM64 native) + deployment

### Support Workflow
- All customer support issues must flow through SOC/support channel (not directly to developers)
- Zero-point stories for quick fixes (<1 hour effort)
- CRM team escalation channels: Enterprise → email only; Tribe/E-Deal/Web CRM → Teams channels in "FCCRM and Marketing Connect Connector" group
- Monday operational reviews recommended for Dead Letter Queue monitoring

---

## 5. Business Rules Reference

### Data Direction
1. Attributes: CRM → Apsis only (unidirectional)
2. Consent/subscription: Bidirectional (the only exception)
3. Outbound events (email sends, clicks, form submissions, MA flow triggers): Apsis → CRM only
4. Form submissions → profile attributes first, then outbound mappings determine CRM sync (direct form→CRM not supported)

### CRM as Master
1. CRM is authoritative for all contact data
2. Apsis enrichment attributes (not in CRM mapping) can be updated via forms and persist in Apsis only
3. External CRM values overwrite Apsis values on next sync for mapped attributes

### Sync Conditions
1. Evaluated client-side in Apsis (all records downloaded from CRM first, then filtered)
2. V1: AND logic only, equals operator only
3. V2 (backend complete, deployment pending CRM support): AND/OR, multiple operators (equals, not_equals, contains, starts_with, ends_with)
4. Fix for large customers (2M+): push conditions server-side to CRM (FSC Enterprise stateless API = pass in query params/body; Site Shop = stored config evaluated internally)

### Integration Limits
1. Only one native CRM integration per section (shared CRM ID field constraint; pre-2020 systems affected)
2. Third-party integrations without inbound sync (e.g., Playable) exempt from one-per-section rule

### Batch & Queue
1. SQS FIFO message group ID: `account_section + integration_id + crm_id` (prevents cross-customer blocking)
2. Batch size threshold: 200KB (SQS limit is 256KB)
3. Batch time threshold: 30 seconds
4. Consent updates bypass batching—sent directly

### Idempotency
1. Batch ID idempotency is external CRM's responsibility (they must handle duplicate batch deliveries)
2. Consent opt-in messages: one-shot only; never redrive from DLQ
3. Consent opt-out and batch activity: safe to redrive (idempotent on CRM side)

### Form Submissions
1. CRM sync checkbox must be enabled on form (hidden if integration not installed)
2. `can_sync_form_activities = false` on CRM → no form events sent regardless of form config
3. Minimum identifying info required: CRM ID OR (email AND/OR phone)—first name + last name alone insufficient
4. Form submission → CRM → CRM responds with `new_records`/`matched_records` → Merge Worker links form profile to CRM keyspace

### Profile Merges
1. Only allowed within same CRM installation's own entity types
2. Email keyspace ↔ CRM keyspace merges: never allowed
3. Must originate from CRM (not Apsis-initiated) to maintain CRM consistency

### Keyspaces
1. Never deleted on uninstallation
2. Keyspace mapping reverts to default on reinstallation
3. Entity-specific keyspaces (leads, silhouettes) separate from main contact keyspace

---

## 6. Known Issues & Workarounds

| Issue | Severity | Workaround | Component |
|-------|----------|-----------|-----------|
| Single CRM integration per section (shared CRM ID field) | High | Use separate sections for multiple CRM integrations; migration to per-integration key spaces unfunded | Field mapping, Audience schema |
| Sync conditions client-side for large customers (2M+ contacts) crashes full sync | High | Push sync conditions server-side to CRM. Immediate: reduce consent list mappings, optimize CRM pagination | Full Sync, Sync Conditions |
| In-memory audience export comparison cache crashes for 8M+ contacts | High | Remove optimization; stream directly to SQS (in progress) | Full Sync Consumer |
| FSC Enterprise consent pagination returns 504 timeout at 8M contacts | High | Request CRM-side sync condition filtering (proven with Dynamics/Sideshop) | Full Sync, FSC Enterprise connector |
| GitHub token in CodeBuild requires updating in TWO locations | High | After rotation: update Parameter Store AND manually reconnect source provider in CodeBuild settings | CI/CD pipeline |
| KMS single-region key blocks Generic Connector credential decryption in DR | High | Use legacy connectors for DR validation; migrate to multi-region KMS key (deprioritized) | DR process, KMS |
| FCC Enterprise returns HTTP 200 for all transport successes including business logic errors | High | Always parse response body for errors; never trust HTTP status alone | FCC Enterprise connector |
| Tribe lead entity configuration non-functional (always returns `contact`) | Medium | Pending Tribe confirmation to remove; do not remove without confirmation (risk of unknown entity types) | Tribe connector |
| Consent checkboxes not pre-filled in pre-filled forms | Medium | Users must manually recall consent state | Form Tool |
| Full sync report shows combined profile + consent message counts | Medium | Backend separated; frontend update pending developer availability (~1-2 days) | Full Sync Report UI |
| Tribe pagination: ~25% duplicate CRM IDs across page boundaries | Medium | Apsis deduplicates naturally by CRM ID; no functional impact | Tribe connector |
| Magento auto-mapping consent bug: both `update_email` and `update_sms` true causes `message_processor` line 426 failure | High | Remove AND NOT clause in conditional check; only check `update_email == true` | `message_processor` |
| Orphaned Consent 1.0 listeners in some accounts after Consent 2.0 migration | High | Remove via broker service API using stored delegation token | Consent management |
| Integration ID format must be exact for Clear Integration endpoint | Medium | Verify format in Integration Manager before request (e.g., `FSC Enterprise 12.0` not `FSC Enterprise`) | Clear Integration endpoint |
| Squid proxy allowlist changes require full Docker rebuild + deploy | Medium | `make build && make deploy`; cannot hot-update allowlist | Squid proxy |
| SQS FIFO 20,000-message scan limit: one customer can block others | High | Group message IDs by CRM ID; implement customer blacklist in `processMessage()` if needed | Outbound Worker, SQS |
| Dead Letter Queue: no upper retry limit (only 14-day SQS retention) | Medium | Monitor Monday operational review; manual intervention for chronic failures | Retry Driver Lambda |
| CRM field schema changes not auto-detected | Medium | Manual re-fetch/re-mapping required | Integration Manager |
| Deprecated integrations still visible in marketplace UI | Medium | Remove from backend integration list API AND frontend Angular components (separate tasks) | Integration marketplace |
| FSC Enterprise 12.1 UI does not show outbound mappings checkbox despite support | Medium | Contact dev team; feature works if configured directly in backend | FSC Enterprise 12.1 connector |
| Full sync concurrent map write race condition in Consumer (exit code 2) | High | Cancel and restart sync; root cause unknown | Full Sync Consumer |

---

## 7. Glossary

| Term | Definition |
|------|-----------|
| **Justin** | The integration middleware platform ("keeps everything in sync"—named after Justin Timberlake). Provides contract/interface connector libraries must fulfill. |
| **Holy Trinity** | Three parameters uniquely identifying an installation: Account ID (or name for pre-UID migration), Section ID, Integration ID |
| **Delta Sync** | Real-time incremental sync via webhooks when CRM contact data changes |
| **Full Sync** | Batch import of all existing CRM contacts; three states: Running → Finalizing → Successful |
| **Sluice Worker** | Gate between DSM and DSW; blocks real-time updates during active full sync to prevent historical data overwriting newer changes |
| **Generic Connector** | Standardized REST API contract CRM systems implement; single code path in Apsis for all conforming systems |
| **Legacy Connector** | Custom per-CRM adapter (FSC Enterprise 12.0, Dynamics legacy, Lime); no new features added |
| **Connector Library** | System-specific translation layer (Dynamics, FC Enterprise, Generic) adapting CRM API to Justin's format |
| **Broker Service** | Decrypts customer credentials (KMS for Generic Connector, plaintext for legacy) and injects into request headers |
| **Squid Proxy** | HTTP-layer egress filter; domain whitelisting to prevent unauthorized data exfiltration |
| **Keyspace** | Profile identifier namespace (e.g., CRM ID keyspace, email keyspace, silhouette keyspace per CRM installation) |
| **Keyspace Discriminator** | Format: `integrations:keyspaces:<8-char-hash>:<crm-logical-name>[:<entity-name>]` |
| **Silhouette** | Lead-like entity in E-Deal/FSC Corporate; represents initial form interest; stored in separate keyspace from contacts |
| **All Sub Queue (aud-sub-queue)** | SQS queue receiving all Audience events for outbound pipeline entry |
| **CRM ID** | External CRM's unique contact identifier; required on profile for outbound event sync |
| **Virtual Consent** | Legacy term: boolean fields on contact card representing consent state |
| **Native Consent** | Dedicated consent resource/entity in CRM (modern) |
| **Justin's Laws** | Mappings Manager rules preventing circular dependencies and invalid mapping configurations |
| **FQSOA** | Legacy shared queue system with contention issues; replaced by on-demand per-job Full Sync stack |
| **Profile List** | CRM list imported into Apsis as tags on existing profiles (does not create new profiles) |
| **Unified Data** | Cross-system complex query via HTTP/2 streaming; joins Audience data with CRM data at query time |
| **Data Provider Pattern** | CRM implements HTTP/2 endpoint returning paginated query results in Apache columnar format |
| **Sync Condition** | Filter rule for which contacts sync; currently AND logic + equals only (V1); V2 backend done |
| **Outbound Mapping** | Apsis attribute → CRM field name mapping for form submissions to CRM |
| **Integration ID (Logical Name)** | Internal identifier in API contracts/logs (e.g., `FSC Corporate`); stable, never changes |
| **Display Name** | Customer-visible connector name (e.g., `E-Deal`); easy to change; used in folder creation |
| **Dead Letter Queue (DLQ)** | SQS queue for failed outbound messages; redriven daily at 6:00 AM UTC |
| **Batch ID** | GUID per outbound batch; primary debugging tool to trace payload and outcome in CloudWatch |
| **Partner** | External org implementing Generic Connector (e.g., Sideshop for Dynamics, Intermail for loyalty) |
| **Supplier** | Legacy: external org hired to build custom integration (zero active contracts) |
| **Pre-filled Form** | Form link sent to specific profile auto-populated with that profile's data |
| **Congestion Pipeline** | Ingests React analytics events into Athena data warehouse |
| **Operations Bus** | Internal event bus that triggers account deletion cascades |
| **Section** | Namespace within Apsis account; one CRM integration per section |
| **Delegation Token** | Stored credential for automated async operations (delta sync, full sync, listener unregistration) |
| **total_is_known flag** | DB flag set by Full Sync Producer on completion; Consumer uses to detect end of data |
| **Auto-mapping** | Automatically generates virtual consent mappings dynamically (currently Magento only) |
| **ACU** | Aurora Capacity Unit; minimum 3 currently (over-provisioned; reduce to 1-2 with aggressive caching) |

---

## 8. Confidence Notes

### High Confidence
- Core pipeline architecture (inbound/outbound flows, SQS queues, Kafka topics, Broker + Squid pattern)
- Business rules (CRM as master, consent bidirectionality, batch thresholds, message group ID format)
- Security patterns (KMS for Generic Connector, HMAC-SHA256 webhooks, Clear Integration endpoint)
- Deployment process (CodeBuild, GitHub Actions hybrid, ARM64, CloudFormation)
- Known bugs (FCC 12.1 HTTP 200 issue, Content-Length header fix, Magento auto-mapping bug)

### Medium Confidence / Verify Before Relying On
- ⚠️ Exact file paths for many components are inferred from context (e.g., `apps/delta-sync-manager/`); verify actual names in codebase
- ⚠️ Database table names (e.g., `integration_mappings`, `allowed_domains`) are inferred; verify against actual schema
- ⚠️ Full Sync Consumer timeout is described as "~8 hours"—verify exact configured value
- ⚠️ Kafka partition per integration (one partition per integration ID) described as bottleneck—verify current actual configuration
- ⚠️ CloudWatch log group names: described by service name patterns but exact names should be confirmed
- ⚠️ Consent 2.0 listener cleanup for Magento is described as "at least 2 affected customers"—current status of remediation unclear

### Gaps / Unresolved
- ⚠️ Tribe confirmation (entity type always `contact`) still pending as of most recent session (3+ weeks overdue)
- ⚠️ Full sync `concurrent map write` race condition root cause unknown
- ⚠️ FSC Enterprise V2 sync condition timeline not confirmed
- ⚠️ Exact KMS encryption algorithm for keyspace discriminator hashing (SHA-1 or SHA-256)
- ⚠️ Whether FSC Enterprise producer stopping on consumer failure is documented
- ⚠️ GitHub Actions deploy workflow (`deploy.yaml`) inactive but exists—migration from CodeBuild to GitHub Actions planned but not yet executed
- ⚠️ Golfamore/Intermail keyspace remapping solution (Option 3 vs Option 2) decision pending meeting (Jan 22, 2026 mentioned in transcript)
- ⚠️ Tribe dynamic entity lead removal—blocked on Tribe confirmation
- ⚠️ Erik Andersson has left or is leaving the organization; knowledge transfer KPIs were being tracked

### Conflicts / Discrepancies
- **Retry window:** One session mentions "18 hours" total backoff; another describes the intent to reduce to "~5 hours" with refactored backoff. The 5-hour version is the **proposed** (not yet deployed) change.
- **Sluice Worker visibility timeout:** Described as "few minutes"—exact value not specified across sessions.
- **Full sync parallelism:** One session says "5 concurrent threads" default; another says this may be configurable. Treat as "approximately 5, may be configurable."