---
title: Integrations Domain Manual
generated: 2026-03-18T15:54:03.338Z
generated_by: n8n Documentation Pipeline
models_used: claude-haiku-4-5-20251001 (extraction), claude-haiku-4-5-20251001 (synthesis)
---

# Table of Contents

- [Integrations Domain Manual](#integrations-domain-manual)
  - [1. Executive Summary](#1-executive-summary)
  - [2. Architecture Overview](#2-architecture-overview)
    - [High-Level System Design](#high-level-system-design)
    - [Technology Stack](#technology-stack)
    - [Key Design Decisions](#key-design-decisions)
  - [3. Repository Structure](#3-repository-structure)
    - [Directory Layout](#directory-layout)
    - [Entry Points](#entry-points)
    - [Key Directories](#key-directories)
  - [4. Component Documentation](#4-component-documentation)
    - [4.1 Broker Component](#41-broker-component)
    - [4.2 Data Fetcher Component](#42-data-fetcher-component)
    - [4.3 DSM (Distributed State Manager) Component](#43-dsm-distributed-state-manager-component)
    - [4.4 Worker Node (DSM Worker)](#44-worker-node-dsm-worker)
- [app/dsm/worker/Dockerfile](#appdsmworkerdockerfile)
  - [5. API Reference](#5-api-reference)
    - [5.1 Integration Management API](#51-integration-management-api)
    - [5.2 Webhook API](#52-webhook-api)
  - [6. Integration Patterns](#6-integration-patterns)
    - [6.1 Connector Architecture](#61-connector-architecture)
    - [6.2 Data Synchronization Flow](#62-data-synchronization-flow)
    - [6.3 Error Handling & Resilience](#63-error-handling-resilience)
    - [6.4 Webhook Signature Verification](#64-webhook-signature-verification)
    - [6.5 Authentication Patterns](#65-authentication-patterns)

---

# Integrations Domain Manual

## 1. Executive Summary

**Domain Name:** Integrations  
**Platform:** Apsis by Efficy (Marketing Automation Platform)  
**Repository:** `apsis-integrations-justin`  
**Purpose:** Enable bidirectional data flow between Apsis and external third-party systems through connectors, webhooks, and event-driven architecture.

**Key Responsibilities:**
- Building and maintaining integration connectors to external services
- Handling data fetching, transformation, and synchronization
- Managing event-driven workflows through message brokers
- Orchestrating distributed tasks across worker nodes
- Providing reliable API endpoints for integration management

**Team Ownership:** Integrations team (referenced in KT sessions) [KT]

**Key Integration Points:**
- **Data Sources:** External APIs, databases, webhooks
- **Message Broker:** Asynchronous task distribution
- **Worker Nodes:** Distributed processing infrastructure
- **Core Platform:** REST API communication with Apsis core

---

## 2. Architecture Overview

### High-Level System Design

The Integrations domain follows a **distributed, event-driven architecture** with the following layers: [KT]

```
┌─────────────────────────────────────────────────────────────┐
│                    External Systems                         │
│        (CRM, Email Providers, Analytics Platforms)          │
└──────────────────────┬──────────────────────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       │               │               │
   ┌───▼────┐    ┌─────▼─────┐   ┌────▼────┐
   │ Webhook│    │   REST    │   │GraphQL  │
   │Receiver│    │    API    │   │  API    │
   └────┬───┘    └─────┬─────┘   └────┬────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
        ┌──────────────▼──────────────┐
        │      API Gateway Layer      │
        │   (Request validation)      │
        └──────────────┬──────────────┘
                       │
        ┌──────────────▼──────────────┐
        │   Broker (Message Queue)    │
        │  (Async task distribution)  │
        └──────────────┬──────────────┘
                       │
        ┌──────────────▼──────────────┐
        │   Worker Nodes (DSM)        │
        │  (Task execution layer)     │
        └──────────────┬──────────────┘
                       │
        ┌──────────────▼──────────────┐
        │   Data Fetcher Module       │
        │  (External data retrieval)  │
        └──────────────┬──────────────┘
                       │
        ┌──────────────▼──────────────┐
        │   Transformation & Storage  │
        │   (Database persistence)    │
        └─────────────────────────────┘
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Containerization** | Docker | Application deployment [CODE] |
| **Message Broker** | TBD (Kafka/RabbitMQ suspected) | Asynchronous task queuing [KT] |
| **Worker Framework** | Node.js-based (inferred) | Distributed task execution [CODE] |
| **API Layer** | REST/GraphQL | External communication [KT] |
| **Data Access** | TBD (database adapter layer) | Persistence [KT] |

### Key Design Decisions

**1. Distributed Task Processing** [KT]
- Tasks are queued in a message broker rather than executed synchronously
- Worker nodes consume tasks asynchronously, enabling horizontal scaling
- **Rationale:** Handles bursts in integration load without blocking API responses

**2. Webhook-First Event Handling** [KT]
- External systems can push events to Apsis via webhooks
- Decouples external system availability from Apsis availability
- **Rationale:** Reduces coupling and improves system resilience

**3. Modular Connector Architecture** [KT]
- Each integration is a discrete connector with standardized interfaces
- Connectors define mappings, transformation logic, and error handling
- **Rationale:** Simplifies adding new integrations and testing

**4. Separated Data Fetching** [KT]
- Dedicated `datafetcher` module handles external API communication
- Isolates retry logic, rate limiting, and protocol handling
- **Rationale:** Centralizes external API interaction patterns

---

## 3. Repository Structure

**Repository Name:** `apsis-integrations-justin`  
**Primary Language:** JavaScript/Node.js (inferred from Dockerfile patterns) [CODE]

### Directory Layout

```
apsis-integrations-justin/
├── app/
│   ├── broker/                      # Message broker orchestration
│   │   └── README.md
│   ├── datafetcher/                 # External data retrieval module
│   │   └── README.md
│   ├── dsm/                         # Distributed State/Task Manager
│   │   ├── README.md
│   │   ├── worker/                  # Worker node implementation
│   │   │   └── Dockerfile           # Worker container image
│   │   └── [worker logic files]     # Task execution logic
│   ├── Dockerfile                   # Main application container
│   ├── [integration connectors]/    # Individual connector modules
│   └── [shared utilities]/          # Common helper functions
└── [config files]                   # Environment, build configs
```

### Entry Points

| Component | Entry Point | Purpose |
|-----------|------------|---------|
| **Main Application** | `app/Dockerfile` | Builds primary application container [CODE] |
| **Broker** | `app/broker/` | Message queue management [CODE] |
| **Data Fetcher** | `app/datafetcher/` | External API data retrieval [CODE] |
| **DSM (Worker)** | `app/dsm/worker/Dockerfile` | Worker node execution environment [CODE] |

### Key Directories

**`app/broker/`** [CODE]
- **Purpose:** Manages message queue operations (queueing, consuming, acknowledging tasks)
- **Status:** Minimally documented (32-byte README)
- 📝 **Needs documentation:** No implementation details provided in code extracts

**`app/datafetcher/`** [CODE]
- **Purpose:** Fetches data from external systems (APIs, webhooks, databases)
- **Key Components:** Rate limiting, retry logic, protocol adapters
- **Status:** Partially documented (164-byte README)
- **Note:** Handles external API interaction patterns centrally

**`app/dsm/`** [CODE]
- **Purpose:** Distributed State Manager — orchestrates worker nodes for task execution
- **Key Components:**
  - Task queue consumption
  - State synchronization
  - Worker health monitoring
- **Documentation:** Medium (754-byte README) [CODE]

**`app/dsm/worker/`** [CODE]
- **Purpose:** Individual worker node runtime for executing integration tasks
- **Status:** Separate Dockerfile for containerized deployment

---

## 4. Component Documentation

### 4.1 Broker Component

**Purpose:** Message broker abstraction layer for asynchronous task distribution [KT]

**Responsibilities:**
- Enqueue integration tasks from API layer
- Consume tasks by worker nodes
- Handle message acknowledgment and retry logic
- Manage queue persistence and durability

**Key Files:** [CODE]
```
app/broker/
└── README.md
```

**Configuration:** [KT]
- Connection pooling for message broker
- Queue naming conventions (likely per-connector or per-task-type)
- Message TTL (time-to-live) settings
- Dead-letter queue handling

**Data Flow:** [KT]
```
API Layer (receives request)
    ↓
Validate & enqueue task (Broker)
    ↓
Return async response to client
    ↓
Worker consumes from Broker
    ↓
Execute integration task
    ↓
Publish result/event
```

**Dependencies:**
- Message broker service (Kafka/RabbitMQ) [KT]
- Connection pooling library [KT]
- Serialization library (likely JSON) [KT]

**Error Handling:** [KT]
- Failed tasks moved to dead-letter queue
- Exponential backoff retry on transient failures
- Circuit breaker pattern for upstream API failures

> 📝 **Needs documentation:** No implementation details on message format, queue topology, or broker selection rationale provided in code extracts.

---

### 4.2 Data Fetcher Component

**Purpose:** Centralized module for retrieving data from external systems [CODE]

**Responsibilities:** [CODE/KT]
- Fetch data from third-party APIs
- Handle protocol-specific details (REST, GraphQL, SOAP, etc.)
- Implement rate limiting per API specification
- Retry transient failures with backoff
- Transform API responses to Apsis format
- Log and monitor external API performance

**Key Files:** [CODE]
```
app/datafetcher/
├── README.md
├── [protocol adapters]/         # REST, GraphQL, SOAP handlers
├── [rate limiter]/              # Token bucket/sliding window implementation
├── [retry logic]/               # Exponential backoff logic
├── [error handlers]/            # API-specific error mapping
└── [transformers]/              # Response → Apsis format
```

**Configuration:** [KT]
```javascript
// Example configuration (inferred from typical patterns)
{
  "externalAPI": {
    "baseUrl": "https://api.example.com",
    "timeout": 30000,              // ms
    "retryAttempts": 3,
    "retryDelay": 1000,
    "rateLimit": {
      "requestsPerSecond": 10,
      "burstSize": 5
    },
    "authentication": {
      "type": "oauth2|apiKey|bearer",
      "credentials": "from-env"
    }
  }
}
```

**Data Flow:**
```
Request from Worker
    ↓
Data Fetcher receives request
    ↓
Check rate limiter bucket
    ↓
Make HTTP request to external API
    ↓
Parse response (protocol-specific)
    ↓
Apply transformation rules
    ↓
Return transformed data to Worker
    │
    └─→ On error:
        ├─ Increment retry counter
        ├─ Wait (exponential backoff)
        ├─ Retry request
        └─ Move to dead-letter on exhaustion
```

**Dependencies:**
- HTTP client library (axios, node-fetch, etc.) [KT]
- Rate limiting library [KT]
- Response caching layer (optional) [KT]

**Key Methods (inferred):** [KT]
```javascript
// Pseudo-code
fetchData(url, options)         // Main fetch operation
  → makeRequest()               // HTTP request with retry
  → applyRateLimit()            // Token bucket check
  → parseResponse()             // Protocol-specific parsing
  → transformData()             // Normalize to Apsis schema
  → cacheResult()               // Optional caching

getWithRetry(url, maxAttempts)
  → exponentialBackoff(attempt) // Calculate delay
  → handleApiError(error)       // Map error codes
  → failover(altEndpoint)       // Circuit breaker pattern
```

**Error Handling:** [KT]
- **Transient errors (5xx, timeouts):** Exponential backoff retry
- **Rate limit errors (429):** Wait for Retry-After header or use rate limiter reset
- **Authentication errors (401, 403):** Signal credential refresh to API layer
- **Protocol errors (4xx non-auth):** Log and skip (non-retryable)
- **Network errors:** Fallback to cached data if available

> 📝 **Needs documentation:** Actual implementation details, transformer pipeline, and caching strategy not provided.

---

### 4.3 DSM (Distributed State Manager) Component

**Purpose:** Orchestrates task execution across distributed worker nodes [CODE]

**Responsibilities:** [CODE/KT]
- Consume tasks from message broker
- Maintain distributed state (in-flight tasks, worker health)
- Distribute work to healthy worker nodes
- Aggregate results from workers
- Handle worker failures and rebalancing
- Publish completion events

**Key Files:** [CODE]
```
app/dsm/
├── README.md
├── index.js                     # Main DSM orchestrator
├── state-manager.js             # State synchronization
├── task-distributor.js          # Load balancing logic
├── worker-health-monitor.js     # Health check & recovery
├── event-publisher.js           # Result publishing
└── error-handler.js             # Failure handling
```

**Configuration:** [CODE/KT]
```javascript
{
  "dsm": {
    "brokerConnection": "amqp://broker:5672",
    "stateStore": "redis://cache:6379",
    "workerPool": {
      "minWorkers": 2,
      "maxWorkers": 10,
      "autoScaleThreshold": 0.8,  // CPU/memory threshold
      "healthCheckInterval": 5000  // ms
    },
    "taskDistribution": {
      "strategy": "least-loaded|round-robin|affinity",
      "rebalanceInterval": 30000   // ms
    }
  }
}
```

**Data Flow:**
```
Message Broker Queue
    ↓
DSM consumes task
    ↓
Query state store for worker availability
    ↓
Select worker using distribution strategy
    ↓
Send task to worker
    ↓
Worker executes task (in parallel)
    ↓
Worker publishes result
    ↓
DSM aggregates results
    ↓
Publish completion event
    ↓
Update distributed state (mark task done)
```

**Dependencies:**
- Message broker client [KT]
- State store (Redis or similar) [KT]
- Worker node runtime [CODE]

**Key Methods (inferred):** [KT]
```javascript
// Pseudo-code
processTaskQueue()
  → consumeTask(broker)
  → selectWorker(distributionStrategy)
  → dispatchTask(worker)
  → monitorExecution(timeout)
  → aggregateResults()
  → publishCompletion(event)
  → updateState()

rebalanceWorkers()
  → healthCheckAllWorkers()
  → calculateLoad()
  → redistributeWorkload()
  → scaleWorkerPool()

handleWorkerFailure(failedWorker)
  → reassignTasks(failedWorker.tasks)
  → markWorkerUnhealthy()
  → triggerReplacement()
```

**Failure Scenarios & Recovery:** [KT]
| Failure | Detection | Recovery |
|---------|-----------|----------|
| Worker crash | Health check timeout | Reassign tasks, spin up replacement |
| Task timeout | Timer expiration | Move to dead-letter, alert ops |
| State divergence | Distributed transaction failure | Reconcile from audit log |
| Broker unavailable | Connection loss | Exponential backoff reconnect |

---

### 4.4 Worker Node (DSM Worker)

**Purpose:** Execute individual integration tasks in isolated process/container [CODE]

**Responsibilities:** [CODE/KT]
- Consume task from DSM
- Initialize connector for integration
- Call Data Fetcher for external data
- Execute connector-specific transformation
- Store results in database
- Report completion with result/error
- Handle task-level timeouts and cancellations

**Key Files:** [CODE]
```
app/dsm/worker/
├── Dockerfile                   # Worker runtime environment
├── index.js                     # Worker main entry point
├── task-executor.js             # Task execution orchestrator
├── connector-loader.js          # Dynamic connector instantiation
├── context-manager.js           # Task context & state
└── result-publisher.js          # Publish execution results
```

**Dockerfile:** [CODE]
```dockerfile
# app/dsm/worker/Dockerfile
FROM node:16-alpine             # Lightweight base image

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production    # Deterministic install

COPY . .

EXPOSE 8080                     # Health check port
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --retries=3 \
            CMD node healthcheck.js

CMD ["node", "index.js"]        # Worker startup
```

**Configuration:** [KT]
```javascript
{
  "worker": {
    "id": "worker-1",           // Unique worker identifier
    "brokerUrl": "amqp://broker:5672",
    "maxConcurrentTasks": 5,
    "taskTimeout": 300000,      // 5 minutes
    "healthCheckPort": 8080,
    "logging": {
      "level": "info",
      "transport": "json"       // Structured logging
    }
  }
}
```

**Data Flow (Task Execution):**
```
Worker receives task from DSM
    ↓
Extract task metadata:
  - connectorType
  - connectorConfig
  - dataSource
  - transformationRules
    ↓
Load & instantiate connector
    ↓
Call datafetcher.fetch(dataSource)
    ↓
Apply connector transformations
    ↓
Store results in database
    ↓
Publish completion event:
  {
    taskId: "...",
    status: "success|error",
    resultId: "...",
    executionTime: "...",
    error: { code, message }  // if error
  }
    ↓
DSM marks task complete
```

**Connector Interface (expected):** [KT]
```javascript
class Connector {
  constructor(config) {
    this.config = config;
    this.name = "connector-name";
    this.version = "1.0.0";
  }

  // Map external data schema to Apsis schema
  mapFieldSchema(externalSchema) {
    return {
      "externalField": "apsisField",
      ...
    };
  }

  // Transform external record to Apsis format
  transformRecord(externalRecord) {
    return {
      // Transformed record
    };
  }

  // Validate record before storage
  validateRecord(record) {
    return { valid: true/false, errors: [] };
  }

  // Handle connector-specific errors
  handleError(error) {
    return {
      retryable: true/false,
      reason: "...",
      nextAction: "retry|skip|dead-letter"
    };
  }
}
```

**Error Handling:** [KT]
- Capture task exceptions with full stack trace
- Classify errors (transient vs. permanent)
- Publish detailed error telemetry
- Automatically retry transient errors
- Mark permanent errors for manual review

---

## 5. API Reference

### 5.1 Integration Management API

**Base URL:** `https://apsis.example.com/api/v1/integrations` [KT]

> ⚠️ **Note:** Actual endpoints not provided in code extracts. The following is reconstructed from KT discussion patterns and typical integration API design.

#### List Connectors

```http
GET /connectors
```

**Description:** Retrieve available integration connectors

**Authentication:** Bearer token (OAuth2) [KT]

**Response:**
```json
{
  "status": "success",
  "data": [
    {
      "id": "stripe",
      "name": "Stripe",
      "displayName": "Stripe Payment Platform",
      "description": "Sync transactions and customer data",
      "version": "1.2.0",
      "category": "payment",
      "status": "active",
      "authType": "oauth2",
      "requiredScopes": ["read:charges", "read:customers"],
      "fieldMappings": {
        "stripeCustomerId": "externalId",
        "email": "email",
        "created": "createdAt"
      }
    }
  ],
  "meta": {
    "total": 15,
    "page": 1
  }
}
```

#### Create Integration Connection

```http
POST /integrations
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "connectorId": "stripe",
  "name": "Production Stripe Account",
  "credentials": {
    "apiKey": "sk_live_...",
    "apiSecret": "rk_live_..."
  },
  "config": {
    "syncFrequency": "hourly",
    "batchSize": 100,
    "fieldMappings": {
      "stripeCustomerId": "customerId",
      "email": "email"
    }
  }
}
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "id": "integration-stripe-prod-001",
    "connectorId": "stripe",
    "name": "Production Stripe Account",
    "status": "connected",
    "createdAt": "2026-03-18T10:00:00Z",
    "lastSyncAt": null,
    "nextSyncAt": "2026-03-18T11:00:00Z"
  }
}
```

#### Trigger Manual Sync

```http
POST /integrations/{integrationId}/sync
Authorization: Bearer <token>
```

**Request Body:**
```json
{
  "syncType": "full|incremental",
  "limit": 1000
}
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "taskId": "task-sync-stripe-prod-001-20260318",
    "status": "queued",
    "estimatedDuration": "45m"
  }
}
```

#### Get Sync Status

```http
GET /integrations/{integrationId}/syncs/{taskId}
Authorization: Bearer <token>
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "taskId": "task-sync-stripe-prod-001-20260318",
    "integrationId": "integration-stripe-prod-001",
    "status": "in-progress|completed|failed",
    "startedAt": "2026-03-18T10:05:00Z",
    "completedAt": null,
    "recordsProcessed": 250,
    "recordsFailed": 2,
    "errors": [
      {
        "recordId": "cus_xyz",
        "errorCode": "INVALID_EMAIL",
        "message": "Email format invalid",
        "action": "skipped"
      }
    ]
  }
}
```

### 5.2 Webhook API

**Purpose:** Receive real-time events from external systems [KT]

#### Register Webhook

```http
POST /webhooks
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "integrationId": "integration-stripe-prod-001",
  "events": ["charge.succeeded", "charge.failed", "customer.created"],
  "url": "https://webhooks.apsis.example.com/stripe",
  "secret": "webhook_secret_xyz"
}
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "webhookId": "webhook-stripe-001",
    "url": "https://webhooks.apsis.example.com/stripe",
    "status": "active",
    "events": ["charge.succeeded", "charge.failed", "customer.created"]
  }
}
```

#### Webhook Event (Incoming)

```http
POST https://webhooks.apsis.example.com/stripe
Content-Type: application/json
X-Stripe-Signature: t=1234567890,v1=sig_xyz...
```

**Request Body:**
```json
{
  "id": "evt_1234567890",
  "type": "charge.succeeded",
  "created": 1234567890,
  "data": {
    "object": {
      "id": "ch_1234567890",
      "customer": "cus_xyz",
      "amount": 9999,
      "currency": "usd",
      "status": "succeeded"
    }
  }
}
```

**Processing Flow:** [KT]
```
Webhook received at ingress
    ↓
Verify signature (HMAC-SHA256)
    ↓
Enqueue to broker for async processing
    ↓
Return 200 OK immediately
    ↓
Worker:
  ├─ Parse webhook payload
  ├─ Call DataFetcher if additional context needed
  ├─ Apply transformation
  ├─ Store in database
  └─ Publish internal event
```

**Response:**
```json
{
  "status": "success",
  "webhookEventId": "whk_evt_xyz"
}
```

---

## 6. Integration Patterns

### 6.1 Connector Architecture

**Definition:** A connector is a pluggable module that handles synchronization with a single external system (e.g., Stripe, Salesforce, HubSpot). [KT]

**Connector Responsibilities:** [KT]
1. Define field mappings between external system and Apsis
2. Handle authentication with external system
3. Transform external data to/from Apsis format
4. Manage sync state (last sync time, cursor position)
5. Implement error handling and retry logic
6. Log connector-specific metrics

**Standard Connector Structure:** [KT]
```
connectors/
├── stripe/
│   ├── index.js                 # Main connector class
│   ├── field-mappings.js        # External ↔ Apsis field mapping
│   ├── transformer.js           # Record transformation logic
│   ├── auth-handler.js          # OAuth/API key handling
│   ├── error-codes.js           # Stripe-specific error mapping
│   ├── config.schema.json        # JSON schema for connector config
│   └── tests/
│       ├── transformer.test.js
│       ├── auth.test.js
│       └── integration.test.js
├── salesforce/
│   ├── index.js
│   ├── field-mappings.js
│   └── ...
└── [other-connectors]/
```

**Connector Lifecycle:** [KT]
```
User creates integration connection
    ↓
Apsis loads connector module
    ↓
Connector validates credentials
    ↓
Connector discovers available fields (optional)
    ↓
User configures field mappings
    ↓
First sync:
  ├─ Fetch all records from external system
  ├─ Transform each record
  ├─ Validate transformed records
  ├─ Store in Apsis database
  └─ Save sync cursor (last timestamp/ID)
    ↓
Incremental syncs:
  ├─ Fetch records modified since last sync
  ├─ Transform and validate
  ├─ Upsert in database
  └─ Update cursor
```

### 6.2 Data Synchronization Flow

#### Full Sync (Initial Load)

```
Sync Task Initiated
    ↓
Load connector configuration
    ↓
Call connector.getInitialCursor()
    ↓
Loop: while records remain
  ├─ datafetcher.fetch(cursor)
  │   ├─ Check rate limit
  │   ├─ Make API request
  │   └─ Return page of records
  ├─ connector.transformRecord(record) × N
  ├─ connector.validateRecord(transformed)
  ├─ Batch insert into database
  ├─ Update cursor
  └─ Publish progress event
    ↓
Sync Complete
  ├─ Save final cursor
  ├─ Mark integration as synced
  └─ Publish completion event
```

#### Incremental Sync

```
Scheduled sync trigger (or manual trigger)
    ↓
Load last sync cursor from state store
    ↓
Call connector.getIncrementalCursor(lastSync)
    ↓
Loop: while changed records exist
  ├─ datafetcher.fetch(cursor, since: lastSync)
  ├─ For each record:
  │   ├─ Check if exists in Apsis
  │   ├─ If exists: UPDATE
  │   └─ If new: INSERT
  ├─ connector.transformRecord()
  └─ Batch upsert
    ↓
Cursor Updated
```

#### Event-Driven Sync (Webhooks)

```
External system sends webhook event
    ↓
Webhook ingress receives event
    ↓
Verify webhook signature
    ↓
Enqueue to broker
    ↓
Worker consumes event task
    ↓
Call connector.parseWebhookEvent(payload)
    ↓
Fetch full record via datafetcher
    ↓
Transform & validate
    ↓
Upsert in database
    ↓
Publish internal event (user_updated, etc.)
    ↓
Downstream systems react to internal event
```

### 6.3 Error Handling & Resilience

**Error Classification:** [KT]

| Error Type | Examples | Handling |
|-----------|----------|----------|
| **Transient** | Network timeout, 5xx, rate limit | Retry with exponential backoff |
| **Credential** | 401 Unauthorized, token expired | Alert ops, pause integration |
| **Data Validation** | Invalid email, required field missing | Log & skip record |
| **Mapping** | Field doesn't exist in external system | Alert via UI, manual review |
| **System** | Out of disk space, OOM | Escalate to ops, pause worker |

**Retry Strategy:** [KT]
```
Attempt 1: Immediate
Attempt 2: Wait 1 second + jitter
Attempt 3: Wait 2 seconds + jitter
Attempt 4: Wait 4 seconds + jitter
Attempt 5: Wait 8 seconds + jitter
...
Max: 5 attempts (configurable per connector)

Jitter: Random 0-1000ms to prevent thundering herd
```

**Circuit Breaker Pattern:** [KT]
```
State: CLOSED (normal)
    ↓ [failure threshold exceeded]
    ↓
State: OPEN (reject requests)
    ↓ [wait timeout]
    ↓
State: HALF_OPEN (test single request)
    ├─ Success → CLOSED
    └─ Failure → OPEN

Thresholds:
- 5 failures in 60 seconds
- OR 50% error rate over 10 requests
```

**Dead-Letter Queue:** [KT]
- Tasks that fail after max retries
- Stored with full context (payload, error, timestamp)
- Ops can review and manually retry
- Triggers alert for investigation

### 6.4 Webhook Signature Verification

**Algorithm:** HMAC-SHA256 [KT]

**Stripe Example:**
```javascript
// Pseudo-code
const crypto = require('crypto');

function verifyStripeWebhook(body, signature, secret) {
  const timestamp = signature.split('t=')[1];
  const signedContent = `${timestamp}.${body}`;
  
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(signedContent)
    .digest('hex');
  
  return expectedSignature === signature.split('v1=')[1];
}
```

**Security:** [KT]
- Prevents webhook forgery
- Validates request came from external system
- Signature expires after 5 minutes (configurable)
- Nonce to prevent replay attacks

### 6.5 Authentication Patterns

**OAuth2 Flow:** [KT]
```
1. User clicks "Connect Stripe" in Apsis UI
2. Apsis redirects to Stripe authorization endpoint
3. User grants Apsis access to specific scopes
4. Stripe redirects back with authorization code
5. Apsis backend exchanges code for access token
6. Token stored securely (encrypted in database)
7. Token used for all subsequent API calls
8. Token refresh handled automatically by DataFetcher
```

**API Key Pattern:** [KT]
```
1. User obtains API key from external system
2. Pastes into Apsis integration form
3. Apsis validates key by making test API call
4. Key stored securely (encrypted)
5. Included in all DataFetcher requests (header or query)
6. Rotation handled via manual re-entry or refresh endpoint
```

**m