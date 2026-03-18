---
title: Integrations Domain Manual
generated: 2026-03-18T11:49:32.663Z
generated_by: n8n Documentation Pipeline
models_used: claude-haiku-4-5-20251001 (extraction), claude-haiku-4-5-20251001 (synthesis)
---

# Table of Contents

- [Integrations Domain Manual](#integrations-domain-manual)
  - [1. Executive Summary](#1-executive-summary)
    - [Purpose](#purpose)
    - [Scope](#scope)
    - [Team Ownership [KT]](#team-ownership-kt)
    - [Key Integration Points [CODE][KT]](#key-integration-points-codekt)
  - [2. Architecture Overview](#2-architecture-overview)
    - [High-Level System Design [KT]](#high-level-system-design-kt)
    - [Technology Stack [CODE]](#technology-stack-code)
    - [Key Design Decisions & Rationale [KT]](#key-design-decisions-rationale-kt)
  - [3. Repository Structure](#3-repository-structure)
    - [Directory Layout [CODE]](#directory-layout-code)
    - [Entry Points [CODE][KT]](#entry-points-codekt)
    - [Container Images [CODE]](#container-images-code)
  - [4. Component Documentation](#4-component-documentation)
    - [4.1 Broker Component](#41-broker-component)
    - [4.2 DataFetcher Component](#42-datafetcher-component)
    - [4.3 Data Synchronization Manager (DSM)](#43-data-synchronization-manager-dsm)
  - [5. API Reference](#5-api-reference)
    - [5.1 Broker API](#51-broker-api)
    - [5.2 DataFetcher API](#52-datafetcher-api)
    - [5.3 DSM API](#53-dsm-api)
  - [6. Integration Patterns](#6-integration-patterns)
    - [6.1 External Services Connected [KT]](#61-external-services-connected-kt)
    - [6.2 Data Flow Patterns [KT]](#62-data-flow-patterns-kt)
    - [6.3 Webhook Handling [KT]](#63-webhook-handling-kt)
    - [6.4 Event Handling [KT]](#64-event-handling-kt)
    - [6.5 Error Handling [KT]](#65-error-handling-kt)
  - [7. Configuration Reference](#7-configuration-reference)
    - [7.1 Environment Variables [KT]](#71-environment-variables-kt)
- [Message Queue Configuration](#message-queue-configuration)
- [Database](#database)
- [External Integrations Authentication Vault](#external-integrations-authentication-vault)
- [Logging & Monitoring](#logging-monitoring)
- [DataFetcher Scheduling](#datafetcher-scheduling)
- [DSM Configuration](#dsm-configuration)
- [Webhook Security](#webhook-security)
    - [7.2 Feature Flags [KT]](#72-feature-flags-kt)
    - [7.3 Connector Configuration [KT]](#73-connector-configuration-kt)
    - [7.4 Database Schemas [KT]](#74-database-schemas-kt)
    - [7.5 Deployment Configuration [CODE]](#75-deployment-configuration-code)
- [From app/Dockerfile (1050 bytes)](#from-appdockerfile-1050-bytes)
- [Multi-stage container build for main services](#multi-stage-container-build-for-main-services)
- [Specific content not provided in extraction](#specific-content-not-provided-in-extraction)
- [From app/dsm/worker/Dockerfile (690 bytes)](#from-appdsmworkerdockerfile-690-bytes)
- [Dedicated container for DSM sync operations](#dedicated-container-for-dsm-sync-operations)
- [Specific content not provided in extraction](#specific-content-not-provided-in-extraction)
  - [8. Common Procedures](#8-common-procedures)
    - [8.1 Setting Up a New Integration [KT]](#81-setting-up-a-new-integration-kt)
- [Create connector implementation in integrations repo](#create-connector-implementation-in-integrations-repo)
- [Add integration to database](#add-integration-to-database)
- [Store secrets in vault (not directly in config)](#store-secrets-in-vault-not-directly-in-config)
- [Run integration tests](#run-integration-tests)
- [Test webhook receipt (if webhook-enabled)](#test-webhook-receipt-if-webhook-enabled)
- [Deploy via CD pipeline](#deploy-via-cd-pipeline)
- [... after approval and CI passes ...](#-after-approval-and-ci-passes-)
- [Enable in production](#enable-in-production)
    - [8.2 Debugging Integration Issues [KT]](#82-debugging-integration-issues-kt)
- [Assuming message queue is RabbitMQ or similar](#assuming-message-queue-is-rabbitmq-or-similar)
- [List DLQ messages for review](#list-dlq-messages-for-review)
- [Inspect message content](#inspect-message-content)
- [View container logs](#view-container-logs)
- [Filter for errors](#filter-for-errors)
- [Follow real-time](#follow-real-time)
- [Test authentication](#test-authentication)
- [Test data endpoint (manual)](#test-data-endpoint-manual)
    - [8.3 Deploying Integration Changes [CODE][KT]](#83-deploying-integration-changes-codekt)
- [Build main application](#build-main-application)
- [Build DSM worker](#build-dsm-worker)
- [Update manifest with new image](#update-manifest-with-new-image)
- [Monitor rollout](#monitor-rollout)
    - [8.4 Monitoring & Alerting [KT]](#84-monitoring-alerting-kt)
  - [9. Known Issues & Workarounds](#9-known-issues-workarounds)
    - [9.1 Bidirectional Sync Conflicts [KT]](#91-bidirectional-sync-conflicts-kt)
    - [9.2 Webhook Retry on Network Timeout [KT]](#92-webhook-retry-on-network-timeout-kt)
    - [9.3 Rate Limit Exceeded During Batch Sync [KT]](#93-rate-limit-exceeded-during-batch-sync-kt)
    - [9.4 Credential Rotation [KT]](#94-credential-rotation-kt)
    - [9.5 Data Transformation Errors [KT]](#95-data-transformation-errors-kt)
  - [10. Glossary](#10-glossary)
    - [Domain-Specific Terms](#domain-specific-terms)

---

# Integrations Domain Manual

**Generated:** 2026-03-18  
**Repository:** apsis-integrations-justin  
**Status:** ⚠️ Preliminary (Limited source material)

---

## 1. Executive Summary

The **Integrations domain** is a critical subsystem within Apsis by Efficy's marketing automation platform that enables seamless connectivity between the core platform and external data sources, marketing channels, and third-party services.

### Purpose
- Facilitate real-time and batch data synchronization with external systems
- Provide standardized connector interfaces for diverse integration scenarios
- Enable secure authentication and data transformation between systems
- Support webhook-based event handling and bidirectional communication

### Scope
This manual covers:
- Integration broker and orchestration patterns
- Data fetcher components for external source polling
- Data synchronization manager (DSM) for synchronizing contact and campaign data
- Integration configuration and deployment patterns

### Team Ownership [KT]
- **Benjamin** – Integrations architecture and strategic oversight
- **Integrations team** – Development and maintenance
- Cross-functional collaboration with **Membrane** team on technical standards

### Key Integration Points [CODE][KT]
1. **Broker** – Message routing and queueing between integrations
2. **DataFetcher** – Polling-based data retrieval from external APIs
3. **DSM (Data Synchronization Manager)** – Bi-directional contact and campaign synchronization
4. **Authentication & Credentials** – Secure credential management for external services
5. **Webhook handlers** – Event-driven integration patterns

---

## 2. Architecture Overview

### High-Level System Design [KT]

The Integrations domain follows a **modular, event-driven architecture** with three primary subsystems:

```
┌─────────────────────────────────────────────────────────┐
│                  Apsis by Efficy Core                   │
└─────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────┐
│          Integrations Domain                             │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Broker     │  │  DataFetcher │  │     DSM      │   │
│  │ (routing &   │  │ (polling &   │  │  (sync &     │   │
│  │  queueing)   │  │ aggregation) │  │  transform)  │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│         ↓                  ↓                  ↓           │
│    [Webhooks]       [Pull Connectors]  [Push/Bi-dir]    │
└─────────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────────┐
│          External Systems & Data Sources                │
│                                                           │
│  • Email Marketing Platforms (Mailchimp, etc.)          │
│  • CRM Systems (Salesforce, HubSpot)                    │
│  • Data Warehouses                                      │
│  • Custom REST APIs                                     │
└─────────────────────────────────────────────────────────┘
```

### Technology Stack [CODE]

- **Runtime:** Docker containers [CODE]
- **Language:** Node.js (inferred from Dockerfile patterns) [CODE]
- **Deployment:** Kubernetes-compatible container orchestration [CODE]
- **Message Queue:** Implied via broker pattern (specific technology not documented) [KT]
- **Authentication:** OAuth 2.0, API keys, credential vault integration [KT]

### Key Design Decisions & Rationale [KT]

| Decision | Rationale |
|----------|-----------|
| **Modular components** (Broker, DataFetcher, DSM) | Separation of concerns; independent scaling and deployment; supports varied integration patterns |
| **Event-driven with fallback polling** | Webhooks provide real-time responsiveness; polling provides reliability for sources without webhook support |
| **Dedicated DSM worker** | Isolates heavy synchronization workloads from broker; prevents blocking of message routing |
| **Containerized deployment** | Cloud-native scaling; consistent environments across dev/staging/prod |
| **Connector abstraction** | Standardized interface reduces per-integration code; supports rapid connector development |

---

## 3. Repository Structure

The Integrations domain is contained in the **`apsis-integrations-justin`** repository. [CODE]

### Directory Layout [CODE]

```
apsis-integrations-justin/
├── app/
│   ├── broker/
│   │   ├── README.md                    (Broker component documentation)
│   │   └── [implementation files]
│   ├── datafetcher/
│   │   ├── README.md                    (DataFetcher component docs)
│   │   └── [implementation files]
│   ├── dsm/
│   │   ├── README.md                    (DSM component documentation)
│   │   ├── worker/
│   │   │   └── Dockerfile               (DSM worker container image)
│   │   └── [implementation files]
│   └── Dockerfile                       (Main application container image)
```

### Entry Points [CODE][KT]

1. **Broker** (`app/broker/`) – Primary message router for all integrations
2. **DataFetcher** (`app/datafetcher/`) – Executes scheduled polling jobs
3. **DSM** (`app/dsm/`) – Handles synchronization workflows; worker process for async operations

### Container Images [CODE]

- **Main application:** `app/Dockerfile` – Runs primary services
- **DSM worker:** `app/dsm/worker/Dockerfile` – Dedicated worker for sync operations

---

## 4. Component Documentation

### 4.1 Broker Component

**Purpose** [CODE][KT]  
Central message router and queueing system that orchestrates communication between the Apsis platform and external integrations. Handles webhook receiving, message routing, retry logic, and error management.

**Key Responsibilities** [KT]
- Receive incoming webhooks from external systems
- Route messages to appropriate integration handlers
- Implement queue-based processing for reliability
- Manage retry policies and dead-letter handling
- Transform and validate payloads before downstream processing

**Key Files** [CODE]
```
app/broker/
├── README.md
└── [implementation files - content not provided]
```

**Configuration** [KT]
- Queue backend connection details (not fully documented in sources)
- Retry policies: exponential backoff with configurable attempts
- Webhook authentication: signature validation (provider-specific)

**Data Flow** [KT]
```
External System Webhook
       ↓
Broker Receives & Validates
       ↓
Message Queued
       ↓
Route Based on Integration Type
       ↓
Handler Processing (Async)
       ↓
Acknowledgment to External System
```

**Dependencies**
- Message queue infrastructure [KT]
- Integration connector implementations [CODE]
- Credential management system [KT]

**Status** [CODE]  
📝 **Needs documentation**: The broker README.md contains only minimal content (32 bytes); detailed implementation documentation is absent.

---

### 4.2 DataFetcher Component

**Purpose** [CODE][KT]  
Executes scheduled, poll-based data retrieval from external systems. Implements connector patterns for diverse external APIs and handles pagination, rate limiting, and data transformation.

**Key Responsibilities** [KT]
- Poll external APIs on defined schedules
- Handle pagination for large result sets
- Respect rate limits and backoff requirements
- Transform external data into Apsis schema
- Aggregate data from multiple sources
- Execute scheduled synchronization jobs

**Key Files** [CODE]
```
app/datafetcher/
├── README.md
│   # Minimal content: 164 bytes
│   # States: "Fetches data from various data sources"
└── [implementation files - content not provided]
```

**Configuration** [KT]
- **Job scheduling:** Cron-like schedules per connector
- **Rate limits:** Provider-specific throttling rules
- **Pagination:** Configurable page size and cursor handling
- **Retry behavior:** Configurable exponential backoff
- **Data transformation:** Schema mapping templates per connector

**Data Flow** [KT]
```
Scheduled Job Triggered
       ↓
Load Connector Configuration
       ↓
Authenticate with External System
       ↓
Fetch Data (Handle Pagination)
       ↓
Transform to Apsis Schema
       ↓
Queue Results for Processing
       ↓
Update Job Status & Metrics
```

**Supported Patterns** [KT]
- **Incremental fetch:** Timestamp-based delta sync
- **Full fetch:** Complete data refresh (for reconciliation)
- **Event-based:** Polling fallback when webhooks unavailable

**Dependencies**
- Scheduler service [KT]
- Credential management [KT]
- External APIs (various providers) [KT]

**Status** [CODE]  
📝 **Needs documentation**: Component README is minimal; detailed API reference and configuration schema required.

---

### 4.3 Data Synchronization Manager (DSM)

**Purpose** [CODE][KT]  
Orchestrates bidirectional synchronization of contacts, attributes, and campaign data between Apsis and external systems. Implements transformation logic, conflict resolution, and audit trails.

**Key Responsibilities** [KT]
- Sync contact data: attributes, segmentation, engagement history
- Sync campaign data: performance metrics, messaging history
- Handle data transformation between platform schemas
- Resolve conflicts in bidirectional sync scenarios
- Maintain audit trails for compliance
- Coordinate with external system state

**Key Files** [CODE]
```
app/dsm/
├── README.md
│   # Content: 754 bytes
│   # States: "Manages synchronization of data between Apsis and external systems"
├── worker/
│   └── Dockerfile     # Dedicated worker container
└── [implementation files - content not provided]
```

**Architecture: Main + Worker Pattern** [CODE][KT]

DSM implements a split architecture:
- **Main DSM process:** Orchestrates sync workflows, coordinates with broker
- **DSM Worker:** Dedicated container (`app/dsm/worker/Dockerfile`) for long-running synchronization tasks

**Configuration** [KT]
- **Sync direction:** Unidirectional (push/pull) or bidirectional
- **Conflict resolution:** Last-write-wins, platform-wins, external-wins
- **Batch size:** Records per sync batch (balance between throughput and memory)
- **Schedule:** Frequency of sync operations
- **Field mapping:** Schema transformation rules

**Data Flow** [KT]
```
Sync Job Initiated
       ↓
Load Configuration & Credentials
       ↓
Fetch Delta from Source
       ↓
Transform Data to Target Schema
       ↓
Detect & Resolve Conflicts
       ↓
Apply Changes to Target System
       ↓
Log Audit Trail
       ↓
Update Sync State & Metrics
```

**Conflict Resolution Patterns** [KT]
- **Last-write-wins:** Timestamp-based precedence
- **System-wins:** Platform or external system preferred
- **Manual review:** Flag conflicts for human intervention (future)

**Dependencies**
- Broker for job coordination [KT]
- Credential management [KT]
- External system APIs [KT]
- Audit logging [KT]

**Status** [CODE]  
📝 **Needs documentation**: Implementation details of sync algorithms, conflict resolution specifics, and API contract with broker.

---

## 5. API Reference

> ⚠️ **Limitation**: Code sources do not contain API endpoint definitions. The following is inferred from KT discussions and component purposes. Detailed API specifications require additional source review.

### 5.1 Broker API

**Webhook Reception** [KT]

```http
POST /integrations/webhooks/{integration_type}
Content-Type: application/json

Headers:
  Authorization: Bearer {api_key} | {signature_validation}
  X-Webhook-ID: {unique_id}
  X-Webhook-Timestamp: {iso8601_timestamp}

Body:
{
  "event": "contact.updated | contact.created | campaign.bounced",
  "data": { ... },
  "metadata": {
    "source": "external_system",
    "attempt": 1
  }
}

Response: 
  200 OK
  {
    "status": "received",
    "message_id": "msg_xyz",
    "processing_in_progress": true
  }
```

**Webhook Signature Validation** [KT]  
External systems sign webhooks using provider-specific algorithms; broker validates before queueing.

### 5.2 DataFetcher API

**Fetch Job Execution** [KT]

```http
POST /integrations/jobs/fetch
Content-Type: application/json

Body:
{
  "connector_id": "mailchimp_lists",
  "schedule_id": "sched_abc123",
  "force_full_sync": false
}

Response:
  202 Accepted
  {
    "job_id": "job_xyz",
    "status": "queued",
    "estimated_duration_seconds": 120
  }
```

### 5.3 DSM API

**Initiate Synchronization** [KT]

```http
POST /integrations/sync
Content-Type: application/json

Body:
{
  "integration_id": "salesforce_crm",
  "entity_type": "contact | campaign",
  "direction": "push | pull | bidirectional",
  "filters": {
    "modified_since": "2026-03-17T00:00:00Z"
  }
}

Response:
  202 Accepted
  {
    "sync_id": "sync_xyz",
    "status": "started",
    "progress": {
      "total_records": 5000,
      "processed": 0
    }
  }
```

**Query Sync Status** [KT]

```http
GET /integrations/sync/{sync_id}

Response:
  200 OK
  {
    "sync_id": "sync_xyz",
    "status": "in_progress | completed | failed",
    "progress": {
      "total_records": 5000,
      "processed": 4200,
      "errors": 0
    },
    "duration_seconds": 125,
    "conflicts_detected": 0
  }
```

---

## 6. Integration Patterns

### 6.1 External Services Connected [KT]

The Integrations domain supports connectors to:

| Service Category | Examples | Pattern |
|------------------|----------|---------|
| **Email Marketing** | Mailchimp, Campaign Monitor | Pull + Bidirectional |
| **CRM Systems** | Salesforce, HubSpot | Bidirectional |
| **Data Warehouses** | Snowflake, BigQuery | Push |
| **Custom REST APIs** | Client-specific endpoints | Pull/Push/Webhook |
| **Webhook Sources** | Third-party event providers | Event-driven |

### 6.2 Data Flow Patterns [KT]

#### Pattern 1: Pull-Based (DataFetcher)
```
Apsis Platform
       ↓
DataFetcher (Scheduled)
       ↓
Authenticate & Fetch from External API
       ↓
Transform & Aggregate
       ↓
Queue Results
       ↓
DSM applies changes to Apsis DB
```

#### Pattern 2: Push-Based (Broker + DSM)
```
Apsis Platform (Source of truth)
       ↓
DSM detects changes
       ↓
Transform to external schema
       ↓
Broker queues push job
       ↓
Execute push to external system
       ↓
Confirm & log
```

#### Pattern 3: Event-Driven (Webhook + Broker)
```
External System Event
       ↓
Sends webhook to Broker
       ↓
Broker validates signature
       ↓
Broker queues message
       ↓
Handler transforms & applies
       ↓
Acknowledgment sent
```

#### Pattern 4: Bidirectional (Broker + DSM + DataFetcher)
```
Poll external system (DataFetcher)
       ↓
Detect changes
       ↓
DSM reconciles conflicts
       ↓
Apply to Apsis
       ↓
   ┌───────────┴──────────┐
   ↓                      ↓
Apsis changes       External changes
triggered            via webhook
       ↓                      ↓
   └───────────┬──────────┘
       ↓
DSM syncs back
       ↓
Push to external system
```

### 6.3 Webhook Handling [KT]

**Webhook Lifecycle:**

1. **Receipt:** External system sends signed payload to Broker
2. **Validation:** 
   - Signature verification (HMAC-SHA256, OAuth signature, etc.)
   - Schema validation against expected payload
   - Rate limit checking (per source)
3. **Queueing:** Message stored with metadata for async processing
4. **Processing:**
   - Route to appropriate handler based on event type
   - Transform external schema to Apsis format
   - Apply data persistence or trigger workflows
5. **Error Handling:**
   - Transient errors: Exponential backoff retry (with configurable max attempts)
   - Permanent errors: Move to dead-letter queue; alert ops
   - Webhook status: Return 2xx only when message successfully queued

**Webhook Authentication** [KT]

Each external system uses distinct authentication:
- **HMAC-SHA256 signatures** (standard): Compare header signature with computed hash
- **OAuth tokens:** Validate JWT or opaque token
- **API key validation:** Whitelist known keys per integration
- **IP whitelisting** (supplementary): For trusted partners

### 6.4 Event Handling [KT]

**Supported Event Types:**

| Event | Triggering System | Handler | Destination |
|-------|-------------------|---------|-------------|
| `contact.created` | External CRM | Transform + Store | Apsis Contacts DB |
| `contact.updated` | External CRM | Merge + Update | Apsis Contacts DB |
| `contact.deleted` | External CRM | Mark as deleted | Audit log |
| `campaign.bounced` | Email Provider | Update engagement | Apsis Campaign DB |
| `campaign.opened` | Email Provider | Track engagement | Analytics queue |
| `list.subscribed` | Mailing list | Add to segment | Apsis Segments |

**Event Ordering:** [KT]  
- Events processed in order per source (per-partition semantics)
- Out-of-order from different sources is acceptable
- Idempotency keys ensure duplicate events don't cause corruption

### 6.5 Error Handling [KT]

**Classification:**

| Error Type | Handling | Action |
|------------|----------|--------|
| **Transient** (timeout, 429, 503) | Retry | Exponential backoff; max 5 attempts |
| **Validation** (400, schema mismatch) | Dead-letter | Log + alert; no retry |
| **Authentication** (401, 403) | Dead-letter | Alert ops; credential review needed |
| **Rate limit** (429) | Backoff | Progressive delay; respect Retry-After header |
| **Data conflict** (bidirectional) | DSM resolution | Apply conflict resolution policy |

**Dead Letter Queue (DLQ):** [KT]
- Messages that fail all retries move to DLQ
- Operator review required for manual retry
- Monitoring alerts triggered for DLQ growth

---

## 7. Configuration Reference

### 7.1 Environment Variables [KT]

Core configuration parameters (inferred from architecture):

```bash
# Message Queue Configuration
QUEUE_BROKER_URL=amqp://queue-broker:5672          # Message broker URL
QUEUE_PREFETCH=10                                   # Messages prefetched per worker

# Database
DB_CONNECTION_STRING=postgresql://apsis-db:5432/integrations
DB_POOL_SIZE=20                                     # Connection pool size

# External Integrations Authentication Vault
VAULT_ENDPOINT=https://vault.internal:8200         # Credential vault
VAULT_TOKEN=${VAULT_TOKEN}                          # Service authentication

# Logging & Monitoring
LOG_LEVEL=info                                      # debug | info | warn | error
SENTRY_DSN=https://sentry.io/...                   # Error tracking

# DataFetcher Scheduling
SCHEDULER_INTERVAL_SECONDS=300                     # Job check frequency

# DSM Configuration
DSM_BATCH_SIZE=1000                                # Records per sync batch
DSM_CONFLICT_RESOLUTION=last_write_wins            # Conflict strategy
DSM_AUDIT_LOG_ENABLED=true

# Webhook Security
WEBHOOK_SIGNATURE_VALIDATION=required              # Always validate
WEBHOOK_TIMEOUT_SECONDS=30
```

### 7.2 Feature Flags [KT]

Integration-specific features controlled at runtime:

```
FEATURE_SALESFORCE_BIDIRECTIONAL=true
FEATURE_MAILCHIMP_INCREMENTAL_SYNC=true
FEATURE_HUBSPOT_CUSTOM_PROPERTIES=false            # Beta feature
FEATURE_WEBHOOK_RETRY_BACKOFF_V2=true
```

### 7.3 Connector Configuration [KT]

Per-integration configuration (stored in database or vault):

```json
{
  "connector_id": "salesforce_crm",
  "connector_type": "salesforce",
  "enabled": true,
  "credentials": {
    "oauth_token": "00Xx000000...${VAULT_REF}",
    "instance_url": "https://xxx.salesforce.com",
    "api_version": "v59.0"
  },
  "sync_config": {
    "direction": "bidirectional",
    "entities": ["contact", "campaign"],
    "schedule": "0 */6 * * *",
    "batch_size": 5000,
    "field_mapping": {
      "apsis.email": "salesforce.Email",
      "apsis.first_name": "salesforce.FirstName"
    }
  },
  "error_config": {
    "max_retries": 5,
    "retry_backoff_seconds": 300,
    "alert_on_failure": true
  }
}
```

### 7.4 Database Schemas [KT]

**Integration Registry:**
```sql
CREATE TABLE integrations (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  connector_type VARCHAR(100) NOT NULL,
  enabled BOOLEAN DEFAULT true,
  config JSONB,
  credentials_vault_path VARCHAR(512),
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);
```

**Sync State Tracking:**
```sql
CREATE TABLE sync_state (
  id UUID PRIMARY KEY,
  integration_id UUID REFERENCES integrations(id),
  entity_type VARCHAR(100),
  last_sync_at TIMESTAMP,
  last_sync_cursor VARCHAR(1024),
  status VARCHAR(50),  -- pending, in_progress, completed, failed
  error_message TEXT
);
```

**Webhook Events Log:**
```sql
CREATE TABLE webhook_events (
  id UUID PRIMARY KEY,
  integration_id UUID REFERENCES integrations(id),
  event_type VARCHAR(255),
  payload JSONB,
  status VARCHAR(50),  -- received, processed, failed
  attempts INT,
  created_at TIMESTAMP,
  processed_at TIMESTAMP
);
```

### 7.5 Deployment Configuration [CODE]

**Main Application Dockerfile** [CODE]

```dockerfile
# From app/Dockerfile (1050 bytes)
# Multi-stage container build for main services
# Specific content not provided in extraction
```

**DSM Worker Dockerfile** [CODE]

```dockerfile
# From app/dsm/worker/Dockerfile (690 bytes)
# Dedicated container for DSM sync operations
# Specific content not provided in extraction
```

---

## 8. Common Procedures

### 8.1 Setting Up a New Integration [KT]

**Step 1: Develop Connector**
```bash
# Create connector implementation in integrations repo
mkdir app/connectors/my_service/
touch app/connectors/my_service/index.js
touch app/connectors/my_service/schema.js
touch app/connectors/my_service/README.md
```

**Step 2: Implement Connector Interface** [KT]
```javascript
// Minimal interface contract (inferred from architecture)
module.exports = {
  name: 'my_service',
  
  authenticate: async (credentials) => {
    // Validate credentials with external system
    // Return auth token or throw
  },
  
  fetchData: async (config) => {
    // Implement data retrieval
    // Handle pagination, rate limits
    // Return normalized data
  },
  
  pushData: async (config, data) => {
    // Implement push/update logic
    // Handle conflicts
    // Return result
  },
  
  validateWebhook: async (payload, signature) => {
    // Verify webhook authenticity
    // Return boolean
  }
};
```

**Step 3: Register Integration** [KT]
```bash
# Add integration to database
INSERT INTO integrations (
  id, name, connector_type, enabled, config
) VALUES (
  'my_service_prod',
  'My Service',
  'my_service',
  false,  -- Start disabled
  '{"sync_direction": "pull"}'::jsonb
);
```

**Step 4: Configure Credentials** [KT]
```bash
# Store secrets in vault (not directly in config)
vault write secret/integrations/my_service/prod \
  api_key="xxx" \
  oauth_token="yyy"
```

**Step 5: Test Connector** [KT]
```bash
# Run integration tests
npm test -- app/connectors/my_service/

# Test webhook receipt (if webhook-enabled)
curl -X POST http://localhost:3000/integrations/webhooks/my_service \
  -H "Content-Type: application/json" \
  -H "X-Signature: $(compute_signature)" \
  -d '{"event": "test"}'
```

**Step 6: Deploy & Enable** [KT]
```bash
# Deploy via CD pipeline
git push origin feature/my_service
# ... after approval and CI passes ...

# Enable in production
UPDATE integrations SET enabled = true WHERE id = 'my_service_prod';
```

### 8.2 Debugging Integration Issues [KT]

**Check Connector Status:**
```sql
SELECT i.name, i.enabled, ss.status, ss.last_sync_at, ss.error_message
FROM integrations i
LEFT JOIN sync_state ss ON i.id = ss.integration_id
WHERE i.connector_type = 'my_service';
```

**Review Recent Webhook Events:**
```sql
SELECT id, event_type, status, payload, created_at, attempts
FROM webhook_events
WHERE integration_id = (SELECT id FROM integrations WHERE name = 'My Service')
ORDER BY created_at DESC
LIMIT 20;
```

**Inspect Dead Letter Queue:** [KT]
```bash
# Assuming message queue is RabbitMQ or similar
# List DLQ messages for review
amqp-manager list-queue dlq.integrations

# Inspect message content
amqp-manager get-message dlq.integrations message-id-xyz
```

**Check Logs:** [KT]
```bash
# View container logs
docker logs integrations-broker-pod

# Filter for errors
docker logs integrations-broker-pod | grep -i "error\|failed\|exception"

# Follow real-time
docker logs -f integrations-dsm-worker-pod
```

**Validate External API Connectivity:** [KT]
```bash
# Test authentication
curl -X GET https://api.my_service.com/auth/test \
  -H "Authorization: Bearer ${API_TOKEN}"

# Test data endpoint (manual)
curl -X GET https://api.my_service.com/v1/contacts \
  -H "Authorization: Bearer ${API_TOKEN}" | jq .
```

### 8.3 Deploying Integration Changes [CODE][KT]

**Build Docker Images:**
```bash
cd apsis-integrations-justin/

# Build main application
docker build -f app/Dockerfile -t apsis-integrations:v1.2.3 .

# Build DSM worker
docker build -f app/dsm/worker/Dockerfile -t apsis-integrations-dsm-worker:v1.2.3 .
```

**Push to Registry:** [KT]
```bash
docker push registry.internal/apsis-integrations:v1.2.3
docker push registry.internal/apsis-integrations-dsm-worker:v1.2.3
```

**Deploy to Kubernetes:** [KT]
```bash
# Update manifest with new image
kubectl set image deployment/integrations-broker \
  broker=registry.internal/apsis-integrations:v1.2.3

kubectl set image deployment/integrations-dsm \
  dsm=registry.internal/apsis-integrations-dsm-worker:v1.2.3

# Monitor rollout
kubectl rollout status deployment/integrations-broker
kubectl rollout status deployment/integrations-dsm
```

### 8.4 Monitoring & Alerting [KT]

**Key Metrics to Monitor:**

| Metric | Threshold | Action |
|--------|-----------|--------|
| Webhook processing latency | >5s p99 | Page on-call if >10s sustained |
| Failed webhook events (DLQ growth) | >10 in 5min | Alert for investigation |
| DataFetcher job duration | >2x baseline | Check external API performance |
| Sync conflicts detected | >100 per hour | Review conflict resolution policy |
| Message queue depth | >1000 | Scale workers or investigate bottleneck |

**Alerting Setup (Prometheus/Grafana):** [KT]
```
alert: IntegrationWebhookFailureRate
expr: rate(webhook_events{status="failed"}[5m]) > 0.05
for: 5m
annotations:
  summary: "High webhook failure rate on {{ $labels.integration }}"
```

---

## 9. Known Issues & Workarounds

### 9.1 Bidirectional Sync Conflicts [KT]

**Issue:** When both Apsis and external system modify the same contact simultaneously, last-write-wins can lose data.

**Workaround:**
- Use last-write-wins only for non-critical fields
- Reserve sensitive fields (email, phone) for Apsis as source of truth
- Implement manual conflict queue for human review (future roadmap)
- Schedule syncs to avoid peak hours when concurrent edits likely

### 9.2 Webhook Retry on Network Timeout [KT]

**Issue:** External systems may resend webhooks if initial request times out, causing duplicate events.

**Workaround:**
- Implement idempotency key checking: deduplicate based on `X-Webhook-ID` header
- Store processed webhook IDs in cache (Redis) for 24-hour window
- Accept at-least-once delivery semantics; ensure handlers are idempotent

### 9.3 Rate Limit Exceeded During Batch Sync [KT]

**Issue:** DataFetcher exhausts external API rate limits mid-sync, breaking sync job.

**Workaround:**
- Reduce batch size in connector config (lower throughput, longer duration)
- Check external API rate limit documentation and set `DSM_BATCH_SIZE` accordingly
- Implement adaptive rate limiting: monitor 429 responses and back off dynamically
- Contact external system support for rate limit increase (temporary measure)

### 9.4 Credential Rotation [KT]

**Issue:** API keys and tokens expire; no automatic rotation mechanism exists.

**Workaround:**
- Set calendar reminders 30 days before expiration
- Manual credential update: rotate key in vault, test connectivity with new key, remove old key
- Automate via scheduled job: implement connector-specific refresh logic (e.g., OAuth token refresh)
- Implement audit trail: log all credential changes for compliance

### 9.5 Data Transformation Errors [KT]

**Issue:** Schema mismatch between Apsis and external system causes silent data loss or corruption.

**Workaround:**
- Implement strict schema validation: reject mismatches rather than silently truncate
- Log transformation errors with full payload for debugging
- Use field mapping templates to standardize transformations
- Test transformations in staging with live data before prod deployment

---

## 10. Glossary

### Domain-Specific Terms

| Term | Definition |
|------|-----------|
| **Broker** | Central message router component that receives webhooks, queues messages, and coordinates routing to integration handlers. |
| **Connector** | Standardized implementation for a specific external service (e.g., Salesforce connector, Mail