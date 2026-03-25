---
source_file: Erik - Outbound flow.txt
domain: Apsis One Integrations
topics: [Outbound Flow Architecture, Event and Consent Synchronization, Message Batching, CRM Integration, Generic Connector, Credential Management, Proxy Architecture]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Outbound Manager, Audience Subscription Worker, Kafka, Batch Production Worker, Outbound Worker, Broker Service, Squid Proxy, Generic Connector, Integration Manager]
session_type: knowledge-transfer
subdomains: [Architecture, Outbound Flow, Generic Connector, Different Types of Connectors]
---

## Session Overview

This session covered the **outbound flow** of the Apsis One Integrations platform—the process of sending data from Apsis to external CRM systems. The discussion focused on how consent updates and activity events (email sends, form submissions, event tool interactions) are synchronized from Apsis to CRM systems through a multi-stage pipeline involving message queuing (SNS/SQS, Kafka), batching, authentication, and security controls. Erik outlined the architectural components, data transformations, and the role of outbound mappings that enable CRM systems to understand and act on the data Apsis sends them. The session identified that lead creation and Microsoft Dynamics Azure integration would require separate, deeper sessions.

---

## Scope and Planning for Related Sessions

Erik clarified that this session covers only the **initial outbound flow**—pushing data from Apsis audiences to CRM systems without handling the responses. Two additional sessions were planned:

1. **Lead Creation/Lead Gathering Session**: This involves the reciprocal communication pattern ("tango with the CRM") where the CRM responds to events Apsis sends, potentially creating new records. Lead creation involves important internal logic around **merges** and duplicate profile handling.

2. **Microsoft Dynamics and Azure Session**: Required because Dynamics integrations utilize Azure resources (Microsoft Entra for OAuth, Power Platform for managing application users). This session will cover:
   - Finding and managing credentials in Azure
   - User permissions and enablement
   - Application user setup across test and production environments
   - How Azure authentication directly affects customer instance connectivity for debugging

[Erik Andersson]: The scope decision was made because these topics are sufficiently complex and distinct to warrant dedicated deep-dives rather than rushed coverage.

---

## Core Outbound Flow: From Audience to CRM

### Two Primary Data Types Sent from Apsis to CRM

The outbound flow synchronizes **two categories of data**:

1. **Consent**: Must remain consistent between Apsis and the external CRM system.
2. **Events**: Activity records from Apsis, including form submissions, email delivery/open/click events, event tool interactions, etc.

### The Event Lifecycle

When a user creates an activity in Apsis (e.g., an email campaign or event tool entry) and configures it to sync to a CRM system, a **sync checkbox** appears. Upon activity creation, Apsis sends a request to the integration containing the activity ID. The outbound flow then:

- Registers event listeners for that specific activity in Audience (the publishing system)
- Monitors for all subsequent events on that activity
- Routes them through the outbound pipeline

[Erik Andersson]: The key design principle is **specificity**: we don't listen to all events on an account; we only listen to events for activities explicitly flagged for sync, and for consents, only for subscriptions with active mappings.

---

## Message Entry Point: SNS Topic and Audience Subscription Queue

### SNS Topic and Message Filtering

All outbound traffic from Audience is published to a single **SNS (Simple Notification Service) topic** within the integration. This topic acts as a universal entry point:

- Every type of message (email, SMS, form, event tool, consent) is dumped into this topic
- The topic immediately routes all messages to a single SQS queue called **`aud_sub_queue`** (audience subscription queue)

### The Audience Subscription Worker

The **audience subscription worker** consumes messages from `aud_sub_queue` and performs critical validation and filtering:

#### Validation Steps

1. **Data Presence Check**: Verifies that all required data is present for the event/consent.

2. **CRM ID Requirement**: Almost always requires the profile to have a CRM ID. This is critical:
   > "We should not send consent updates to the CRM for profiles that have no CRM ID because that means that these profiles did not come from the CRM."
   
   **Real-World Example**: If a file import adds 60,000 new profiles with consent enabled on a subscribed topic but without CRM IDs, the worker will discard all 60,000 consent updates. These are local-only profiles, not records synced from the external system.

3. **Installation Verification**: Confirms that an active installation exists for the account/section combination. Discards messages if:
   - The installation has been removed
   - There was a temporary outage and event listeners were not properly cleaned up
   - Messages are for an outdated installation

4. **Feature Release Gate**: Verifies that the feature for syncing the specific activity type has been released to production. Example: if lead creation (survey sync) is in development and the listening code accidentally reaches production, this gate catches it before messages clutter the queues.

#### Performance Implications of File Import

File imports represent the **most problematic scenario** for the worker because:

- Volumes can be massive (400,000+ profiles in documented incidents)
- Data quality issues are common and unavoidable
- CRM ID format corruption occurs frequently (e.g., Excel formatting large integers as floats with commas, converting integers to strings)

[Erik Andersson]: "Not with 2 million, but I had a customer that imported 400,000 profiles that with a incorrect CRM ID. And yeah, if we put it like that the I mean we have, we do have auto scaling on everything in the outbound. No, but yes, the worker was put under heavy load."

---

## Auto-Scaling and Load Handling

### Scaling Strategy

- **Metric**: CPU usage triggers scaling (not memory)
- **Technology**: ECS-based auto-scaling; additional tasks are added/removed based on thresholds
- **Limitation**: Scaling takes time; it is not instantaneous

### Load Characteristics

Processing is grouped by: **account + section + integration ID + event type**

- A single large email send (500,000 recipients) generates 500,000 sent events, 500,000 delivered events, plus click/open events—potentially millions of messages
- Form submissions generate far fewer events and create less dramatic spikes
- The bottleneck is predictable to specific customers; their traffic gets throttled

### SLA and Expectations

The outbound flow is fully **asynchronous**. Under normal conditions, messages move from Audience to the CRM in seconds. However, customers are told to expect:

> "You can easily expect it to take like up to 5 minutes before it is in the CRM."

This buffer accommodates auto-scaling delays and ensures resilience.

### Retry and Backoff Mechanism

A common library handles message failures across the entire Justin platform:

- Messages are placed back on the SQS queue with a **backoff delay**
- The delay duration varies depending on the service and failure type
- Retried messages are re-processed when the system has capacity

---

## Message Transformation: Event Structure

### Input vs. Output Message Format

After validation, the worker transforms raw Audience events into a standardized structure:

```
profile_key: <Apsis profile identifier>
event_type: <email|sms|form|event_tool|consent>
activity_id: <internal activity ID>
correlation_id: <correlation ID from Audience>
source: <source system identifier>
event_time: <timestamp>
event_data: <raw event data from Audience>
profile_fields: <identifying fields: CRM ID, email, phone>
mapped_fields: <mapped custom fields>
```

#### Profile Fields

These are **static identifying fields** used by the CRM to locate records:

- CRM ID (primary identifier for existing contacts)
- Email address
- Phone number

The worker includes whichever of these three are present on the profile.

#### Event Data

Raw event payload from Audience. Examples:

- **Event Tool**: `score` property and other event-specific attributes
- **Form Submission**: all form field values and responses

---

## Kafka: Message Queue for Batching Consumers

### Why Kafka Instead of SQS?

1. **Removes SQS limitations**: SQS has architectural constraints that Kafka overcomes
2. **Easier consumer connectivity**: Multiple independent consumers can subscribe to the same topic without competing for messages
3. **Topic-based organization**: Each activity type gets its own Kafka topic (separate queue for email events, form events, SMS events, etc.)

### Operational Aspects

**Maintenance Burden**: 
- Kafka is an Apache project (not a native AWS service); Apsis uses AWS's managed Kafka service
- Patches/updates are released by AWS and require clicking "apply update"
- **Frequency**: Approximately twice per year (every 6 months)
- **Incident History**: [Erik Andersson] reports zero significant issues in years of operation beyond applying updates

**Configuration**:
- More complex than SQS setup but manageable
- Configured in CloudFormation files
- Sets up connectivity, security groups, and access controls
- Deployed as a shared resource before any microservices

**Developer Perspective**:
- For maintenance and development, understanding that Kafka is "another type of message queue" that is more powerful than SQS is sufficient for most scenarios
- Detailed Kafka internals and configuration will be covered separately if needed

### Deployment Order

The integration infrastructure deploys in layers:

1. **Shared cloud formation resources** (deployed first):
   - Kafka instance
   - SNS topic
   - SQS queues
   - ECR repositories
   - Parameter store configuration values
   - Security groups

2. **Microservices** (deployed after shared resources exist)

---

## Batch Production: Aggregating Events Before CRM Delivery

### Why Batching?

Raw events arrive one-at-a-time from Kafka (e.g., for a 500-recipient email: one sent event, then another sent event, then another...). Sending these individually to the CRM would incur:

- Excessive handshaking overhead
- Unnecessary API authentication calls
- Poor network efficiency

### Batch Production Worker Architecture

There are **six Batch Production Workers**, each running the same code but listening to different Kafka topics:

- `batch_production_email`
- `batch_production_sms`
- `batch_production_form`
- `batch_production_event_tool`
- `batch_production_ma_flow`
- `batch_production_consent`

All run identical Docker images; the only difference is which Kafka topic they consume from.

### Batching Logic

For each Kafka topic, the worker:

1. **Reads events continuously** from the topic
2. **Groups by installation**: Creates separate in-memory batches for each account/section/integration combination
   - Example: If account A has a Dynamics installation and account B has an E-deal installation, two separate batches are maintained
3. **Generates a unique batch ID** for tracking and debugging
4. **Flushes the batch** when either condition is met:
   - **Size limit reached**: 200 KB per batch (SQS has a 256 KB message limit, so 200 KB provides safety margin)
   - **Time limit reached**: 30 seconds with no new events for this specific installation

### Batch Output

After flushing, the batch is sent to the **Outbound Worker Queue** (another SQS queue), containing:

- Batch ID
- Integration ID
- Section ID (account identifier)
- Account ID
- List of profiles with their events grouped together

#### Example Batch Structure

For a form submission batched with other events:

```json
{
  "batch_id": "<generated UUID>",
  "integration_id": "<installation ID>",
  "section_id": "<section ID>",
  "account_id": "<account ID>",
  "batch_type": "form",
  "profile_data": [
    {
      "profile_key": "<Apsis profile ID>",
      "fields": {
        "email": "user@example.com",
        "crm_id": "Contact-12345"
      },
      "events": [
        {
          "event_type": "form_submit",
          "event_data": {
            "field1": "value1",
            "field2": "value2"
          },
          "correlation_id": "..."
        }
      ]
    }
  ]
}
```

---

## Outbound Worker: Final CRM Delivery

The **outbound worker** performs the final delivery to external systems:

### Core Responsibility

1. Reads batches from the Outbound Worker Queue
2. Determines the target CRM system (from integration ID)
3. Identifies the event type (email, SMS, form, event tool, consent)
4. Routes to the appropriate CRM endpoint

### Logging and Debugging

All batches are logged in **CloudWatch** with their batch ID, allowing support to trace:

- Whether the batch was sent
- Success or failure status
- The exact contents of the batch (via the batch ID)

---

## Consent Handling: Special Fast-Path Processing

Unlike events, consent updates **bypass Kafka and batching**:

```
Audience Subscription Worker -> Outbound Worker (direct, no batching)
```

**Rationale**: Consent changes sometimes need rapid propagation to the CRM. Rather than waiting for the batch window (up to 30 seconds) or size threshold, consent goes directly to its destination.

### Consent Request Structure

```json
{
  "account_id": "<account ID>",
  "section_id": "<section ID>",
  "integration_id": "<installation ID>",
  "subscription_id": "<subscription/topic in Apsis>",
  "consent_type": "<email|sms>",
  "consent_status": "<opted_in|opted_out>",
  "source": "<file_import|ui|api|etc>",
  "last_updated": "<timestamp>"
}
```

#### Source Tracking Value

The `source` field allows debugging of consent import failures. Example:

- Batch fails with `404 Contact Not Found`
- Support checks the source field
- If source is `file_import`, they can investigate CRM ID data quality in the import
- Likely root cause: incorrect CRM ID format in the imported file

---

## Outbound Mappings and Mapped Fields

### The Problem They Solve

In the initial implementation of lead gathering (form submissions leading to CRM contact creation), the CRM system was dependent on field names:

- Form field named `first_name` → assumed to map to CRM first name
- Form field named `email` → assumed to map to CRM email
- Form field named `custom_color` → unmappable; CRM doesn't know what to do with it

This approach was fragile and unreliable.

### The Solution: Two-Stage Mapping

**Stage 1** (Form Tool): Map form field names to Apsis attributes
```
Form field "Name" -> Apsis attribute "first_name"
Form field "Email Address" -> Apsis attribute "email"
Form field "Favorite Color" -> Apsis attribute "custom_color"
```

**Stage 2** (Integration Outbound Mappings): Map Apsis attributes to CRM fields
```
Apsis "email" -> E-deal "per_mail" (person mail)
Apsis "phone" -> E-deal "per_phone" (person phone)
```

### How Mapped Fields Are Used

When the outbound worker sends a form submission to the CRM, it includes `mapped_fields`:

```json
{
  "mapped_fields": {
    "per_mail": "user@example.com",
    "per_phone": "+1-555-0123"
  }
}
```

If the CRM decides to create a new contact, it uses these explicitly mapped fields:

> "If you decide to create something new from this, you should create a new silhouette which is like a type of entity inside the FC corporate. You should populate the seal_mail attribute with this attribute and you should populate the silhouette mobile field with this phone number."

### Critical Design Principle

**Apsis never creates records in the CRM.** The outbound flow delivers events and data. The CRM system's developers decide whether to:

- Create a new record
- Merge with an existing record
- Ignore the data entirely

Different CRM implementations have different logic:

- Some always create new records and have humans review them later
- Some have fully automated logic that validates and creates/rejects based on data quality

---

## Mapping Field Names: Static Contracts Between Systems

### Generic Connector Field Mapping

For Generic Connector integrations, the field names are **static and contractual** between Apsis and the CRM:

**Example: E-deal (Efficy Corporate)**
```
per_mail = email field
per_phone = phone field
person_id = CRM ID field
```

**Example: Efficy Enterprise 12.1**
```
emailaddress1 = email field
telephone1 = phone field
contactid = CRM ID field
```

The field names vary by CRM, but they are **not customer-configurable**. They represent the logical names of fields in the CRM's data model.

### Identifying Fields vs. Mapped Fields

The `fields` property in a batch contains **static identifying fields** that all CRM systems understand:
- CRM ID
- Email
- Phone number

The `mapped_fields` property is built from the customer's configured outbound mappings and can include any custom attributes the form captured.

---

## Generic Connector Interface

### Purpose

The Generic Connector defines a **standardized API contract** that all supported CRM systems implement. This allows Apsis to communicate with multiple diverse CRM platforms using a unified interface.

### Endpoints

Common Generic Connector endpoints include:

```
GET /v1/{entity}/schema
POST /v1/consents/{entity}
POST /v1/events/{entity}
GET /v1/contacts/schema
```

The `{entity}` placeholder is the logical name of the entity in that CRM (e.g., `contact`, `silhouette`, `person`).

### Feature Declaration

CRM systems declare which features they support (defined in a Swagger/OpenAPI specification):

- Email campaign sync
- SMS campaign sync
- Form sync
- Event tool sync
- Outbound mappings
- Consent management

Based on declared features, Apsis calls only the endpoints the CRM has implemented.

### Versioning and Breaking Changes

The Generic Connector API is currently on **v1**. To maintain backward compatibility:

- New properties can be added without version change (CRMs use loose schema validation and ignore unknown properties)
- Removing or significantly changing existing properties would require a **v2** endpoint
- Any breaking change must be communicated to all integrating partners via their internal channels

---

## Authentication and Security: Broker and Squid Proxy

### The Broker Service: Credential Management

**Purpose**: Secure credential handling for CRM API authentication

**Historical Context**: Originally, Apsis stored customer API credentials in clear text in the database (encrypted at rest, but still readable by developers). This was beneficial for debugging but violated security best practices.

**Current Design**:

1. Customer API credentials are **hashed** before storage in the database
2. The **Broker service** retrieves credentials from the database for a specific installation
3. Broker **decrypts** the credentials using KMS (Key Management Service)
4. Broker adds the decrypted credentials to the Authorization header
5. The request is forwarded to the next stage

**Result**: Developers cannot read actual customer credentials from the database; they only see that authentication succeeded or failed.

**Scope**: Broker is "a very stupid service"—it has one responsibility: decrypt and attach credentials. No business logic.

### Squid Proxy: Domain Whitelisting and Security

**Purpose**: Prevent requests to unauthorized external domains

**How It Works**:

1. Customer specifies their CRM domain during installation (e.g., `https://mycompany.mycrm.com`)
2. The domain is added to a **whitelisted domains table** in the database
3. Squid Proxy validates every outbound request:
   - Is the target domain in the whitelist?
   - If yes: allow the request
   - If no: reject the request

**Static Whitelists**: Some domains are always required:
- Microsoft authentication servers (for Dynamics)
- Other service domains that integrations call as part of their normal operation

**Configuration**: Handled via HTTPS proxy environment variables; the HTTP client library automatically routes traffic through the proxy without explicit business logic.

### Request Flow Summary

```
Outbound Worker
    ↓
Broker Service (decrypt credentials, add auth header)
    ↓
Squid Proxy (validate target domain)
    ↓
External CRM System
```

Both Broker and Squid Proxy are used by:
- Outbound Worker (for sending events and consent)
- Integration Manager (for inbound schema retrieval and field mapping)
- Any other service that needs to reach the external CRM

---

## Data Flow Diagram: Complete Outbound Journey

```
Audience (publishing system)
    ↓
SNS Topic (audience events topic)
    ↓
SQS Queue (aud_sub_queue)
    ↓
Audience Subscription Worker (validation, filtering)
    ├─ Consent → direct to Outbound Worker
    └─ Events → Kafka
        ↓
    Kafka Topics (one per event type)
    │   ├─ email_events
    │   ├─ sms_events
    │   ├─ form_events
    │   ├─ event_tool_events
    │   └─ consent_events (actually bypasses here)
        ↓
    Batch Production Workers (aggregate by installation)
        ↓
    SQS Queue (outbound_worker_queue)
        ↓
    Outbound Worker
        ├─ Consent Endpoint: /v1/consents/{entity}
        └─ Event Endpoints: /v1/events/{entity}
            ↓
        Broker Service (authentication)
            ↓
        Squid Proxy (domain validation)
            ↓
        External CRM System
```

---

## Documentation and Knowledge Resources

### Available References

1. **Wiki Pages** (internal, linked in integration team Slack channel):
   - Squid Proxy (comprehensive, up-to-date)
   - Lime Connector (detailed, current)
   - Other connectors (some outdated; will be refreshed during knowledge transfer)

2. **Generic Connector API Specification** (Swagger/OpenAPI):
   - Authoritative source of truth for CRM integration contract
   - Defines all endpoints, request formats, response formats
   - Specifies which features tie to which endpoints

3. **Troubleshooting and Incident Documentation** (in progress):
   - Log group naming conventions and how to find them in CloudWatch
   - Common errors and their resolutions
   - Troubleshooting decision trees
   - Being enriched continuously as incidents are resolved

### Recommended Learning Path

[Erik Andersson]: Rather than diving into wiki pages immediately, priority should be:

1. **Golang fundamentals** (programming language for all Justin services):
   - Syntax and language paradigm
   - Static typing and pointers
   - Common data structures (lists, maps)
   - Basic I/O and iteration patterns
   - This is the biggest hurdle for newcomers

2. **Service-to-service communication patterns** (after basic Go)

3. **Specific system deep-dives** (Squid Proxy, Generic Connector, etc.) as needed for specific work

---

## Key Takeaways

1. **Outbound flow is unidirectional from Apsis to CRM**: Apsis never creates CRM records; it only delivers events and consent updates. The CRM system decides what to do with the data.

2. **Specificity at every stage**: Event listeners are registered per-activity (not account-wide); consent listeners are per-subscription. This prevents unnecessary message volume.

3. **Validation gates protect quality**: The Audience Subscription Worker screens out invalid data (missing CRM IDs, inactive installations, unreleased features) before processing.

4. **Batching improves efficiency**: Events are grouped (up to 200 KB or 30 seconds) to reduce API overhead, but consent updates bypass batching for rapid propagation.

5. **Kafka enables topic-based routing**: Six independent batch workers listen to separate event-type topics, scaling independently.

6. **Security is layered**: Credentials are hashed in the database, decrypted only when needed by Broker, and all outbound requests pass through domain-validation Squid Proxy.

7. **Outbound mappings enable field translation**: Two-stage mapping (form field → Apsis attribute → CRM field) allows the CRM system to understand which data should populate which fields when creating new records.

8. **Generic Connector standardizes multi-CRM integration**: Rather than building distinct connectors for each CRM, a standardized API contract (with declared feature support) allows Apsis to communicate with diverse systems through a unified interface.

9. **File imports are the worst case for performance**: Large volumes, data quality issues, and format corruption make file imports the primary load driver on the outbound worker.

10. **Asynchronous with expected delays**: Under normal conditions, seconds. Realistically, customers should expect up to 5 minutes as the system scales and processes.

---

## Unresolved Questions and Action Items

### Sessions to Schedule

1. **Lead Creation/Lead Gathering**: Handle CRM responses, merges, duplicate profile management
2. **Microsoft Dynamics and Azure**: Credential management, Entra integration, Power Platform user management, test/prod environment setup
3. **Generic Connector Deep-Dive**: API spec walkthrough, feature declarations, versioning strategy
4. **Squid Proxy and Broker**: Detailed architecture, configuration, incident response
5. **Cloudwatch and Logging**: How to find and interpret logs across Justin services

### Documentation to Complete

1. Update wiki pages for accuracy (some are outdated)
2. Continue enriching troubleshooting/incident documentation
3. Consolidate diagrams and presentations to shared knowledge transfer folder
4. Create Golang learning resources or curated links for newcomers

### Immediate Homework for Team

1. Establish shared folder structure (diagrams, presentations, transcripts)
2. Review and update wiki pages for accuracy
3. Begin Golang fundamentals learning (higher priority than deep-diving into systems)

---

## Appendix: Terminology

- **Audience**: Apsis's internal publishing/event system
- **SNS/SQS**: AWS messaging services (SNS = pub/sub topic, SQS = queue)
- **Kafka**: Message queue with topics; supports multiple independent consumers
- **Batch Production Worker**: Aggregates individual events into larger batches
- **Generic Connector**: Standardized API contract that all supported CRM systems implement
- **CRM ID**: The unique identifier of a contact/person in the external CRM system
- **Outbound Mapping**: Configuration that maps Apsis attributes to CRM fields
- **Broker**: Service that decrypts and attaches authentication credentials
- **Squid Proxy**: Reverse proxy that validates destination domains against a whitelist
- **ECS**: Elastic Container Service (AWS service for running containerized applications)
- **Silhouette**: Entity type in Efficy Corporate (equivalent to "contact" in other systems)
- **Consent Basis/Consent List**: A named collection of consent records; Apsis subscriptions map to these