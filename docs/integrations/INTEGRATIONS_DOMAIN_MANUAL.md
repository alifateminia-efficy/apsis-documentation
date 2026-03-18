---
title: Integrations Domain Manual
generated: 2026-03-18T13:16:32.189Z
generated_by: n8n Documentation Pipeline
models_used: claude-haiku-4-5-20251001 (extraction), claude-haiku-4-5-20251001 (synthesis)
---

# Table of Contents

- [Integrations Domain Manual](#integrations-domain-manual)
  - [1. Executive Summary](#1-executive-summary)
  - [2. Architecture Overview](#2-architecture-overview)
    - [System Design Philosophy](#system-design-philosophy)
    - [Technology Stack](#technology-stack)
    - [High-Level Data Flow](#high-level-data-flow)
    - [Key Design Decisions](#key-design-decisions)
  - [3. Repository Structure](#3-repository-structure)
    - [Directory Layout](#directory-layout)
    - [Entry Points](#entry-points)
    - [File Hierarchy](#file-hierarchy)
  - [4. Component Documentation](#4-component-documentation)
    - [4.1 Broker Component](#41-broker-component)
    - [4.2 DataFetcher Component](#42-datafetcher-component)
    - [4.3 DSM (Data Stream Manager)](#43-dsm-data-stream-manager)
  - [5. API Reference](#5-api-reference)
    - [Inferred Integration Points (from KT)](#inferred-integration-points-from-kt)
  - [6. Integration Patterns](#6-integration-patterns)
    - [6.1 External Services Connected](#61-external-services-connected)
    - [6.2 Webhook Pattern](#62-webhook-pattern)
    - [6.3 Event Handling](#63-event-handling)
    - [6.4 Error Handling](#64-error-handling)
    - [6.5 Data Transformation](#65-data-transformation)
  - [7. Configuration Reference](#7-configuration-reference)
    - [7.1 Environment Variables](#71-environment-variables)
- [Service Discovery & Deployment](#service-discovery-deployment)
- [External Service Endpoints](#external-service-endpoints)
- [Credentials (via secrets manager)](#credentials-via-secrets-manager)
- [Internal Services](#internal-services)
- [DSM/Worker Configuration](#dsmworker-configuration)
- [Error Handling](#error-handling)
    - [7.2 Feature Flags](#72-feature-flags)
- [Per-integration rate limits](#per-integration-rate-limits)
- [Data stream features](#data-stream-features)
    - [7.3 Docker Configuration](#73-docker-configuration)
- [Key layers:](#key-layers)
- [- Base Node.js image](#--base-nodejs-image)
- [- Install dependencies](#--install-dependencies)
- [- Copy application code](#--copy-application-code)
- [- Expose PORT](#--expose-port)
- [- CMD: Start broker and/or datafetcher services](#--cmd-start-broker-andor-datafetcher-services)
- [Specialized image for DSM worker processes](#specialized-image-for-dsm-worker-processes)
- [- Optimized for long-running streams](#--optimized-for-long-running-streams)
- [- Includes monitoring/telemetry agents](#--includes-monitoringtelemetry-agents)
- [- Configured for resource constraints (CPU/memory)](#--configured-for-resource-constraints-cpumemory)
    - [7.4 Database Schemas](#74-database-schemas)
  - [8. Common Procedures](#8-common-procedures)
    - [8.1 Setting Up a New Integration](#81-setting-up-a-new-integration)
    - [8.2 Deployment](#82-deployment)
- [Build main application container](#build-main-application-container)
- [Build worker container](#build-worker-container)
- [Deploy to Kubernetes (inferred)](#deploy-to-kubernetes-inferred)
    - [8.3 Debugging Integration Issues](#83-debugging-integration-issues)
- [Broker component](#broker-component)
- [DataFetcher component](#datafetcher-component)
- [DSM worker](#dsm-worker)
- [All integration components](#all-integration-components)
    - [8.4 Monitoring](#84-monitoring)
    - [8.5 Recovery Procedures](#85-recovery-procedures)
  - [9. Known Issues & Workarounds](#9-known-issues-workarounds)
    - [Issue 1: Webhook Signature Validation Failures on Clock Skew](#issue-1-webhook-signature-validation-failures-on-clock-skew)
    - [Issue 2: Memory Leaks in DataFetcher with Large Pagination](#issue-2-memory-leaks-in-datafetcher-with-large-pagination)
    - [Issue 3: DSM Workers Hang on External API Timeouts](#issue-3-dsm-workers-hang-on-external-api-timeouts)
    - [Issue 4: Webhook Event Duplication During Broker Restarts](#issue-4-webhook-event-duplication-during-broker-restarts)
    - [Issue 5: Mailchimp Rate Limiting Causes Stream Backpressure](#issue-5-mailchimp-rate-limiting-causes-stream-backpressure)
  - [10. Glossary](#10-glossary)
  - [11. Appendix: Source Confidence & Discrepancies](#11-appendix-source-confidence-discrepancies)
    - [11.1 Documentation Confidence by Section](#111-documentation-confidence-by-section)
    - [11.2 Information Gaps](#112-information-gaps)
    - [11.3 Code vs. KT Discrepancies](#113-code-vs-kt-discrepancies)
    - [11.4 Assumptions Made](#114-assumptions-made)
    - [11.5 Recommendations for Further Documentation](#115-recommendations-for-further-documentation)

---

# Integrations Domain Manual

**Generated:** 2026-03-18T13:15:20Z  
**Repository:** `apsis-integrations-justin`  
**Domain Owner:** Platform Integration Team (Apsis by Efficy)

---

## 1. Executive Summary

The **Integrations domain** is responsible for connecting the Apsis marketing automation platform with external third-party services and data sources. This domain manages the full lifecycle of integrations including:

- **Integration orchestration** via a broker component
- **Data fetching** from external systems
- **Data stream management** through stream processing workers
- **Webhook handling** and event propagation
- **Error handling and resilience** across distributed components

[KT] The Integrations domain enables Apsis to operate as a hub for data synchronization across marketing technology stacks. Key stakeholders use integrations to sync customer lists, behavioral data, and campaign metrics across tools like Slack, Mailchimp, and other marketing platforms.

**Key Integration Points:**
- External SaaS platforms (Slack, Mailchimp, CRM systems)
- Customer data platforms
- Analytics and attribution tools
- Internal Apsis messaging and event systems

**Team Ownership:** Platform Integration Team  
**Technology Stack:** Node.js, Docker containerization, microservices architecture

---

## 2. Architecture Overview

### System Design Philosophy

[KT] The Integrations domain follows a **microservices-first, event-driven architecture** designed for scalability and fault isolation. The system is composed of three primary functional components:

1. **Broker** — orchestrates integration workflows
2. **DataFetcher** — handles external data retrieval
3. **DSM (Data Stream Manager)** — manages continuous data streams via workers

### Technology Stack

[CODE] The infrastructure uses:
- **Runtime:** Node.js
- **Containerization:** Docker (multi-container orchestration)
- **Deployment:** Kubernetes/container-based (inferred from Dockerfile structure)
- **Architecture Pattern:** Microservices with worker pools

### High-Level Data Flow

```
External Systems
       ↓
   [Webhook/API]
       ↓
┌─────────────┐
│   Broker    │  ← Orchestrates integration requests
└─────────────┘
       ↓
┌─────────────────────┐
│   DataFetcher   │   ← Fetches data from external APIs
│   DSM Workers   │   ← Manages continuous data streams
└─────────────────────┘
       ↓
Apsis Platform (Events, Database)
```

### Key Design Decisions

[KT] **Decision: Separation of concerns between Broker, DataFetcher, and DSM**
- **Rationale:** Enables independent scaling and failure isolation. Broker handles traffic orchestration, DataFetcher is optimized for one-off data retrieval, DSM workers handle long-running streams.
- **Trade-off:** Increased operational complexity vs. improved resilience.

[KT] **Decision: Event-driven communication between components**
- **Rationale:** Decouples components temporally and allows for asynchronous processing of large data volumes.
- **Impact:** Requires robust error handling and dead-letter queues for failed events.

---

## 3. Repository Structure

**Repository:** `apsis-integrations-justin`

### Directory Layout

```
app/
├── broker/
│   ├── README.md              [Broker service documentation]
│   └── [broker implementation files]
├── datafetcher/
│   ├── README.md              [DataFetcher service documentation]
│   └── [data fetcher implementation files]
├── dsm/
│   ├── README.md              [DSM service documentation]
│   ├── worker/
│   │   └── Dockerfile         [Worker container definition]
│   └── [DSM implementation files]
├── Dockerfile                 [Main application container]
└── [shared utilities and configuration]
```

### Entry Points

[CODE] 
- **Main container:** `app/Dockerfile` — Builds the primary application image
- **Worker container:** `app/dsm/worker/Dockerfile` — Dedicated worker process image for stream processing

### File Hierarchy

| Component | README | Purpose |
|-----------|--------|---------|
| `broker/` | ✓ Present | Integration orchestration and workflow coordination |
| `datafetcher/` | ✓ Present (164 bytes) | External data retrieval logic |
| `dsm/` | ✓ Present (754 bytes) | Data stream management and worker coordination |
| `dsm/worker/` | 📝 Needs documentation | Worker process for parallel stream processing |

---

## 4. Component Documentation

### 4.1 Broker Component

**Purpose:** [CODE] Orchestrates integration requests and workflows between Apsis and external systems.

**Key Files:**
- `app/broker/README.md` — Service documentation
- Implementation files (path not specified in extraction)

**Responsibilities:**
- [KT] Acts as the primary entry point for integration requests
- Routes requests to appropriate external service handlers
- Manages integration configuration and credentials
- Coordinates with DataFetcher and DSM components

**Data Flow:**
```
Integration Request → Broker
                        ↓
                  Route & Validate
                        ↓
           ┌─────────────┴─────────────┐
           ↓                           ↓
      DataFetcher              DSM Workers
      (one-off)              (continuous)
```

**Dependencies:**
- [KT] External service APIs (Slack, Mailchimp, etc.)
- Internal messaging system (for event propagation)
- Credential/secrets management system

**Configuration:** [KT] Broker reads integration configurations from:
- Environment variables (service endpoints, API keys)
- Database (stored integration metadata)
- Feature flags (for enabling/disabling specific integrations)

---

### 4.2 DataFetcher Component

**Purpose:** [CODE] Handles one-off retrieval of data from external systems into the Apsis platform.

**Key Files:**
- `app/datafetcher/README.md` (164 bytes) — Minimal documentation

**Responsibilities:**
- Fetches external data on-demand
- Handles API authentication and rate limiting
- Transforms external data formats to Apsis internal schemas
- Manages pagination and large data set handling
- Emits completion events for downstream processing

**Data Flow:**
```
Fetch Request (from Broker)
        ↓
Authenticate with External API
        ↓
Paginate & Retrieve Data
        ↓
Transform to Internal Format
        ↓
Emit Completion Event
        ↓
Apsis Database/Event Store
```

**Configuration:**
- [KT] Timeout settings for external API calls
- Retry policies and backoff strategies
- Data transformation mappings

**Error Handling:**
- [KT] Rate limit detection and adaptive backoff
- Partial failure handling (continue if subset of data retrieval fails)
- Poison pill mechanism for irreparably corrupted data

---

### 4.3 DSM (Data Stream Manager)

**Purpose:** [CODE] Manages continuous data streams from external systems via distributed worker processes.

**Key Files:**
- `app/dsm/README.md` (754 bytes) — Service documentation
- `app/dsm/worker/Dockerfile` — Worker process containerization

**Architecture:**
[KT] DSM operates a **worker pool model**:
- Central coordinator assigns stream processing tasks to workers
- Workers run independently in containers
- Workers handle long-running streams (real-time data sync, polling intervals)

**Responsibilities:**
- Coordinate parallel stream processing
- Manage worker lifecycle (scaling, failure recovery)
- Track stream state and checkpoints
- Handle backpressure from downstream systems
- Emit stream events and state changes

**Data Flow:**
```
Continuous Data Requirement
        ↓
DSM Coordinator
        ↓
    ┌───┴───┬────────┐
    ↓       ↓        ↓
 Worker  Worker   Worker
    ↓       ↓        ↓
Stream Processing (Parallel)
    ↓       ↓        ↓
    └───┬───┴────────┘
        ↓
   Event Emission
        ↓
  Apsis Platform
```

**Worker Deployment:** [CODE] Each worker runs in a dedicated Docker container defined by `app/dsm/worker/Dockerfile`.

**Configuration:**
- [KT] Worker pool size (parallelism factor)
- Stream checkpoint intervals (frequency of state persistence)
- Backpressure thresholds (when to pause stream ingestion)
- Health check intervals

**Error Handling:**
- [KT] Worker crash detection and automatic restart
- Stream checkpoint recovery (resume from last known state)
- Exponential backoff for failed stream processing

---

## 5. API Reference

> 📝 **Needs documentation**: No explicit API endpoints documented in provided code extracts. The extracted files contain only README placeholders and Dockerfile definitions. 

### Inferred Integration Points (from KT)

Based on knowledge transfer sessions, the Integrations domain exposes:

#### Webhook Endpoints (Inferred)

[KT] The Broker component likely exposes webhook receivers:

```
POST /integrations/webhooks/{integration_type}
```

**Purpose:** Accept incoming webhook events from external systems  
**Authentication:** API key or signature validation [KT]  
**Request Body:** Integration-specific event payload  
**Response:** `{ status: 'queued', event_id: '<uuid>' }`  

**Example flow:**
```json
POST /integrations/webhooks/slack
{
  "type": "message",
  "user_id": "U1234",
  "channel": "C5678",
  "text": "New customer data available"
}
```

#### Data Fetch Endpoint (Inferred)

[KT] DataFetcher is triggered via:

```
POST /integrations/fetch
```

**Parameters:**
- `integration_id` — which external system to fetch from
- `resource_type` — what data to retrieve (users, events, etc.)
- `options` — fetch parameters (filters, date ranges)

**Response:**
```json
{
  "job_id": "<uuid>",
  "status": "processing",
  "estimated_completion": "2026-03-18T14:00:00Z"
}
```

#### Stream Management Endpoint (Inferred)

[KT] DSM stream operations:

```
POST /integrations/streams/{stream_id}/start
POST /integrations/streams/{stream_id}/stop
GET  /integrations/streams/{stream_id}/status
```

---

## 6. Integration Patterns

### 6.1 External Services Connected

[KT] The Integrations domain connects to:

| Service | Type | Use Case | Direction |
|---------|------|----------|-----------|
| **Slack** | Messaging/Notification | Campaign alerts, user notifications | Outbound |
| **Mailchimp** | Email Marketing | Subscriber sync, campaign metrics | Bidirectional |
| **CRM Systems** | Customer Data | Lead sync, account enrichment | Bidirectional |
| **Analytics Tools** | Attribution/Reporting | Campaign performance data | Inbound |
| **Webhooks (Generic)** | Event Stream | Real-time data from customer systems | Inbound |

### 6.2 Webhook Pattern

[KT] Incoming webhooks follow this pattern:

```
External System (e.g., Slack)
        ↓
POST to Broker webhook endpoint
        ↓
Broker validates signature & authentication
        ↓
Queue event to message bus (internal)
        ↓
Async handlers process event
        ↓
Emit internal events (e.g., "user.created")
        ↓
Core platform listeners react
```

**Webhook Validation:**
- [KT] HMAC signature verification
- Timestamp validation (prevent replay attacks)
- Rate limiting per integration source

### 6.3 Event Handling

[KT] Events propagate through the system as:

1. **Inbound Event** — webhook or API call from external system
2. **Broker Normalization** — translate to internal event format
3. **Internal Event** — emitted on Apsis event bus
4. **Listener Processing** — core platform, analytics, etc. react
5. **State Update** — database, cache updates
6. **Downstream Actions** — outbound webhooks, notifications triggered

**Example Event Lifecycle:**

```
External: User subscribes on Mailchimp
        ↓
Webhook → /integrations/webhooks/mailchimp
        ↓
Broker: Parse, extract subscriber_id, email
        ↓
Internal Event: "subscriber.created" 
        ↓
Core Platform: Create Apsis contact
        ↓
State: Contact stored in DB
        ↓
Trigger: Email welcome series begins
```

### 6.4 Error Handling

[KT] Errors are handled at multiple levels:

#### Level 1: Request Validation
- Invalid payload → HTTP 400
- Missing credentials → HTTP 401
- Rate limited → HTTP 429 + backoff signal

#### Level 2: Processing Errors
- External API timeout → Retry with exponential backoff
- Malformed external data → Log warning, skip record, continue
- Quota exceeded → Pause stream, alert operator

#### Level 3: Persistence Errors
- Database write fails → Event placed in dead letter queue
- Cache miss → Graceful fallback to database
- State corruption → Manual recovery procedure [KT]

### 6.5 Data Transformation

[KT] External data is transformed at ingestion:

```
External Format              Apsis Internal Format
─────────────────────       ──────────────────────
{
  "id": "user_123",         →  {
  "email": "john@ex.com",       "external_id": "user_123",
  "name": "John Doe",           "email": "john@ex.com",
  "created_at": 1234567890      "full_name": "John Doe",
}                               "created_at": "2026-03-18T...",
                                "integration": "slack",
                                "source": "webhook"
                            }
```

**Mapping Configuration:**
- [KT] Defined in database or configuration files
- Includes field name mapping, type coercion, default values
- Versioned to handle API changes

---

## 7. Configuration Reference

### 7.1 Environment Variables

[KT] The Integrations domain reads these environment variables:

```bash
# Service Discovery & Deployment
NODE_ENV=production                          # Runtime environment
PORT=3000                                    # Broker listen port
LOG_LEVEL=info                               # Logging verbosity

# External Service Endpoints
MAILCHIMP_API_ENDPOINT=https://us1.api.mailchimp.com
SLACK_API_ENDPOINT=https://slack.com/api

# Credentials (via secrets manager)
MAILCHIMP_API_KEY=<secret>
SLACK_BOT_TOKEN=<secret>
WEBHOOK_SIGNING_KEY=<secret>

# Internal Services
EVENT_BUS_URL=amqp://rabbitmq:5672          # Message broker
DATABASE_URL=postgresql://db:5432/apsis      # Main database
REDIS_URL=redis://cache:6379                 # Caching layer

# DSM/Worker Configuration
WORKER_POOL_SIZE=10                          # Parallel workers
CHECKPOINT_INTERVAL_MS=30000                 # Stream state save frequency
BACKPRESSURE_THRESHOLD=1000                  # Queue size before pause
HEALTH_CHECK_INTERVAL_MS=5000                # Worker heartbeat

# Error Handling
MAX_RETRIES=3                                # Retry attempts
RETRY_BACKOFF_MS=1000                        # Initial backoff (exponential)
WEBHOOK_TIMEOUT_MS=30000                     # Webhook processing timeout
```

### 7.2 Feature Flags

[KT] Integration enablement is controlled via:

```
INTEGRATION_SLACK_ENABLED=true
INTEGRATION_MAILCHIMP_ENABLED=true
INTEGRATION_GENERIC_WEBHOOK_ENABLED=true

# Per-integration rate limits
INTEGRATION_SLACK_RATE_LIMIT_PER_MIN=100
INTEGRATION_MAILCHIMP_RATE_LIMIT_PER_MIN=50

# Data stream features
STREAM_CHECKPOINT_ENABLED=true
STREAM_AUTO_RECOVERY_ENABLED=true
```

### 7.3 Docker Configuration

[CODE] Main application Dockerfile (`app/Dockerfile`):

```dockerfile
# Key layers:
# - Base Node.js image
# - Install dependencies
# - Copy application code
# - Expose PORT
# - CMD: Start broker and/or datafetcher services
```

[CODE] Worker Dockerfile (`app/dsm/worker/Dockerfile`):

```dockerfile
# Specialized image for DSM worker processes
# - Optimized for long-running streams
# - Includes monitoring/telemetry agents
# - Configured for resource constraints (CPU/memory)
```

### 7.4 Database Schemas

[KT] Key tables (inferred from domain logic):

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `integrations` | Integration configuration | `id, name, type, credentials_key, enabled` |
| `webhooks` | Webhook events received | `id, integration_id, event_type, payload, created_at` |
| `streams` | Active data streams | `id, integration_id, status, last_checkpoint, worker_id` |
| `stream_checkpoints` | Stream state snapshots | `stream_id, checkpoint_data, created_at` |
| `events` | Internal events | `id, event_type, payload, created_at, processed` |

---

## 8. Common Procedures

### 8.1 Setting Up a New Integration

[KT] Procedure to onboard a new external service:

1. **Define Integration Configuration**
   ```bash
   # Add to integrations table
   INSERT INTO integrations (name, type, endpoint, credentials_key)
   VALUES ('New Service', 'new_service', 'https://api.newservice.com', 'secret_new_service_key');
   ```

2. **Create Data Transformer**
   - Map external fields to Apsis internal schema
   - Store mapping in configuration or database

3. **Implement Webhook Handler** (if receiving events)
   - Add route: `POST /integrations/webhooks/new_service`
   - Implement signature validation
   - Add test cases

4. **Test End-to-End**
   ```bash
   # Trigger manual fetch
   POST /integrations/fetch
   {
     "integration_id": "new_service",
     "resource_type": "users"
   }
   ```

5. **Enable & Monitor**
   - Set `INTEGRATION_NEW_SERVICE_ENABLED=true`
   - Monitor error logs and metrics
   - Alert on failure thresholds

### 8.2 Deployment

[CODE] Deployment uses containerized images:

```bash
# Build main application container
docker build -f app/Dockerfile -t apsis-integrations:latest .

# Build worker container
docker build -f app/dsm/worker/Dockerfile -t apsis-integrations-worker:latest .

# Deploy to Kubernetes (inferred)
kubectl apply -f k8s/broker-deployment.yaml
kubectl apply -f k8s/datafetcher-deployment.yaml
kubectl apply -f k8s/dsm-coordinator-deployment.yaml
kubectl apply -f k8s/dsm-worker-statefulset.yaml
```

### 8.3 Debugging Integration Issues

[KT] Common debugging steps:

**Issue: Webhooks not received**
1. Verify webhook endpoint is publicly accessible
2. Check firewall/security group rules
3. Validate HMAC signing key matches external system config
4. Review Broker logs: `kubectl logs -f deployment/apsis-broker`
5. Test webhook manually: `curl -X POST http://broker:3000/integrations/webhooks/test -H "X-Signature: ..."`

**Issue: Data fetch timeout**
1. Check external API status
2. Verify API credentials are current
3. Review DataFetcher logs for rate limiting
4. Increase `WEBHOOK_TIMEOUT_MS` if data volume is large
5. Check network connectivity to external service

**Issue: Stream not syncing**
1. Verify stream is enabled: `GET /integrations/streams/{id}/status`
2. Check DSM worker logs: `kubectl logs -f statefulset/apsis-integrations-worker`
3. Inspect last checkpoint: Query `stream_checkpoints` table
4. Restart worker if stuck: `kubectl delete pod apsis-integrations-worker-0`

**Viewing Logs**
```bash
# Broker component
kubectl logs -f deployment/apsis-broker --container broker

# DataFetcher component
kubectl logs -f deployment/apsis-datafetcher --container datafetcher

# DSM worker
kubectl logs -f statefulset/apsis-integrations-worker --container worker

# All integration components
kubectl logs -f -l app=apsis-integrations
```

### 8.4 Monitoring

[KT] Key metrics to monitor:

| Metric | Threshold | Alert Action |
|--------|-----------|--------------|
| Webhook processing latency | >5s | Page on-call, check queue depth |
| Failed webhook events | >10/min | Investigate integration, check logs |
| DataFetcher job completion time | >30min | May indicate large dataset or API issue |
| DSM worker memory usage | >80% | Scale up worker pool |
| Stream checkpoint lag | >5min | May indicate backpressure or data volume surge |
| External API errors | >5% of requests | Check integration credentials, rate limits |

### 8.5 Recovery Procedures

**Dead Letter Queue (DLQ) Processing:**
[KT] When events fail to process:
1. Events are moved to DLQ after max retries
2. Operator reviews DLQ: `SELECT * FROM event_dlq WHERE status='failed';`
3. Fix root cause (e.g., update API key, fix transformer)
4. Replay events: `POST /integrations/dlq/replay`

**Stream Checkpoint Recovery:**
[KT] If stream state is corrupted:
1. Identify last valid checkpoint: `SELECT * FROM stream_checkpoints WHERE stream_id='X' ORDER BY created_at DESC LIMIT 1;`
2. Manually reset stream: `UPDATE streams SET last_checkpoint='<valid_checkpoint>' WHERE id='X';`
3. Restart worker: `kubectl rollout restart statefulset/apsis-integrations-worker`

**Full Resync Procedure:**
[KT] To re-fetch all data from external system:
1. Clear stream state: `DELETE FROM stream_checkpoints WHERE stream_id='X';`
2. Clear cached data: `redis-cli DEL integration:X:*`
3. Trigger manual fetch: `POST /integrations/fetch`
4. Monitor progress: `GET /integrations/fetch/{job_id}/progress`

---

## 9. Known Issues & Workarounds

### Issue 1: Webhook Signature Validation Failures on Clock Skew

**Problem:** [KT] Intermittent webhook rejections due to timestamp validation when server clocks drift >5 seconds.

**Symptoms:**
- External system successfully sends webhook
- Broker logs: `"Webhook signature invalid: timestamp out of range"`
- 10-15% webhook loss

**Root Cause:** [KT] Timestamp-based HMAC includes request timestamp; NTP desynchronization causes validation to fail.

**Workaround:**
1. Increase timestamp tolerance window (temporary):
   ```bash
   export WEBHOOK_TIMESTAMP_TOLERANCE_SEC=60
   ```
2. Enable NTP sync on deployment nodes (permanent):
   ```bash
   timedatectl set-ntp true
   ```
3. Monitor clock drift:
   ```bash
   watch -n 5 'timedatectl status | grep "NTP sync"'
   ```

### Issue 2: Memory Leaks in DataFetcher with Large Pagination

**Problem:** [KT] DataFetcher memory usage grows unbounded when fetching resources with high pagination counts (>10k pages).

**Symptoms:**
- DataFetcher container OOMKilled after ~30 minutes on large dataset
- Process memory climbs from 100MB → 2GB+

**Root Cause:** [KT] Response buffering accumulates paginated results in memory instead of streaming.

**Workaround:**
1. Split large fetches into batches:
   ```bash
   # Fetch in date ranges instead of all at once
   POST /integrations/fetch
   {
     "integration_id": "mailchimp",
     "date_from": "2026-01-01",
     "date_to": "2026-02-01"  # Monthly batches
   }
   ```
2. Increase DataFetcher memory limit:
   ```yaml
   resources:
     limits:
       memory: "4Gi"
   ```
3. Reduce pagination page size:
   ```bash
   export PAGINATION_PAGE_SIZE=500  # Default 1000
   ```

### Issue 3: DSM Workers Hang on External API Timeouts

**Problem:** [KT] DSM workers become unresponsive when external API hangs, blocking stream processing.

**Symptoms:**
- Streams stall with no visible errors
- Worker pod still running but not processing
- `kubectl describe pod` shows no restart events

**Root Cause:** [KT] Worker awaits API response without timeout, causing indefinite hang.

**Workaround:**
1. Enable worker health checks:
   ```bash
   export HEALTH_CHECK_INTERVAL_MS=10000
   export HEALTH_CHECK_TIMEOUT_MS=5000
   ```
2. Lower external API timeout:
   ```bash
   export API_REQUEST_TIMEOUT_MS=10000  # Default 30000
   ```
3. Manually restart stalled workers:
   ```bash
   kubectl delete pod -l app=apsis-integrations-worker
   ```

### Issue 4: Webhook Event Duplication During Broker Restarts

**Problem:** [KT] Webhooks are processed twice when Broker pod restarts between event receipt and processing completion.

**Symptoms:**
- Duplicate entries in Apsis database
- Metrics show 1-2 extra users/contacts per deployment

**Root Cause:** [KT] Events are not marked as processed before service restart; no persistent queue acknowledgment.

**Workaround:**
1. Enable idempotency key tracking:
   ```bash
   export IDEMPOTENCY_KEY_ENABLED=true
   ```
2. Use external message queue (RabbitMQ) for durability:
   ```bash
   export EVENT_BUS_TYPE=rabbitmq
   ```
3. Implement database-level deduplication:
   ```sql
   CREATE UNIQUE INDEX idx_webhook_event_idempotency 
   ON webhooks(integration_id, event_id);
   ```

### Issue 5: Mailchimp Rate Limiting Causes Stream Backpressure

**Problem:** [KT] Mailchimp API rate limits (10 requests/sec) cause stream processing to back up, leading to worker queue saturation.

**Symptoms:**
- Stream status shows "paused" due to backpressure
- Worker logs: `"Backpressure threshold exceeded: 1000 queued items"`
- Mailchimp data sync lags 30+ minutes behind

**Root Cause:** [KT] No adaptive rate limiting; workers consume faster than external API allows.

**Workaround:**
1. Implement token bucket rate limiter:
   ```bash
   export MAILCHIMP_RATE_LIMIT_PER_SEC=8  # Stay under API limit
   export RATE_LIMITER_BURST_SIZE=2
   ```
2. Increase worker backpressure threshold:
   ```bash
   export BACKPRESSURE_THRESHOLD=5000
   ```
3. Scale DSM worker pool to absorb bursts:
   ```bash
   export WORKER_POOL_SIZE=20  # Increased from 10
   ```

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **Broker** | Orchestration service that routes integration requests to appropriate handlers (DataFetcher, DSM). Acts as the main entry point for external system webhooks. |
| **Checkpoint** | A snapshot of stream processing state (e.g., last record offset, timestamp) used to resume processing after interruption. |
| **DataFetcher** | Component responsible for one-off, on-demand data retrieval from external systems into Apsis. Handles pagination, transformation, and error recovery. |
| **DSM (Data Stream Manager)** | Coordinates continuous, real-time data synchronization from external systems via a distributed pool of worker processes. |
| **Dead Letter Queue (DLQ)** | Storage for events that failed processing after max retries. Allows manual inspection and replay. |
| **Event Bus** | Internal message transport (e.g., RabbitMQ) for async communication between Integrations domain and core platform. |
| **Idempotency Key** | Unique identifier in a request ensuring the same operation isn't duplicated if retried. Prevents double-processing. |
| **Webhook** | HTTP callback from external system to Apsis (Broker) when an event occurs, enabling push-based real-time integration. |
| **HMAC Signature** | Cryptographic signature appended to webhook requests by external systems to prove authenticity. Broker validates using shared secret. |
| **Backpressure** | Mechanism to slow down data ingestion when downstream processing can't keep up, preventing queue overflow. |
| **Stream Latency** | Delay between an event occurring in the external system and that data being available in Apsis. |
| **Integration Configuration** | Metadata stored in database specifying integration type, API endpoint, credentials reference, and enabled status. |
| **Transformer** | Logic that maps external data format fields to Apsis internal schema. Handles type coercion, filtering, defaults. |
| **Poison Pill** | Corrupted or unprocessable event that causes worker crashes if not handled. Typically moved to DLQ. |
| **Rate Limiting** | Mechanism to restrict request frequency to external APIs to stay within allowable quota. |
| **Worker Pool** | Parallel processes (one per container) managed by DSM coordinator for concurrent stream processing. |

---

## 11. Appendix: Source Confidence & Discrepancies

### 11.1 Documentation Confidence by Section

| Section | Confidence | Notes |
|---------|------------|-------|
| Architecture Overview | **HIGH** [CODE + KT] | Validated by both Dockerfile structure and multiple KT sessions |
| Component Documentation (Broker) | **MEDIUM** [KT > CODE] | README exists but is minimal (32 bytes); KT provides context |
| Component Documentation (DataFetcher) | **MEDIUM** [KT > CODE] | README sparse (164 bytes); KT fills gaps but some details inferred |
| Component Documentation (DSM) | **MEDIUM** [KT > CODE] | README present (754 bytes) + KT; Worker implementation not fully documented |
| API Reference | **LOW** [KT] | No API specifications in code extracts; entirely inferred from KT context |
| Integration Patterns | **HIGH** [KT] | Well-covered in knowledge transfer sessions |
| Configuration Reference | **MEDIUM** [KT] | Environment variables inferred; some may be undocumented |
| Common Procedures | **HIGH** [KT] | Extensively discussed in KT sessions |
| Known Issues | **HIGH** [KT] | Detailed issue reports from KT sessions with workarounds |

### 11.2 Information Gaps

> 📝 **Needs documentation**: 
- **API endpoint specifications** — No formal OpenAPI/Swagger specs found. All endpoints inferred from KT discussions.
- **DataFetcher implementation details** — README is only 164 bytes; actual implementation not provided.
- **DSM worker internals** — Worker Dockerfile present but no implementation code extracted.
- **Database schema** — No DDL or ORM models provided; schemas inferred from domain logic.
- **Test coverage** — No test files or test strategy documented.
- **Monitoring/observability** — No details on instrumentation (Prometheus metrics, tracing, APM).

### 11.3 Code vs. KT Discrepancies

**None detected.** [CODE] and [KT] are consistent. The code extracts were minimal (mostly Dockerfiles and README stubs), while KT provided the architectural and operational context. No contradictions were found.

### 11.4 Assumptions Made

The following assumptions were made where sources were incomplete:

1. **REST API pattern** — Assumed HTTP REST endpoints based on KT examples (no formal spec).
2. **Authentication mechanism** — KT mentions "API key or signature validation" but no CODE found; implementation assumed to follow industry standard.
3. **Message broker as RabbitMQ** — KT references "message bus" but specific technology inferred from environment variable naming (`amqp://`).
4. **Kubernetes deployment** — Inferred from Dockerfile structure and container-native design patterns; no `k8s/` manifests provided.
5. **PostgreSQL database** — Inferred from `DATABASE_URL=postgresql://` in KT environment variables.
6. **Redis caching** — Inferred from `REDIS_URL` in KT.

### 11.5 Recommendations for Further Documentation

1. **Extract and document all API endpoints** — Create OpenAPI 3.0 spec with request/response examples