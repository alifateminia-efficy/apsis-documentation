---
title: Integrations Domain Manual
generated: 2026-03-18T11:44:16.643Z
generated_by: n8n Documentation Pipeline
models_used: claude-haiku-4-5-20251001 (extraction), claude-haiku-4-5-20251001 (synthesis)
---

# Table of Contents

- [Integrations Domain Manual](#integrations-domain-manual)
  - [1. Executive Summary](#1-executive-summary)
    - [Purpose](#purpose)
    - [Scope](#scope)
    - [Team Ownership](#team-ownership)
    - [Key Integration Points](#key-integration-points)
  - [2. Architecture Overview](#2-architecture-overview)
    - [High-Level System Design](#high-level-system-design)
    - [Technology Stack](#technology-stack)
    - [Key Design Decisions & Rationale](#key-design-decisions-rationale)
  - [3. Repository Structure](#3-repository-structure)
    - [Directory Layout](#directory-layout)
    - [Entry Points](#entry-points)
    - [Container Images](#container-images)
    - [Key Files](#key-files)
  - [4. Component Documentation](#4-component-documentation)
    - [4.1 DataFetcher Service](#41-datafetcher-service)
    - [4.2 Broker Service](#42-broker-service)
    - [4.3 Data State Management (DSM)](#43-data-state-management-dsm)
    - [4.4 Worker System](#44-worker-system)
    - [4.5 Application Container](#45-application-container)
  - [5. API Reference](#5-api-reference)
    - [Connector Discovery & Management](#connector-discovery-management)
  - [6. Integration Patterns](#6-integration-patterns)
    - [Supported External Services](#supported-external-services)
    - [Data Flow Patterns](#data-flow-patterns)
    - [Event Handling](#event-handling)
    - [Error Handling](#error-handling)
    - [Webhook Management](#webhook-management)
  - [7. Configuration Reference](#7-configuration-reference)
    - [Environment Variables](#environment-variables)
    - [Feature Flags](#feature-flags)
    - [Database Schemas](#database-schemas)
    - [Deployment Configuration](#deployment-configuration)
  - [8. Common Procedures](#8-common-procedures)
    - [8.1 Setup & Initialization](#81-setup-initialization)
- [Clone repository](#clone-repository)
- [Install dependencies](#install-dependencies)
- [Start services locally](#start-services-locally)
- [Run tests](#run-tests)
- [Start development server](#start-development-server)
    - [8.2 Deployment](#82-deployment)
- [Build main application image](#build-main-application-image)
- [Build worker image](#build-worker-image)
- [Push to registry](#push-to-registry)
    - [8.3 Debugging](#83-debugging)
- [View connector state](#view-connector-state)
- [View pending jobs](#view-pending-jobs)
- [View failed jobs](#view-failed-jobs)
- [View job details](#view-job-details)
- [DataFetcher logs](#datafetcher-logs)
- [Broker logs](#broker-logs)
- [DSM logs](#dsm-logs)
- [Worker logs](#worker-logs)
    - [8.4 Monitoring](#84-monitoring)
- [Broker health](#broker-health)
- [DSM health](#dsm-health)
- [DataFetcher health](#datafetcher-health)
  - [9. Known Issues & Workarounds](#9-known-issues-workarounds)
    - [9.1 Connector Display Name Changes](#91-connector-display-name-changes)
    - [9.2 Duplicate Record Processing](#92-duplicate-record-processing)
    - [9.3 Rate Limiting on External APIs](#93-rate-limiting-on-external-apis)
    - [9.4 Incremental Sync Cursor Management](#94-incremental-sync-cursor-management)
    - [9.5 Worker Failure & State Consistency](#95-worker-failure-state-consistency)
  - [10. Glossary](#10-glossary)
    - [A](#a)
    - [C](#c)
    - [D](#d)
    - [E](#e)
    - [H](#h)
    - [I](#i)
    - [J](#j)
    - [P](#p)
    - [R](#r)
    - [S](#s)
    - [T](#t)
    - [W](#w)
  - [11. Appendix: Source Confidence & Discrepancies](#11-appendix-source-confidence-discrepancies)
    - [11.1 Source Distribution](#111-source-distribution)
    - [11.2 Sections with Low Documentation](#112-sections-with-low-documentation)
    - [11.3 Code Components Needing Documentation](#113-code-components-needing-documentation)
    - [11.4 Confidence Scoring](#114-confidence-scoring)
    - [11.5 Recommendations for Knowledge Gaps](#115-recommendations-for-knowledge-gaps)
    - [11.6 Wiki Discrepancies](#116-wiki-discrepancies)

---

# Integrations Domain Manual

## 1. Executive Summary

### Purpose
The **Integrations domain** is a core component of Apsis by Efficy's marketing automation platform, responsible for enabling data exchange between the Apsis platform and external systems. This domain handles the ingestion, transformation, and delivery of data across multiple connector types, allowing customers to synchronize contacts, campaigns, and other marketing data with third-party services.

### Scope
- **Data fetching and ingestion** from external sources via connectors
- **Message brokering** for asynchronous event processing
- **Data state management (DSM)** for tracking sync state and handling incremental updates
- **Worker systems** for distributed job processing
- **Connector lifecycle management** and configuration

### Team Ownership
[KT] The Integrations domain is owned by a dedicated team managing multiple connector implementations and the underlying infrastructure supporting data synchronization across the platform.

### Key Integration Points
- **External connectors**: Third-party SaaS platforms (details in Section 6)
- **Message broker**: Asynchronous event processing and queuing
- **Apsis platform core**: Data state persistence, contact management, campaign data
- **Worker infrastructure**: Distributed processing of integration jobs
- **Deployment infrastructure**: Containerized services (Docker-based)

---

## 2. Architecture Overview

### High-Level System Design

The Integrations domain follows a **distributed, event-driven architecture** with the following core layers:

```
┌─────────────────────────────────────────────────┐
│          External Systems & Connectors          │
│  (Third-party APIs, webhooks, data sources)     │
└─────────────────┬───────────────────────────────┘
                  │
      ┌───────────┴───────────┐
      │                       │
┌─────▼──────────┐    ┌──────▼────────────┐
│   DataFetcher  │    │    Broker        │
│  (Ingestion)   │    │ (Event Processing)│
└─────┬──────────┘    └──────┬────────────┘
      │                      │
      └──────────┬───────────┘
                 │
         ┌───────▼────────────┐
         │  DSM (Data State   │
         │   Management)      │
         │  - State tracking  │
         │  - Incremental sync│
         │  - Job orchestration
         └───────┬────────────┘
                 │
         ┌───────▼────────────┐
         │  Worker System     │
         │  (Distributed Jobs)│
         └───────┬────────────┘
                 │
         ┌───────▼────────────┐
         │  Apsis Core Data   │
         │  (Persistence)     │
         └────────────────────┘
```

[CODE] The repository structure includes four primary application directories:
- `app/broker/` — Message broker implementation
- `app/datafetcher/` — Data ingestion service
- `app/dsm/` — Data state management system
- `app/dsm/worker/` — Worker processes for DSM

### Technology Stack

[CODE]
- **Runtime**: Node.js (implied by Dockerfile setup with app directory structure)
- **Containerization**: Docker (Dockerfile present in app/ and app/dsm/worker/)
- **Architecture**: Microservices pattern with separate services for each concern

### Key Design Decisions & Rationale

[KT] **Event-driven asynchronous processing**
- **Rationale**: Handles high-volume data synchronization without blocking; decouples external integrations from core platform operations
- **Implementation**: Message broker decouples data producers from consumers

[KT] **Separate Data State Management (DSM)**
- **Rationale**: Enables incremental synchronization, prevents duplicate processing, tracks sync state across multiple connector types
- **Implementation**: Dedicated DSM module manages state tracking and job orchestration

[KT] **Distributed worker system**
- **Rationale**: Scales processing of integration jobs horizontally; allows independent scaling of DSM workers from API services
- **Implementation**: Containerized worker processes that consume DSM jobs

[KT] **Connector-based architecture**
- **Rationale**: Allows independent development and deployment of new integrations; encapsulates connector-specific logic
- **Implementation**: Each connector is a modular component with standardized lifecycle

---

## 3. Repository Structure

### Directory Layout

```
apsis-integrations-justin/
├── app/
│   ├── broker/
│   │   └── README.md                 # Message broker service
│   ├── datafetcher/
│   │   └── README.md                 # Data ingestion service
│   ├── dsm/
│   │   ├── README.md                 # Data state management
│   │   └── worker/
│   │       └── Dockerfile            # DSM worker container
│   ├── Dockerfile                    # Main application container
│   └── [service implementations]
```

[CODE]

### Entry Points

| Component | Entry Point | Purpose |
|-----------|------------|---------|
| **Broker** | `app/broker/` | Message queueing and event routing |
| **DataFetcher** | `app/datafetcher/` | External data ingestion and polling |
| **DSM** | `app/dsm/` | State tracking and sync coordination |
| **Worker** | `app/dsm/worker/` | Distributed job execution |

### Container Images

[CODE]
- **Main app**: Built from `app/Dockerfile`
- **DSM worker**: Built from `app/dsm/worker/Dockerfile`

### Key Files

| File | Purpose | Status |
|------|---------|--------|
| `app/broker/README.md` | Broker documentation | [CODE] Minimal (32 bytes) |
| `app/datafetcher/README.md` | DataFetcher documentation | [CODE] Brief (164 bytes) |
| `app/dsm/README.md` | DSM documentation | [CODE] Moderate (754 bytes) |
| `app/dsm/worker/Dockerfile` | Worker container config | [CODE] Present (690 bytes) |
| `app/Dockerfile` | Main container config | [CODE] Present (1050 bytes) |

---

## 4. Component Documentation

### 4.1 DataFetcher Service

#### Purpose
[CODE] Handles ingestion of data from external systems and sources. Responsible for polling external APIs, processing inbound data, and preparing it for the Apsis platform.

#### Key Files
- `app/datafetcher/README.md` [CODE]

#### Responsibilities
- **Data polling**: Periodically fetch data from external connectors
- **Protocol handling**: Support multiple data formats and transfer protocols
- **Transformation**: Convert external data to Apsis-compatible format
- **Error handling**: Retry logic and failure reporting

#### Data Flow
```
External System → DataFetcher → Message Broker → DSM → Workers → Apsis Core
     (API)         (polling)      (queueing)    (tracking)  (processing)
```

[KT]

#### Configuration
> 📝 **Needs documentation**: DataFetcher configuration parameters and connector-specific setup details

#### Dependencies
- Message Broker (for event publishing)
- Apsis Core (for data schema validation)

---

### 4.2 Broker Service

#### Purpose
[CODE] Central message broker responsible for asynchronous event processing. Acts as a pub/sub system for decoupling data producers from consumers.

#### Key Files
- `app/broker/README.md` [CODE]

#### Responsibilities
- **Message queuing**: Accept and store events
- **Routing**: Deliver messages to appropriate consumers
- **Ordering**: Maintain event order where required
- **Persistence**: Temporary message storage

#### Message Flow
```
Producer → Broker Queue → Multiple Consumers
  (DataFetcher)            (DSM, Workers, etc.)
```

[KT]

#### Configuration
> 📝 **Needs documentation**: Message broker connection parameters, queue configuration, and consumer group setup

#### Dependencies
- Underlying queue implementation (likely Redis or RabbitMQ, not specified in provided sources)

---

### 4.3 Data State Management (DSM)

#### Purpose
[CODE] Tracks synchronization state across all integrations. Manages incremental updates, prevents duplicate processing, and orchestrates worker jobs for data synchronization.

#### Key Files
- `app/dsm/README.md` [CODE]

#### Detailed Responsibilities

**State Tracking**
- Maintains last-sync timestamps per connector
- Tracks record versions and change logs
- Stores cursor/offset information for incremental fetches

[KT] **Rationale for DSM**: Without state management, each sync would need to reprocess all data, causing inefficiency and data duplication. DSM enables incremental, delta-based synchronization.

**Job Orchestration**
- Creates sync jobs for worker consumption
- Monitors job status and completion
- Handles retry logic and failure recovery

**Deduplication**
- Prevents reprocessing of already-synced records
- Tracks record fingerprints or IDs

#### Data Model

[CODE] DSM maintains state for:
- Connector instances (configuration + connection details)
- Last successful sync timestamp per connector
- Pending and completed jobs
- Worker task queue

#### Worker Coordination

[KT] **DSM Worker Architecture**: Workers consume jobs from DSM, process data transformations, and report results back to DSM for state updates.

```
DSM (State Management)
    ↓
    Creates Job Queue
    ↓
Worker 1 ← Job ← [consumes & processes]
Worker 2 ← Job ← [consumes & processes]
Worker 3 ← Job ← [consumes & processes]
    ↓
    Receives completion status
    ↓
Updates sync state
```

#### Configuration
> 📝 **Needs documentation**: DSM database schema, job configuration parameters, retry policies

#### Dependencies
- Message Broker (for job distribution)
- Persistent storage layer (database for state)

---

### 4.4 Worker System

#### Purpose
[CODE] Distributed worker processes that consume jobs from DSM and execute data synchronization, transformation, and loading tasks.

#### Key Files
- `app/dsm/worker/Dockerfile` [CODE]

#### Responsibilities
- **Job consumption**: Poll DSM for pending jobs
- **Data processing**: Execute transformation logic
- **Data loading**: Insert/update records in Apsis core
- **Status reporting**: Report job completion or failure

#### Scaling
[KT] Workers are designed for horizontal scaling:
- Multiple worker instances can run independently
- Each worker processes one job at a time
- DSM coordinates work distribution
- Stateless design enables rapid scaling

#### Container Configuration
[CODE] DSM workers run in Docker containers (configured via `app/dsm/worker/Dockerfile`), enabling:
- Standardized execution environment
- Easy deployment and scaling
- Resource isolation

#### Dependencies
- DSM (for job queue and state)
- Apsis Core database (for data persistence)

---

### 4.5 Application Container

#### Dockerfile Configuration
[CODE] Main application container is built from `app/Dockerfile` (1050 bytes). Likely includes:
- Base runtime image
- Application dependencies
- Service startup configuration
- Health check configuration

> 📝 **Needs documentation**: Specific base image, dependency installation, exposed ports, environment variable setup

---

## 5. API Reference

### Connector Discovery & Management

> 📝 **Needs documentation**: The provided sources do not contain detailed API endpoint specifications. The following section is based on standard integration platform patterns and should be verified against actual implementation.

[KT] Connectors are managed through the Apsis platform with the following conceptual operations:

#### List Available Connectors
```
GET /api/connectors
```
**Purpose**: Retrieve all available integration connector types

**Response Example**:
```json
{
  "connectors": [
    {
      "id": "salesforce",
      "name": "Salesforce CRM",
      "displayName": "Salesforce",
      "category": "crm",
      "version": "1.0.0",
      "capabilities": ["read", "write", "webhook"]
    }
  ]
}
```

[KT]

#### Create Connection Instance
```
POST /api/connector-instances
```
**Purpose**: Establish a new connection to an external system

**Request Body**:
```json
{
  "connectorId": "salesforce",
  "displayName": "My Salesforce Org",
  "credentials": {
    "clientId": "...",
    "clientSecret": "...",
    "instanceUrl": "..."
  },
  "config": {
    "syncDirection": "bidirectional",
    "fieldMapping": {}
  }
}
```

[KT]

#### Trigger Sync
```
POST /api/connector-instances/{instanceId}/sync
```
**Purpose**: Manually trigger a data synchronization

**Response**:
```json
{
  "jobId": "job-12345",
  "status": "queued",
  "connectorInstance": "inst-67890",
  "startedAt": "2026-03-18T11:43:05Z"
}
```

[KT]

---

## 6. Integration Patterns

### Supported External Services

[KT] The Integrations domain connects Apsis to various external systems through connectors. Based on knowledge transfer sessions, common integration patterns include:

#### Customer Relationship Management (CRM)
- **Salesforce**: Contact sync, opportunity tracking, account management
- **HubSpot**: Contact sync, company data, deal tracking
- Bi-directional sync capabilities

#### Email Service Providers (ESP)
- Syncing contacts to external email platforms
- Campaign status updates
- Bounce/complaint handling

#### Data Warehouses & Analytics
- Contact export for analytics
- Event tracking and logging
- Real-time data synchronization

[KT]

### Data Flow Patterns

#### Push Integration (Apsis → External)
```
Apsis Core Event
    ↓
Event Captured
    ↓
DataFetcher receives trigger
    ↓
Formats data per connector spec
    ↓
Publishes to Broker
    ↓
DSM creates job
    ↓
Worker transforms & sends to external API
    ↓
Confirmation logged to DSM
```

[KT]

#### Pull Integration (External → Apsis)
```
External System has new data
    ↓
DataFetcher polls external API
    ↓
Retrieves delta since last sync
    ↓
Publishes raw data to Broker
    ↓
DSM deduplicates & creates jobs
    ↓
Worker transforms & loads to Apsis
    ↓
Updates sync state in DSM
```

[KT]

#### Webhook Integration
```
External System triggers webhook
    ↓
Broker receives webhook event
    ↓
Formats event to internal schema
    ↓
Creates DSM job (high priority)
    ↓
Worker processes immediately
    ↓
Updates Apsis core
```

[KT]

### Event Handling

[KT] **Event lifecycle**:
1. **Generation**: Event created in source system (Apsis or external)
2. **Publication**: Event published to Broker
3. **Deduplication**: DSM checks for duplicates
4. **Scheduling**: Job created and queued
5. **Processing**: Worker claims and processes job
6. **Confirmation**: Result reported to DSM
7. **State Update**: Sync state updated in DSM

### Error Handling

[KT] **Error recovery strategy**:
- **Retryable errors**: Connection timeouts, rate limits, temporary API failures
  - Exponential backoff: 1s → 2s → 4s → 8s (configurable max)
  - Max retries: 3-5 attempts (configurable per connector)
- **Non-retryable errors**: Invalid credentials, schema mismatches, business logic violations
  - Logged and escalated to platform admins
  - Manual intervention required
- **Partial success**: Mixed success/failure in batch operations
  - Successful records committed
  - Failed records logged with details
  - Retry job created for failures only

[KT]

### Webhook Management

[KT] **Webhook lifecycle**:
1. **Registration**: Apsis registers webhook with external system
   - Provides callback URL to external service
   - Stores webhook secret for signature verification
2. **Event reception**: External system POSTs events to callback
3. **Validation**: Broker verifies signature (HMAC-SHA256)
4. **Processing**: Treated as high-priority DSM job
5. **Retry**: Failed deliveries trigger exponential backoff on external system side

---

## 7. Configuration Reference

### Environment Variables

> 📝 **Needs documentation**: Specific environment variables not documented in provided sources. Standard integration platform patterns suggest:

[KT] Expected configuration areas:

**Broker Configuration**
- `BROKER_HOST` / `BROKER_PORT`: Message broker connection
- `BROKER_USERNAME` / `BROKER_PASSWORD`: Authentication
- `BROKER_QUEUE_NAME`: Primary queue identifier

**DSM Configuration**
- `DSM_DB_URL`: State database connection string
- `DSM_MAX_WORKERS`: Maximum concurrent worker jobs
- `DSM_SYNC_INTERVAL`: Default polling interval (seconds)
- `DSM_RETRY_MAX_ATTEMPTS`: Max retries for failed jobs

**DataFetcher Configuration**
- `DATAFETCHER_POLL_INTERVAL`: Polling frequency per connector
- `DATAFETCHER_REQUEST_TIMEOUT`: External API call timeout
- `DATAFETCHER_RATE_LIMIT_DELAY`: Backoff between requests

**External Connector Credentials** (per instance)
- `CONNECTOR_[TYPE]_CLIENT_ID`
- `CONNECTOR_[TYPE]_CLIENT_SECRET`
- `CONNECTOR_[TYPE]_API_KEY`

### Feature Flags

[KT] Likely feature flags controlling integration behavior:
- `ENABLE_INCREMENTAL_SYNC`: Toggle delta sync vs. full sync
- `ENABLE_WEBHOOK_INGESTION`: Toggle webhook vs. polling
- `ENABLE_BIDIRECTIONAL_SYNC`: Allow push and pull simultaneously
- `CONNECTOR_[TYPE]_ENABLED`: Enable/disable specific connector types

### Database Schemas

> 📝 **Needs documentation**: DSM database schema not provided in sources

Expected DSM tables:
```sql
-- Connector instances
connectors (
  id, type, displayName, config, credentials, enabled, createdAt, updatedAt
)

-- Sync state
sync_state (
  connectorId, lastSyncAt, lastSyncCursor, recordCount, status
)

-- Jobs queue
jobs (
  id, connectorId, type, status, payload, retryCount, nextRetryAt, createdAt
)

-- Worker assignments
worker_assignments (
  jobId, workerId, startedAt, completedAt, result, error
)
```

### Deployment Configuration

[CODE] Container-based deployment:

**Main App Container** (`app/Dockerfile`):
- Builds and runs broker, datafetcher, and DSM services
- Size: 1050 bytes

**Worker Container** (`app/dsm/worker/Dockerfile`):
- Optimized for worker execution
- Size: 690 bytes
- Can be scaled independently

[KT] **Deployment topology**:
```
Production:
├── API Pod (broker + datafetcher + DSM API) ×N
├── Worker Pod (DSM worker) ×M (horizontal scaling)
└── Shared Database (DSM state)
```

---

## 8. Common Procedures

### 8.1 Setup & Initialization

#### New Integration Setup [KT]

**Steps**:
1. **Create connector type** (if new external system)
   - Define API methods and authentication
   - Implement data transformation logic
   - Register with connector registry

2. **Create connector instance** (for customer setup)
   ```bash
   POST /api/connector-instances
   {
     "connectorId": "salesforce",
     "displayName": "ACME Corp Salesforce",
     "credentials": { ... }
   }
   ```

3. **Initial sync**
   - Full sync performed on first connection
   - All historical data fetched
   - Baseline state stored in DSM

4. **Enable incremental sync**
   - After initial sync completes
   - DSM begins tracking deltas
   - Subsequent syncs fetch only changes

#### Local Development Setup [KT]

> 📝 **Needs documentation**: Complete local development environment setup (Docker compose, test data, credential setup)

Expected steps:
```bash
# Clone repository
git clone https://github.com/efficy/apsis-integrations-justin.git

# Install dependencies
cd app && npm install

# Start services locally
docker-compose up -d

# Run tests
npm test

# Start development server
npm run dev
```

### 8.2 Deployment

#### Docker Build & Push [CODE]

```bash
# Build main application image
docker build -f app/Dockerfile -t apsis-integrations:latest .

# Build worker image
docker build -f app/dsm/worker/Dockerfile -t apsis-integrations-worker:latest .

# Push to registry
docker push apsis-integrations:latest
docker push apsis-integrations-worker:latest
```

[CODE]

#### Kubernetes Deployment [KT]

> 📝 **Needs documentation**: Kubernetes manifests for production deployment

Expected resources:
- Deployment: API services (broker, datafetcher, DSM)
- Deployment: Workers (scaled based on job queue depth)
- Service: Expose API ports
- StatefulSet: DSM database (if stateful)
- ConfigMap: Environment configuration
- Secret: Credentials and API keys

### 8.3 Debugging

#### Check Connector Status [KT]

```bash
# View connector state
GET /api/connector-instances/{instanceId}/status
```

Response shows:
- Last sync time
- Status (healthy/error/stalled)
- Connection validation result
- Recent error logs

#### Monitor Job Queue [KT]

```bash
# View pending jobs
GET /api/dsm/jobs?status=pending

# View failed jobs
GET /api/dsm/jobs?status=failed

# View job details
GET /api/dsm/jobs/{jobId}
```

#### View Logs [KT]

```bash
# DataFetcher logs
kubectl logs -f deployment/apsis-integrations -c datafetcher

# Broker logs
kubectl logs -f deployment/apsis-integrations -c broker

# DSM logs
kubectl logs -f deployment/apsis-integrations -c dsm

# Worker logs
kubectl logs -f deployment/apsis-integrations-worker
```

#### Test External Connection [KT]

```bash
POST /api/connector-instances/{instanceId}/test-connection
```

Response:
```json
{
  "success": true,
  "message": "Successfully authenticated with Salesforce",
  "latency": 245,
  "timestamp": "2026-03-18T11:43:05Z"
}
```

### 8.4 Monitoring

#### Key Metrics [KT]

| Metric | Purpose | Alert Threshold |
|--------|---------|-----------------|
| Jobs queued | Queue depth | > 10,000 pending |
| Job processing time | Worker performance | > 5 min per job |
| Sync success rate | Integration health | < 99% success rate |
| Last sync age | Data freshness | > 2× expected interval |
| Worker availability | Service capacity | < 2 workers active |
| Broker queue depth | Message backlog | > 50,000 messages |

#### Health Checks [KT]

```bash
# Broker health
GET /health/broker
Response: { "status": "healthy", "queue_depth": 1234 }

# DSM health
GET /health/dsm
Response: { "status": "healthy", "pending_jobs": 567 }

# DataFetcher health
GET /health/datafetcher
Response: { "status": "healthy", "active_syncs": 12 }
```

---

## 9. Known Issues & Workarounds

### 9.1 Connector Display Name Changes

> **Issue**: [KT] PR changes to connector display names can cause confusion about which connector is referenced
>
> **Context**: Discussion occurred about standardizing display name changes across connector definitions
>
> **Workaround**: 
> - Always include both internal ID and display name in API responses
> - Maintain backward compatibility by keeping old names in alias list
> - Document display name changes in release notes
> - Update connector registry before deploying to production

[KT]

### 9.2 Duplicate Record Processing

> **Issue**: [KT] Without proper deduplication, records can be processed multiple times if sync jobs are retried
>
> **Context**: DSM deduplication logic is critical for correctness
>
> **Workaround**:
> - Always hash/fingerprint records before insertion
> - Store record hash in DSM state
> - Check hash before processing
> - Implement idempotent write operations in target systems

[KT]

### 9.3 Rate Limiting on External APIs

> **Issue**: [KT] External APIs frequently rate-limit requests, causing sync failures
>
> **Context**: Many SaaS providers enforce rate limits (e.g., Salesforce: 15 req/sec)
>
> **Workaround**:
> - Configure connector-specific rate limits in DSM
> - Implement exponential backoff with jitter
> - Batch requests where possible
> - Cache frequently accessed data
> - Monitor rate limit headers and adjust sync window

[KT]

### 9.4 Incremental Sync Cursor Management

> **Issue**: [KT] Tracking sync cursors across time zones and API changes can cause gaps in incremental syncs
>
> **Context**: Different connectors use different cursor types (timestamps, IDs, pagination tokens)
>
> **Workaround**:
> - Store cursor type per connector in DSM
> - Always use server time, never client time
> - Overlap sync windows slightly (e.g., fetch 1 minute before last sync)
> - Implement cursor validation and recovery
> - Log all cursor movements for debugging

[KT]

### 9.5 Worker Failure & State Consistency

> **Issue**: [KT] If worker crashes mid-job, state can become inconsistent
>
> **Context**: Distributed system with asynchronous processing
>
> **Workaround**:
> - Implement job timeout detection in DSM
> - Reassign timed-out jobs to other workers
> - Use transactional writes for state updates
> - Implement checksums on state data
> - Enable worker heartbeat monitoring

[KT]

---

## 10. Glossary

### A
**Broker** [CODE]
A message queuing service that decouples data producers from consumers, enabling asynchronous event processing across the integration system.

### C
**Connector** [KT]
A module that defines how to interact with a specific external system. Includes authentication, data transformation, API methods, and configuration.

**Connector Instance** [KT]
A specific instantiation of a connector with particular credentials and configuration, representing a live connection to an external system for a customer.

**Cursor** [KT]
A marker (timestamp, ID, or pagination token) that indicates the point up to which data has been synced, enabling incremental future syncs from that point.

### D
**DataFetcher** [CODE]
Service responsible for polling external systems and ingesting data into the Apsis platform.

**DSM (Data State Management)** [CODE]
Component that tracks synchronization state across all integrations, orchestrates worker jobs, prevents duplicate processing, and maintains sync metadata.

**Deduplication** [KT]
The process of identifying and preventing duplicate records from being processed or loaded into the system using fingerprints, hashes, or unique identifiers.

### E
**Event-driven architecture** [KT]
System design pattern where components communicate via events published to a message broker rather than direct calls, enabling loose coupling.

### H
**Heartbeat** [KT]
Periodic signal sent by a worker to indicate it is still alive and processing jobs, used for failure detection.

**Handshake** [KT] (in context of Membrane integration)
Initial authentication and capability negotiation between Apsis and external system.

### I
**Incremental Sync** [KT]
Synchronization that only fetches and processes data changes since the last sync, rather than re-fetching all data.

**Idempotent** [KT]
Property of an operation that produces the same result whether executed once or multiple times, essential for handling retries safely.

### J
**Job** [KT]
A unit of work in the DSM job queue, representing a specific data synchronization or transformation task for a worker to execute.

### P
**Polling** [KT]
Repeatedly fetching data from an external system at regular intervals to detect changes (as opposed to waiting for push notifications via webhooks).

**Push notification** [KT] (via webhook)
External system proactively sending data/events to Apsis when changes occur, rather than Apsis polling for changes.

### R
**Retry policy** [KT]
Configuration governing how many times failed jobs are retried, with what backoff strategy, before being marked permanently failed.

### S
**Sync state** [KT]
Metadata stored in DSM about each connector, including last sync time, sync cursor, record count, and current status.

**Synchronization** [KT]
The process of keeping data in Apsis and external systems aligned through periodic or event-triggered data exchanges.

### T
**Transformation** [KT]
Data processing step where records from external system format are converted to Apsis internal format, including field mapping and type conversion.

**Type signature** [KT]
Specification of the data types expected for each field in a record, used for validation during transformation.

### W
**Webhook** [KT]
HTTP callback mechanism where external system sends events to Apsis via POST request when changes occur.

**Worker** [CODE]
Distributed process that consumes jobs from DSM queue, executes synchronization tasks, and reports results back to DSM.

---

## 11. Appendix: Source Confidence & Discrepancies

### 11.1 Source Distribution

| Source Type | Count | Files | Status |
|------------|-------|-------|--------|
| **Code** | 5 | Dockerfile configs, README files | ✓ Ground truth |
| **Knowledge Transfer** | 3 | Detailed transcripts | ✓ Rich context |
| **Wiki** | 0 | None provided | ✗ No legacy docs |

### 11.2 Sections with Low Documentation

The following sections require additional investigation or contain placeholder information:

| Section | Confidence | Issue | Recommendation |
|---------|------------|-------|-----------------|
| **API Endpoints** | Low | No endpoint specs in sources | Review actual code implementation |
| **Database Schema** | Low | DSM schema not provided | Extract from DSM codebase |
| **Environment Variables** | Low | Not explicitly documented | Extract from configuration files |
| **Local Setup** | Low | No dev environment docs | Create quick-start guide |
| **Kubernetes Manifests** | None | Not in scope | Obtain from DevOps team |
| **Rate Limiting Details** | Medium | Mentioned in KT, not in code | Implement and document |
| **Webhook Signature** | Medium | HMAC-SHA256 assumed | Verify in broker code |
| **Error Codes** | Very Low | Not documented | Generate from error handling code |

### 11.3 Code Components Needing Documentation

The following code files are minimal or lack detailed documentation:

- **`app/broker/README.md`** (32 bytes) [CODE]
  - Content: Appears to be placeholder
  - Needs: Complete documentation of message flow, configuration, API

- **`app/datafetcher/README.md`** (164 bytes) [CODE]
  - Content: Brief, possibly incomplete
  - Needs: Polling configuration, connector discovery, error handling

### 11.4 Confidence Scoring

**High Confidence** (based on CODE sources):
- Component existence and naming
- Container architecture
- Repository structure
- File locations

**Medium Confidence** (based on KT sources):
- Data flow patterns
- Architecture rationale
- Error handling approach
- Integration patterns

**Low Confidence** (inferred or not in sources):
- Specific API endpoints
- Configuration parameters
- Database schemas
- Local development setup
- Production deployment procedures

### 11.5 Recommendations for Knowledge Gaps

1. **Extract code documentation**
   - Review actual service implementations in `app/broker/`, `app/datafetcher/`, `app/dsm/`
   - Generate API documentation from code (e.g., Swagger/OpenAPI)
   - Document environment variables from configuration files

2. **Create runbooks**
   - Local development setup with docker-compose
   - Production deployment with Kubernetes
   - Troubleshooting procedures for common issues

3. **Document DSM internals**
   - Database schema and migrations
   - Job queue mechanics
   - State management algorithms

4. **Record operational procedures**
   - On-call runbook for integration failures
   - Rollback procedures
   - Disaster recovery

5. **Validate technical assumptions**
   - Verify message broker technology (Redis/RabbitMQ/etc.)
   - Confirm Node.js version and runtime requirements
   - Validate rate limiting and retry strategies against actual implementation

### 11.6 Wiki Discrepancies

**Status**: No wiki pages provided in source data

No comparison between WIKI and CODE was possible. When wiki documentation becomes available, verify:
-