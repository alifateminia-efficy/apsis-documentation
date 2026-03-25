---
source_file: Erik - Outbound flow.txt
domain: Apsis One Integrations
topics: 
  - Outbound data flow architecture
  - Consent synchronization to CRM systems
  - Event publishing and batching
  - Message queue infrastructure (SNS, SQS, Kafka)
  - Profile identification and CRM mapping
  - Generic connector interface
  - API authentication and credential management
  - Outbound field mappings
speakers: 
  - Erik Andersson (Product Developer/Architect)
  - Lukasz Grabowski (Knowledge Transfer Lead)
  - Michal Rosikiewicz (Team Member)
  - Tomasz Kowalski (Team Member)
key_components: 
  - Audience (APSIS core system)
  - SNS Topic (SMS topic)
  - SQS Queues (audience subscription queue, outbound worker queue)
  - Kafka (event streaming)
  - Audience Subscription Worker
  - Batch Production Worker
  - Outbound Worker
  - Outbound Manager
  - Broker Service
  - Squid Proxy
  - Generic Connector
  - Integration Manager
session_type: knowledge-transfer
subdomains:
  - Architecture
  - Microsoft Dynamics Integration
  - Efficy Enterprise 12.0 Integration
  - Efficy Enterprise 12.1 Integration
  - E-deal Integration
---

## Session Overview

This knowledge transfer session covers the **outbound flow** of the Apsis One Integrations platform—the process of sending data from Apsis to external CRM systems. The session explains how consent updates and activity events (email sends, form submissions, event tool activities, etc.) are captured, validated, batched, and delivered to CRM systems via a multi-stage microservice pipeline. Key topics include message queue architecture (SNS, SQS, Kafka), the role of profile identification, field mappings, and the broker/squid proxy layer for secure API communication. The session establishes that a separate knowledge transfer session will be needed to cover **lead creation** (the inbound response handling from CRM systems) and **Microsoft Dynamics/Azure** specifics.

---

## Scope and Session Planning

Erik and Lukasz establish that the outbound flow will be covered in this session, while two related but distinct topics require separate sessions:

1. **Lead Creation/Lead Gathering** – This covers the second part of the outbound flow where APSIS receives responses from the CRM (acknowledgments that contacts were created or matched). Erik emphasizes this requires understanding of **profile merges**, which is critical domain knowledge.

2. **Microsoft Dynamics and Azure** – A dedicated session needed because Dynamics integrations utilize Azure resources, Microsoft Entra (for OAuth flows), and Power Platform for managing application users across test and production environments.

Erik notes: > "The second part of this is the lead creation because that's when we start having like a tango with the CRM that we say something, they say something, we do something and then we do something more internally."

---

## High-Level Outbound Flow Architecture

### What Gets Sent to CRM Systems

APSIS sends two primary data types to CRM systems:

1. **Consent** – Subscription/topic consent status (opt-in, opt-out) must remain synchronized between Apsis and the CRM
2. **Events** – Activity events from Apsis tools:
   - Email events (sent, delivered, opened, clicked)
   - SMS events
   - Form submissions
   - Event tool activities
   - Other tools (MA flow, etc.)

### Example: Event Sync Flow

When a user creates an event in the Apsis event tool and marks it to sync to a connected CRM system (e.g., E-deal), a sync checkbox becomes available. Upon event creation, Apsis sends an initial request to the CRM containing:
- The activity ID
- Profile information
- Event metadata

The CRM may then respond with information about new leads or matching existing contacts (handled in the lead creation session).

---

## Message Flow: From Audience to the CRM System

### Step 1: Event Publishing via SNS Topic

All activity events from Apsis tools (email, SMS, form, event tool, consent) are published to a **single SNS topic** in the integration platform called the **SMS topic**. 

[Erik Andersson]: > "Everything that audience sends to us is sent to this SMS topic. We have one SMS topic inside integration where audience sends everything to and then we filter like what type of message is this?"

The SNS topic immediately dumps all messages into an **SQS queue** called the **audience subscription queue** (aud_sub_queue).

### Step 2: Audience Subscription Worker – Initial Validation

The **audience subscription worker** reads messages from the SQS queue and performs critical filtering and validation:

#### Message Type Filtering
Determines if the incoming message is:
- Email event
- SMS event
- Form submission
- Event tool activity
- Consent update
- Other activity type

#### Validation Checks

1. **Profile CRM ID Check** – Almost all event types require the profile to have a CRM ID. If a profile lacks a CRM ID, the message is discarded.

   **Scenario**: File import adds 60,000 profiles with consent on a tracked topic, but none have a CRM ID assigned. These profiles did not originate from the CRM, so consent updates are not sent for them.

2. **Installation Verification** – Confirms that:
   - An installation exists for this account/section
   - The installation is not outdated (e.g., a listener wasn't properly removed during a failed uninstall)

3. **Feature Release Check** – Verifies that the specific activity sync type has been released to production. For example, if survey sync support is still in development and the listening code is accidentally released, the worker discards these messages to prevent queue clutter.

#### Message Reconstruction

Once validated, the audience subscription worker rebuilds the message into a standardized internal format:

```
{
  "profileKey": <apsis_profile_key>,
  "eventType": "email|sms|form|event_tool|consent",
  "activityId": <activity_id>,
  "correlationId": <correlation_id_from_audience>,
  "source": <source_identifier>,
  "eventTime": <timestamp>,
  "eventData": <raw_event_data>,
  "profileFields": {
    "crmId": <value_if_present>,
    "email": <value_if_present>,
    "phone": <value_if_present>
  },
  "mappedFields": <to_be_explained_later>
}
```

**Profile Fields**: These are identifying fields used by the CRM to locate contacts. They include:
- CRM ID (primary identifier)
- Email address
- Phone number
- Any other profile attributes the CRM recognizes

If the profile has a CRM ID, email, or phone, these are explicitly included in `profileFields` so the CRM knows how to identify the contact.

---

## Message Queue Infrastructure: From SQS to Kafka

### Why Kafka Instead of SQS?

After validation in the audience subscription worker, messages are sent to **Kafka** rather than continuing through SQS. 

[Erik Andersson]: > "One of the reasons why we have Kafka here instead of SQS is that you can remove some annoying limitations that SQS have. And also it's a bit easier to connect consumers to each of the different topics."

**Key differences from SQS perspective**:
- **Maintenance**: Kafka requires periodic updates (AWS patches released typically every 6 months), applied via an update button in the AWS console. So far, no significant operational issues post-update.
- **Configuration**: More granular than SQS (SQS is just "give me a queue"). Kafka configuration is managed via CloudFormation files defining connectivity, security groups, and broker settings.
- **Deployment**: Kafka is deployed as a shared resource before microservices. CloudFormation handles the Kafka cluster, parameter store values, ECR repos, SNS topics, and SQS queues—all foundational infrastructure.

### Kafka Topic Organization

One **Kafka topic** exists per activity type:
- One topic for email events
- One topic for SMS events  
- One topic for form submissions
- One topic for event tool activities
- One topic for MA flow events
- etc.

Each topic is consumed by corresponding batch production workers (explained in next section). Kafka is only used by the audience subscription worker (producer) and batch production workers (consumers).

---

## Batching: From Individual Events to Request Batches

### The Batch Production Worker

**Purpose**: Prevent overwhelming the CRM with individual API requests. Instead, events are batched before sending.

The **batch production worker** reads messages from Kafka topics one by one (e.g., for a campaign sending 500,000 emails: 500,000 sent events, 500,000 delivered events, thousands of open/click events). Instead of calling the CRM API for each event:

1. **Reads events from Kafka** for a specific Kafka topic
2. **Groups by installation** – Creates an in-memory batch for each combination of:
   - Account ID
   - Section ID
   - Integration ID
   - Event type

3. **Batching limits** – A batch is flushed (sent downstream) when:
   - **Size limit reached**: 200 KB per batch (SQS has a 256 KB message limit, so 200 KB provides safety margin)
   - **Time limit reached**: 30 seconds of inactivity (if no new events arrive, flush what you have)

4. **Batch ID generation** – Each batch receives a GUID for traceability.

### Architecture: Multiple Batch Production Workers

Six replicas of the same batch production worker service run in ECS, one for each event type:

- Batch Production Worker (Email)
- Batch Production Worker (Event Tool)
- Batch Production Worker (Form)
- Batch Production Worker (MA Flow)
- Batch Production Worker (SMS)
- (others as needed)

All replicas run the same Docker image; they differ only in which Kafka topic they consume from.

---

## Performance: Scaling and Load Characteristics

### Auto-Scaling Behavior

The outbound service uses **ECS auto-scaling** based on **CPU usage**:

```
If CPU > threshold → ECS adds tasks
If CPU < threshold → ECS removes tasks
```

Scaling takes time to activate (not instantaneous). However, this is acceptable because the outbound flow is **fully asynchronous**—end-to-end latency of a few seconds to several minutes is expected and acceptable to customers.

[Erik Andersson]: > "Usually for this outbound service like we have, if things are as it should, we have like it usually takes a couple of seconds for the message to come from like from audience to the CRM. But we have said told the customer like you can easily expect it to take like up to 5 minutes before it is in the CRM."

### Load Spikes and Failure Scenarios

**High-impact operations that cause worker load spikes**:

1. **File Import** – Worst case. If 1,000,000 profiles are imported with consent on a tracked subscription:
   - Each profile generates one consent update message
   - The audience subscription worker processes all 1,000,000 one-by-one
   - Worker is put under heavy load; everything takes longer
   - **Primary impact**: That specific customer's messages are throttled
   - Other customers' workloads are unaffected due to grouping by account/section/integration/event type

2. **Large Email Campaign** – If 500,000 recipients receive an email:
   - 500,000 sent events + 500,000 delivered events + (opens × N) + (clicks × N)
   - Massive spike, but acceptable because it's asynchronous

3. **Form Submissions** – Sporadic, creates minimal load spikes

**Real-world incident** [Erik Andersson]: A customer imported 400,000 profiles with incorrect CRM IDs. The worker handled the load but was heavily stressed during processing.

### Failure and Retry Mechanism

A common library across the entire Justin platform handles SQS message reading. On failure:
1. Message is placed back on the queue
2. Exponential backoff applied (varies by service/stage)
3. Message is automatically retried

If a message fails due to high CPU and auto-scaling hasn't caught up yet, it will be retried once capacity becomes available.

### Data Validation Issues During Import

**CRM ID formatting problem**: Excel automatically formats large numbers with commas and converts them to floats. A customer imported CRM IDs that were integers (e.g., `contact123500000`) but Excel converted them to strings with commas. The import proceeded, but all CRM IDs were now wrong format, causing validation failures downstream.

**Workaround warning**: Never recommend file import for profiles that need CRM IDs. This is why the audience subscription worker discards profiles without CRM IDs.

---

## Consent Updates: Separate Fast Path

**Unique behavior**: Unlike events, **consent updates do NOT go through Kafka or batch production workers**.

When a consent update arrives at the audience subscription worker:
- It skips Kafka and batch production entirely
- It goes directly to the **outbound worker**
- Sent immediately to the CRM

[Erik Andersson]: > "Consent updates are the only thing which is not actually batched. So audience will send it to us and it will reach the odd sub worker just as normal. However, instead of the consent going through Kafka and then the batch production worker and waiting there, it will go immediately from the odd sub worker to the outbound workers."

**Reason**: Consent changes often require rapid synchronization for regulatory compliance (e.g., GDPR opt-out).

---

## Message Structure: Consent vs. Event Batches

### Consent Update Message

```json
{
  "account_id": "<account_id>",
  "section_id": "<section_id>",
  "integration_id": "<installation_id>",
  "topic": "email|sms",
  "consent_status": "opted_in|opted_out",
  "source": "file_import|ui|api|...",
  "timestamp": "<ISO_timestamp>"
}
```

The **source field** is crucial for debugging. If consent updates fail with 404 (contact not found), the source field reveals whether it came from a file import (likely indicates bad CRM ID data) or elsewhere.

[Erik Andersson]: > "I here's where you would see if it was from a file import. So let's say that I do a file import and I completely mess up the CRM ID... then you can look at the source and then if the consent update failed with a like 404 then if it is file important, you can probably conclude that yeah, this consent update came because from the file important."

### Event Batch Message

```json
{
  "account_id": "<account_id>",
  "section_id": "<section_id>",
  "integration_id": "<installation_id>",
  "batch_id": "<generated_uuid>",
  "batch_type": "email|sms|form|event_tool|ma_flow",
  "profile_data": [
    {
      "profile_key": "<apsis_profile_key>",
      "fields": {
        "crmId": "179",
        "email": "user@example.com",
        "phone": "+1234567890"
      },
      "events": [
        {
          "event_type": "email.sent",
          "event_data": { ... },
          "timestamp": "..."
        },
        {
          "event_type": "email.delivered",
          "event_data": { ... },
          "timestamp": "..."
        },
        {
          "event_type": "email.opened",
          "event_data": { ... },
          "timestamp": "..."
        }
      ]
    },
    { ... more profiles ... }
  ]
}
```

**Fields in profile_data**:
- **profile_key**: APSIS internal profile identifier
- **fields**: Identifying attributes (CRM ID, email, phone) for CRM to locate contact
- **events**: All events for this profile, grouped together

All messages are **logged with batch ID** in CloudWatch, enabling traceability and debugging.

---

## The Outbound Worker: Final Delivery to CRM

The **outbound worker** receives batches from the SQS queue (downstream of batch production worker) and:

1. **Reads the batch** from the queue
2. **Identifies the installation** (account + section + integration IDs)
3. **Determines the event type** (email, form, event tool, etc.)
4. **Routes to the appropriate CRM endpoint**

Example routing:
- Email/SMS/form events → `POST /api/v1/activities`
- Consent updates → `POST /api/v1/consents`
- Others as per generic connector spec

The outbound worker does not perform transformations—it calls the CRM API with the prepared payload.

---

## Outbound Field Mappings and the Mapped Fields Concept

### The Problem: Field Name Mismatch

In earlier integrations (before mapped fields), when a form submission arrived, the CRM had to guess which Apsis fields corresponded to which CRM attributes:

**Example failure scenario**:
- Form has fields: "refrigerator", "my_custom_color", "favorite_color"
- CRM tries to find fields named "firstName", "email", "phone"
- No matches → CRM cannot populate contact fields intelligently

This is why **outbound field mappings** were introduced.

### The Solution: Two-Stage Field Mapping

**Stage 1 (Form Tool → Apsis)**: Form field mappings are set up in the form tool itself:
```
Form field "First Name" → APSIS attribute "first_name"
Form field "Email Address" → APSIS attribute "email"
Form field "Phone" → APSIS attribute "phone"
```

**Stage 2 (Apsis → CRM)**: Outbound mappings are configured on the integration page:
```
APSIS attribute "email" → CRM field "per_mail" (for E-deal)
APSIS attribute "phone" → CRM field "per_phone" (for E-deal)
```

Or for Dynamics:
```
APSIS attribute "email" → CRM field "emailaddress1"
APSIS attribute "phone" → CRM field "telephone1"
```

### Enabling the CRM to Create New Contacts

When outbound mappings are configured, the payload includes **mapped fields** that tell the CRM:

> "If you decide to create a new contact from this form submission, here is the data you should use and here are the corresponding CRM field names where you should place it."

**Key point**: APSIS never creates contacts in the CRM. The CRM decides, based on its own logic:
- Do I have enough data to create a new contact?
- Should I create a new contact or add to an existing one?
- What is my validation logic?

---

## The Generic Connector: Standard API Interface

### Why a Generic Connector?

Different CRM systems (E-deal, Dynamics, Tribe, Efficy Enterprise, etc.) have different native APIs. Instead of building bespoke connectors for each, a **generic connector specification** defines a standard interface that all CRM integrations must implement.

### Generic Connector Endpoints

All generic connector implementations expose these endpoints:

```
GET  /api/v1/{entity_name}/schema
     → Returns available fields for mapping

POST /api/v1/consents/contacts
     → Update consent for a contact

POST /api/v1/activities/contacts
     → Send activity events (email, form, etc.)

POST /api/v1/activities/{entity_name}
     → Send activity events for custom entities
```

Where `{entity_name}` varies:
- E-deal: "silhouette"
- Dynamics: "contact"
- Tribe: "contact"
- etc.

### API Specification Document

The **Generic Connector API Specification** (a Swagger/OpenAPI document) is the source of truth. It defines:
- Which endpoints are available
- Required and optional fields in request/response payloads
- Data types and formats
- Which features require which endpoints

**Versioning**: The generic connector uses **loose schema validation**:
- APSIS can add new optional properties to requests without requiring a version bump
- CRM implementations ignore properties they don't support
- If APSIS must **remove or change** existing properties, a version 2 would be required

### CRM Feature Declaration

When integrating a CRM, it declares which features it supports:
- Email campaign sync → requires event endpoints for email
- Form campaign sync → requires event endpoints for form
- Outbound mappings → requires mapped_fields in event payloads
- Consent management → requires consent endpoints
- etc.

APSIS only calls endpoints for features the CRM declares as supported.

---

## Security and Authentication Layer: Broker and Squid Proxy

### Credential Management: The Broker Service

**Problem**: Previously, CRM API credentials (API keys, tokens) were stored in plaintext in the database. While the database itself is encrypted, any developer with database access could read the credentials.

**Solution**: Introduce the **broker service**.

The broker service:
1. **Reads encrypted credentials** from the database (hashed, not plaintext)
2. **Decrypts** them using AWS KMS
3. **Adds authentication** to the outbound request (e.g., `Authorization: Bearer <token>`)
4. **Passes the request** downstream

[Erik Andersson]: > "The difference now for the general connector is that we we hash the secrets before we store it. So in the database like I as a developer I cannot go in and look at the whatever the customer has has entered."

**Important**: Developers see only that authentication succeeded; the actual credential values remain hidden.

### Network Security: The Squid Proxy

**Purpose**: Acts as an HTTP/HTTPS proxy to whitelist only approved domains.

**Flow**:
1. Outbound worker makes a request to a CRM domain
2. Request passes through the squid proxy
3. Squid proxy checks: **Is this domain in the whitelist?**
4. If yes → request passes through
5. If no → request is rejected

**Whitelisted domains are configured per integration**. For example:
- Dynamics: Microsoft authentication servers (static) + customer domain (dynamic)
- E-deal: Customer domain
- etc.

**Maintenance**: Squid proxy requires configuration updates only when a new tool or service is added that integration needs to call. This is rare; Erik notes he hasn't needed to modify it in 5 years.

### Architecture Diagram: Worker → Broker → Squid Proxy → CRM

```
Outbound Worker 
    ↓
Broker Service (add auth header)
    ↓
Squid Proxy (verify domain whitelist)
    ↓
CRM API Endpoint
```

**Both services are used by**:
- **Integration Manager** (inbound flow, querying CRM schema)
- **Outbound Worker** (outbound flow, sending events/consent)
- Any other service that needs to communicate with the CRM

---

## Outbound Manager: Configuration and Listener Registration

The **outbound manager** is a separate service (not shown in the main outbound flow diagram) responsible for:

1. **Registering listeners** in Audience when a sync is enabled
   - Example: User enables "sync email events to E-deal"
   - Outbound manager tells Audience: "Register a listener for email events on this section/activity"

2. **Unregistering listeners** when a sync is disabled
   - User disables the sync
   - Outbound manager removes the listener

3. **Managing the sync configuration** in the database

The outbound manager is called by the UI/integration page when users configure which activities to sync to which CRM systems.

---

## Workflow Summary: From Event Creation to CRM Delivery

### Example: Email Campaign Send Event

1. **User sends email campaign** to 500,000 recipients
2. **Audience generates events**: 500,000 sent, 500,000 delivered, 500,000+ opens/clicks
3. **Events published to SNS topic** (all at once, initial burst)
4. **Audience subscription worker** consumes from SQS:
   - Filters by type (email)
   - Validates each: has CRM ID? Installation active? Feature released?
   - Reconstructs to standard format
5. **Messages sent to Kafka** (email topic)
6. **Batch production worker (email)** consumes from Kafka:
   - Groups by installation
   - Batches by 200 KB or 30 seconds timeout
   - Sends batches to outbound worker queue
7. **Outbound worker** consumes from queue:
   - Reads batch
   - Determines CRM and event type
   - Calls broker service for credentials
   - Routes through squid proxy
   - Calls CRM generic connector endpoint with event batch
8. **CRM system** receives the batch:
   - Updates activity records
   - May trigger internal workflows (create leads, score contacts, etc.)
   - Optionally sends response back to APSIS (lead creation flow, next session)

### Example: Consent Update

1. **User opts out** of email in Apsis
2. **Consent attribute updated** (triggers audience listener)
3. **Audience publishes to SNS topic**
4. **Audience subscription worker** consumes:
   - Identifies as consent update
   - Validates profile has CRM ID
   - Reconstructs message with consent status and source
5. **Message sent DIRECTLY to outbound worker** (no Kafka, no batching)
6. **Outbound worker** (immediately):
   - Looks up consent mapping (Apsis subscription → CRM consent base)
   - Calls broker for credentials
   - Routes through squid proxy
   - POSTs to CRM `/api/v1/consents/contacts` endpoint
7. **CRM updates** the consent record

---

## Troubleshooting and Observability

### CloudWatch Logs

All services log to CloudWatch log groups named after the service:
- `/aws/ecs/audience-subscription-worker`
- `/aws/ecs/batch-production-worker-email`
- `/aws/ecs/outbound-worker`
- `/aws/ecs/integration-manager`
- `/aws/ecs/mappings-manager`
- etc.

**Batch tracing**: All events in a batch are logged with the batch ID, enabling full traceability:

```
Batch ID: a1b2c3d4-e5f6-g7h8-i9j0
Status: SUCCESS
Events included: [...]
```

Developers can search CloudWatch by batch ID to see the exact payload sent and the CRM response.

### Clean-up Needed

Multiple old Lambda log groups remain from when the platform used AWS Lambda (now replaced with ECS tasks). These are being cleaned up gradually as they can be misleading. The only Lambda still relevant:
- **Operations bus Lambda** – Triggered when account deletion/termination events occur, forwards to ECS tasks for cleanup

[Erik Andersson]: > "Everything in integration is a lambda anymore apart from the service which listens to the operations bus."

---

## Key Takeaways

1. **Outbound flow is fully asynchronous**: Events take seconds to minutes to reach the CRM, not milliseconds. This is acceptable and expected.

2. **Validation prevents bad data from flowing downstream**: The audience subscription worker is the critical gatekeeper, filtering out profiles without CRM IDs, checking installations are active, and verifying features are released.

3. **Batching is essential**: Individual events become requests in batches (200 KB or 30-second chunks) to reduce API overhead. Consent updates are an exception and use a fast path.

4. **Kafka provides flexibility over SQS**: While SQS is simpler, Kafka allows one topic per event type and cleaner consumer organization, at the cost of slightly more maintenance (updates every 6 months).

5. **Field mappings enable flexible form → CRM data flow**: A two-stage approach (form field → Apsis attribute → CRM field) decouples form design from CRM schema, but both stages must be configured for outbound mappings to work.

6. **Generic connector is the standard interface**: All new CRM integrations implement the generic connector spec, not custom APIs. The spec is the contract and source of truth.

7. **Broker + Squid Proxy secure API calls**: Credentials are hashed and decrypted on-demand; domains are whitelisted. This is transparent to most developers.

8. **The CRM decides what to do with incoming data**: APSIS delivers events and potential lead data; the CRM's own logic determines whether to create/update records.

9. **Auto-scaling handles spikes, but file imports are the worst load**: Large campaigns and form surges are handled gracefully. File imports with incorrect data are high-risk; avoid using file import for CRM ID updates.

10. **Documentation and wiki pages exist but need updates**: The squid proxy and lime connector pages are well-documented. Others are outdated. These will be refreshed over the next 2-3 months.

---

## Unresolved Questions and Action Items

### For Future Sessions
1. **Lead Creation / Lead Gathering Session** – Handling CRM responses, profile merges, new contact creation acknowledgments
2. **Microsoft Dynamics and Azure Session** – Azure app registration, Microsoft Entra, Power Platform, credential management for Dynamics instances
3. **Squid Proxy Deep Dive** – Reverse proxy mechanics, configuration file details, whitelisting domain logic
4. **Generic Connector API Spec Review** – Detailed walkthrough of the Swagger spec, endpoint requirements, request/response formats
5. **CloudWatch and Incident Debugging Session** – How to find logs, common error patterns, troubleshooting workflows (Shravidya has drafted initial content)

### Homework Suggestions
- **Learn Go fundamentals** (higher priority): syntax, static typing, pointers, common data structures, iteration, functional patterns. This is essential for understanding any code in the integration platform.
- **Optional**: Read the squid proxy wiki page as background, but Go learning should take priority.

### Knowledge Base Maintenance
- Share presentation slides and diagrams to a common folder (SharePoint or shared drive) for future reference
- Continue enriching the troubleshooting/incident documentation
- Review and update wiki pages that are outdated

---

## Technical Notes

### Why Kafka over SQS for This Stage?

The decision to use Kafka after the audience subscription worker is driven by:
- **Multiple consumer patterns**: One Kafka topic per event type allows clean consumer separation
- **Reduced message size fragmentation**: SQS has per-message overhead; batching before a second queue stage saves API calls
- **Partial failure resilience**: Kafka's offset tracking allows resumption without re-processing entire batches

SQS is still used for the final outbound worker queue because a single consumer (outbound worker) pulls all batched events and doesn't need Kafka's multi-topic capabilities.

### Why 200 KB Batch Limit?

SQS message size limit is 256 KB. The batch production worker uses 200 KB to account for:
- JSON wrapper overhead
- Metadata fields (account ID, section ID, batch ID, etc.)
- Safe margin for edge cases

This ensures no batch ever exceeds SQS limits when serialized.

### Why 30-Second Flush Timeout?

Prevents stale events from being held indefinitely. A form submission or event should reach the CRM within 30 seconds of occurrence, even if only a few events trickled in. This balances:
- Latency requirements (near real-time)
- Batching efficiency (wait a bit for more events)

---

## Revision Notes

This transcript was captured over approximately 1 hour 57 minutes. The session included a 7-minute break, after which note-taking briefly resumed. Transcription was stopped partway through the second half. The knowledge contained here represents the full technical content delivered; later discussions about Golang learning and documentation organization are included but are not technical content about the outbound flow itself.