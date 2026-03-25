---
source_file: Erik - Outbound flow.txt
domain: Apsis One Integrations
topics: [Outbound Data Flow, Message Processing Pipeline, Batching and Queuing, Profile Identification, Consent Synchronization, Event Synchronization, Credential Management, External System Communication, Generic Connector]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Audience Subscription Worker, Kafka, Batch Production Worker, Outbound Worker, Broker Service, Squid Proxy, Generic Connector, SMS Topic, SQS Queues]
session_type: knowledge-transfer
subdomains: [Outbound Flow, Generic Connector, Architecture]
---

## Session Overview

This session covered the **outbound flow** in Apsis One Integrations—the process of sending data from Apsis to external CRM systems. The discussion focused on how events and consent changes are captured, validated, batched, and delivered to partner systems. Key topics included message pipeline architecture (SNS → SQS → Kafka → Batching → External APIs), profile identification requirements, the role of mappings in data transformation, and supporting infrastructure (broker service for credential management, squid proxy for security). The session identified several topics requiring separate detailed sessions: lead creation/gathering, Microsoft Dynamics with Azure integration, and the squid proxy.

---

## Scope and Session Planning

Erik clarified that the outbound flow session would cover only the initial delivery of data from Apsis to CRMs, while **lead creation** (the two-way interaction where CRMs respond and create new records) requires a separate session. The lead creation process involves internal merges and is significantly more complex. 

A separate session was also planned for **Microsoft Dynamics**, which requires knowledge of Azure resources, Microsoft Entra, and Power Platform credential management—topics outside the current scope.

---

## High-Level Outbound Flow Architecture

[Erik Andersson]: The outbound flow sends two types of data from Apsis to CRM systems:

1. **Consent** – Must be synchronized so consent status is identical in both systems
2. **Events** – Activity data from Apsis tools (email, SMS, forms, event tool, etc.)

### Trigger Mechanism

When a user creates an event in the event tool (or similar activity) and selects sync to a CRM system, a request is sent to Apsis identifying:
- The customer's desire to sync this activity
- The activity ID

Apsis then registers **event listeners** for all discriminators (event types) associated with that activity. This listener registration initiates the flow.

---

## Message Flow: SNS Topic to SQS Queue

**Step 1: Centralized Topic (SNS)**

All data from Audience (the messaging engine) flows into a single **SNS topic** within the integration system. This topic receives messages of all types:
- Email
- SMS
- Form submissions
- Event tool activities
- List consent changes
- Any other activity type

**Step 2: Queue Buffering (SQS)**

The SNS topic immediately dumps all messages into an **SQS queue** called the **"aud_sub_queue"** (Audience Subscription queue). This acts as a buffer for message processing.

---

## Audience Subscription Worker

### Core Responsibilities

The **aud_sub_worker** (or Audience Subscription Worker) is a consumer that processes messages from the aud_sub_queue. Its main tasks are:

1. **Identify message type** – Determine if the message is consent, event, form submission, etc.
2. **Validate required data** – Ensure all necessary fields are present before proceeding
3. **Apply business rules** – Decide whether the message should be forwarded or discarded

### Critical Validation Rules

#### Rule 1: Require CRM ID for Non-Imported Profiles
[Erik Andersson]: 
> If the profile does not have a CRM ID, we discard it in the slightest. We should not send consent updates to the CRM for profiles that have no CRM ID because that means these profiles did not come from the CRM.

**Example scenario**: A file import adds 60,000 new profiles with consent on a mapped subscription, but none have CRM IDs. These messages are discarded because they represent new profiles that should not be synced to an existing CRM.

#### Rule 2: Verify Installation Validity
The worker checks:
- Does a current installation exist for this account/section?
- Is the listener registration still active, or is this from an outdated/failed setup?

This prevents outdated listeners from creating erroneous messages.

#### Rule 3: Feature Release Verification
The worker verifies that the specific activity type (e.g., survey sync) has been released to production. If an unreleased feature's listener accidentally reaches production, the worker catches and discards these messages, preventing queue pollution.

### Message Scope and Specificity

[Lukasz Grabowski]: Does Audience send all events to the SNS topic for the entire section or account?

[Erik Andersson]: Only events for the specific activity are sent. We don't listen globally. If a user configures sync for Activity X, we register listeners only for that activity. Similarly for consent, we listen only to changes for the specific subscription that has a mapping configured, not all consent changes in the account.

---

## Data Validation and Scale Challenges

### File Import as a Performance Risk

[Lukasz Grabowski]: How does the worker handle large imports?

[Erik Andersson]: Unfortunately, yes. The aud_sub_worker processes messages one by one. If you import 1,000,000 profiles with consent changes on a mapped subscription, all 1,000,000 messages are processed individually.

**Real-world example**: One customer imported 400,000 profiles with incorrect CRM IDs. The worker came under heavy load.

### Impact and Auto-Scaling

While nothing crashes, processing time increases significantly. The outbound services use **ECS auto-scaling based on CPU usage**, but scaling is not instantaneous. 

[Erik Andersson]:
> We have auto scaling on everything in the outbound. Yes, the worker was put under heavy load. Nothing crashes, it's just that everything will take longer time, especially for this specific customer, because everything is grouped by the account, section, integration ID, and then the type of event.

Services are throttled per customer, so a single customer's heavy load primarily affects their messages, not global system performance.

**Typical processing time**: A few seconds to reach the CRM under normal conditions; customers should expect up to 5 minutes during heavy load or scaling events.

**Asynchronous retry logic**: Common library across Justin reads SQS messages and implements backoff-based retry on failure. When auto-scaling completes and CPU returns to normal, failed messages are reprocessed.

### Data Validation Issues in Imports

[Erik Andersson]: The import process is problematic for two reasons:

1. **Volume**: Large number of records processed simultaneously
2. **Data validation impossibility**: CRM IDs for systems like FSC Enterprise are integers (e.g., Contact 123-500000), but Excel formatting converts large numbers to floats with commas. Upon import, these become strings instead of integers, causing failures. Alternatively, customers use profile keys as identifiers instead of CRM IDs, which is equally invalid.

---

## Message Normalization

### Building the Outbound Event Structure

Once validated, the aud_sub_worker transforms the raw event data into a standardized message structure for downstream consumption:

```
{
  "profileKey": <string>,
  "eventType": <"email" | "sms" | "form" | "event_tool" | "consent">,
  "activityId": <string>,
  "correlationId": <string from audience>,
  "source": <string>,
  "eventTime": <timestamp>,
  "eventData": <raw event data>,
  "profileFields": <identifying fields>,
  "mappedFields": <mapped attribute data>
}
```

#### Profile Fields
These are identifying attributes the CRM uses to locate a contact. The worker includes whichever are available:
- CRM ID
- Email address
- Phone number

#### Mapped Fields
Data transformed according to configured mappings (covered in detail later in this section).

#### Event Data
The raw event payload varies by type:
- **Event tool**: Score properties, custom fields
- **Form submission**: All submitted field data
- **Email/SMS**: Delivery and engagement data
- **Consent**: Consent status and related metadata

---

## Kafka Topic Distribution

### Producer: Audience Subscription Worker

After normalization, messages are sent to a **Kafka instance**. Unlike SQS, Kafka is more powerful and allows finer control over topics and consumers.

[Lukasz Grabowski]: Is Kafka used only for this worker?

[Erik Andersson]: It's only used by the Audience Subscription Worker as a producer and then by consumers downstream.

### Kafka vs. SQS: Maintenance and Configuration Differences

[Lukasz Grabowski]: What are the operational differences for us?

[Erik Andersson]: There are two main differences:

**Maintenance**: Kafka requires periodic updates (released by AWS MSK). These happen infrequently—roughly once or twice per year. We've never experienced significant issues beyond clicking the update button.

**Configuration**: Kafka requires more setup than SQS (which is simpler—just "give me a queue"). Cloud Formation handles Kafka connectivity and security groups. The difference in configuration complexity is modest.

### Deployment Order

Kafka is deployed as a **shared resource** before microservices. The Cloud Formation template defines:
- Parameter store values for service configuration
- ECR repositories for container images
- Common queues and topics
- Security groups and networking

### Topic Structure

Each activity type has its own Kafka topic. When the Batch Production Worker reads from Kafka, it subscribes to topics and creates consumers for each activity type (email, SMS, form, event tool, etc.).

---

## Batch Production Worker

### Purpose: Efficiency Through Batching

The raw event stream from Kafka comes one by one (e.g., 500,000 individual email events). Sending these to the CRM one-at-a-time would create enormous handshake overhead.

The **Batch Production Worker** groups these events into **batches** before forwarding to the CRM.

### Batching Logic

For each Kafka topic, a consumer continuously reads events and organizes them **in-memory** by:
- Installation
- Activity type

**Batch flush triggers**:

1. **Size limit (200 KB)**: When batch reaches 200 KB, it flushes immediately. This limit exists because downstream SQS has a 256 KB message limit; 200 KB ensures safety margin.
2. **Time limit (30 seconds)**: If no new events arrive for 30 seconds, the batch flushes regardless of size.

**Batch ID**: Each batch receives a UUID for tracking and debugging.

### Deployment

The Batch Production Worker runs as **6 identical service instances** (one per activity type: email, form, event tool, SMS, MA flow, etc.). All use the same Dockerfile but listen to different Kafka topics.

[Lukasz Grabowski]: Where can we see these queues in AWS?

[Erik Andersson]: Kafka topics aren't visualized in AWS console the way SQS queues are. The Kafka cluster is visible, but individual topics inside it are not. Kafka is Apache Kafka adapted by AWS MSK, not a native AWS service, so it lacks the same UI polish as SQS.

---

## Outbound Worker Queue and Final Delivery

### Architecture

After batching, messages enter the **"outbound_worker_queue"** (SQS). This queue holds all batches awaiting delivery to CRMs.

### Outbound Worker Responsibility

The **Outbound Worker** reads messages from this queue and:

1. **Identifies the integration** – Which CRM and installation is this for?
2. **Identifies the batch type** – Is this email events, form submission, event tool, etc.?
3. **Calls the appropriate CRM endpoint** – Sends the batch via the corresponding API

---

## Consent Synchronization

### Consent as a Special Case

Consent updates are **not batched**. While events go through Kafka → Batch Producer → Queue → Worker, consent flows differently:

```
Audience Subscription Worker → (direct) → Outbound Worker
```

Consent bypasses the Kafka batching pipeline because consent status must propagate rapidly in some customer scenarios.

### Consent Payload Structure

```json
{
  "accountId": <string>,
  "sectionId": <string>,
  "integrationId": <string>,
  "subscriptionId": <string>,
  "topic": <string>,  // "email" or "sms"
  "consentStatus": <"opt_in" | "opt_out">,
  "source": <string>  // e.g., "file_import", "profile_update", "manual"
}
```

### Debugging via Source Field

The **source** field is recently added and crucial for troubleshooting. If consent updates fail with "contact not found" (404), the source reveals the cause:

[Erik Andersson]:
> If the consent update failed with a like 404, if it is file import, you can probably conclude that yes, this consent update came because from the file import someone imported the incorrect CRM ID.

For example, a 404 error with `source: "file_import"` indicates the import used wrong CRM IDs.

---

## Event Batch Payload Structure

### Example Batch Format

```json
{
  "batchId": <uuid>,
  "integrationId": <string>,
  "sectionId": <string>,
  "accountId": <string>,
  "batchType": "form",  // or "email", "sms", etc.
  "profileData": [
    {
      "profileKey": "key123",
      "fields": {
        "email": "user@example.com",
        "crmId": "179"
      },
      "events": [
        {
          "eventType": "form_submit",
          "eventTime": "2025-10-20T10:30:00Z",
          "eventData": { /* form field values */ },
          "correlationId": "corr-xyz"
        }
      ]
    }
  ]
}
```

### Batch ID Tracking

All outbound worker logs are keyed by batch ID, enabling debugging:

[Erik Andersson]:
> We log all of this. So if you would need to debug log, have have this specific event been sent to the CRM system? Did it succeed or not? We always log the batch with this specific ID was successful or the batch with this specific ID failed, and then you can search in Cloudwatch on the batch ID.

### Debugging in CloudWatch

Log groups follow the naming pattern of service names:
- `integration_manager` for Integration Manager logs
- `mappings_manager` for mappings-related logs
- Similar naming for other services

> Note: Many old Lambda log groups from the legacy system remain in CloudWatch and can be misleading. These should be cleaned up (ongoing effort).

---

## Profile Identification Fields

### Static CRM Contracts

The **fields** property in outbound payloads contains CRM-identifying attributes. These are **static and contractual** between Apsis and each CRM:

[Erik Andersson]:
> These fields are nothing that the customer configures. This is static and is like a contractual agreement between us and the CRM system.

### Field Mapping Examples

**For E-deal (FEC Corporate)**:
- `per_mail` = email address
- `per_phone` = phone number
- `per_id` = person ID

**For Microsoft Dynamics**:
- `contact_id` = CRM ID
- `email_one` = email address (logical name)

**For FSC Enterprise**:
- `contact` = contact entity type
- Field names follow FSC naming conventions

These logical names are negotiated between Apsis and the CRM vendor once and remain constant. They are **not configurable** by customers.

---

## Outbound Mappings and Transformed Data

### Problem the Mappings Solve

Early iterations of lead gathering attempted to infer field mappings automatically by matching form field names (e.g., "first name", "email") to CRM fields. This failed when:
- Form fields didn't match CRM field naming
- Custom field names ("my_custom_color", "refrigerator") had no obvious CRM counterpart

[Erik Andersson]:
> How would the CRM system know what data to put in which attributes in the CRM system? You wouldn't. It would be a complete gamble.

### Solution: Outbound Mappings Configuration

On the integration page, customers configure **outbound mappings**—a reverse mapping from Apsis attributes to CRM fields:

```
Apsis Profile → CRM System
email → CRM email field
phone → CRM mobile field
firstName → CRM first name field
```

This is **separate from inbound mappings** (CRM → Apsis). The process creates a two-stage pipeline:

**Stage 1 (Form Tool)**: Map form field names → Apsis attributes
```
Form "Full Name" → firstName (Apsis attribute)
Form "Mobile" → phone (Apsis attribute)
```

**Stage 2 (Outbound Mapping)**: Map Apsis attributes → CRM fields
```
firstName (Apsis) → FirstName (CRM field)
phone (Apsis) → MobilePhone (CRM field)
```

### Mapped Fields in Payloads

When outbound mappings are configured, the batch includes **mappedFields**. These tell the CRM which new fields are being provided and can guide lead creation:

[Erik Andersson]:
> If the CRM would decide like I have enough data from this form submission to create a new contact. With utilizing the mapped fields, we can tell them like explicitly: if you decide to create something new from this, you should create a new silhouette which is like a type of entity inside the FEC corporate. You should populate the email attribute with this attribute and you should populate the silhouette mobile field with this phone number.

The CRM can use mapped fields as hints when creating new records, but **Apsis never requests record creation**. The CRM decides autonomously whether to:
- Create a new record
- Match to an existing record
- Ignore the submission

This two-way interaction is **lead creation**, covered in a separate session.

---

## Transformed Request Format

### What the CRM Actually Receives

The normalized Apsis payload (with profile fields) is transformed into CRM-specific format. Example for **E-deal**:

```json
{
  "recordId": "179",  // CRM ID
  "consentStatus": "false",
  "source": "profile_update",
  "topicId": <subscription_id>,
  "consentBaseId": <mapped_consent_base_id>,
  "mappedFields": {
    "FirstName": "John",
    "MobilePhone": "+1234567890"
  }
}
```

#### Key Fields Explained:

- **recordId**: The CRM ID from the profile. Property name is unfortunately "personId" in source code but represents the CRM ID.
- **consentStatus**: Boolean opt-in/out status
- **source**: Metadata indicating where the consent change originated (file_import, profile_update, manual, etc.)
- **topicId**: The Apsis subscription identifier
- **consentBaseId**: The CRM's consent list/basis, determined from the subscription mapping configuration
- **mappedFields**: Additional attributes the CRM can use for enrichment or lead creation (only if outbound mappings are configured)

### Generic Connector Standardization

All CRMs using the **Generic Connector** receive data in this standardized format. The CRM vendor implements endpoints according to the **Generic Connector API Specification**, which is the "absolute source of truth and Holy Bible" for the contract.

---

## Broker Service: Credential Management

### Problem: Credential Exposure

Previously, API credentials were stored in **plain text** in the database (though the database was encrypted). This allowed developers to query the database and view customer credentials directly—a security anti-pattern.

### Solution: Credential Hashing + Broker Service

For **Generic Connector integrations** (newer CRMs), credentials are **hashed before storage**. The **Broker Service** is an intermediary that:

1. **Reads hashed credentials** from the database
2. **Decrypts them** using KMS (Key Management Service)
3. **Adds authorization headers** to outbound requests
4. **Passes the request** to the squid proxy

[Lukasz Grabowski]: Does the broker talk to the database?

[Erik Andersson]: Yes, it reads credentials (read-only). It never writes or updates.

### Legacy Connectors

Older integrations (Lime, FSC Enterprise 12.0) still store credentials in plain text in the database. Only Generic Connector integrations use the broker service.

---

## Squid Proxy: Security and Request Routing

### Purpose: Domain Whitelisting

The **Squid Proxy** is an HTTP(S) proxy that ensures outbound requests go only to whitelisted domains. When a customer installs an integration:

1. Their CRM domain (e.g., `https://customername.subsomething.crmsystem.com`) is added to the **whitelisted domains** table
2. Every outbound request from any integration service is routed through Squid Proxy
3. The proxy checks: *Is this domain in the whitelist?*
4. If yes → request passes through
5. If no → request is rejected

[Erik Andersson]:
> The squid proxy is there to verify the domain that you are trying to connect to. Do we allow that? Yes or no?

### Configuration

Configuration is handled automatically:
- Services start with an `HTTPS_PROXY` environment variable
- HTTP clients route traffic through the proxy automatically
- No explicit business logic required

### Integration with Broker Service Flow

```
Outbound Worker
    ↓
Broker Service (adds auth headers)
    ↓
Squid Proxy (checks domain whitelist)
    ↓
CRM System API
```

### Static Whitelists

Some endpoints are **statically whitelisted** because they're accessed by multiple integrations:
- Microsoft Dynamics authentication servers
- Generic connector schema endpoints

### Complexity and Maintenance

[Erik Andersson]:
> The Squid proxy is by far the most complicated of all of these. So let's have a separate session about it. Luckily, I have a very extensive wiki page about this.

The proxy requires detailed knowledge for troubleshooting but rarely needs modification. Erik hasn't touched it in five years—it "just exists." A separate session and wiki review are planned.

---

## Generic Connector: Contract Between Systems

### API Specification as the Source of Truth

The **Generic Connector API Specification** is a Swagger document defining all available endpoints and expected request/response formats. It serves as the contract between Apsis and all CRM vendors using the generic connector.

### Feature Support Declaration

CRMs declare which features they support in their configuration:
- Email campaign sync
- Form submission handling
- Consent management
- Outbound mappings
- Others

If a CRM declares support for "email campaign sync," Apsis expects them to have implemented event endpoints ready. If they don't declare support, Apsis never calls those endpoints.

### Versioning Strategy

The specification hasn't required a major version bump yet. Instead, Apsis has:
- **Added new properties** (e.g., source, lastUpdated fields to consent)
- **Maintained loose schema validation** – CRMs should expect additional properties and discard what they don't use
- **Agreed with all CRM partners** that property additions are safe

### When Versioning Is Needed

A version 2 would be required if Apsis:
- Removes existing properties
- Changes existing property names
- Modifies response formats

Any such breaking change requires communication to all integrated CRM partners.

### Extending the Specification

[Erik Andersson]:
> Any change of the generic contract needs to be communicated to each and everyone that has integrated with the generic connector.

New functionality is added rarely, but each addition involves:
1. Internal discussion with product and development
2. Update to the specification
3. Communication to all CRM partners
4. Coordinated implementation

---

## Integration Manager's Role in Outbound Flow

### Outbound Manager (not covered in detail in this session)

The **Outbound Manager** is a separate service responsible for:
- Receiving "sync" requests from Apsis tools (email, SMS, form, event tool)
- **Registering listeners** in Audience for the specified activity
- **Managing subscription mappings** for consent

When a customer clicks "sync to CRM," the Outbound Manager:
1. Registers event listeners for all discriminators of that activity
2. Creates mappings if consent sync is enabled
3. Marks the installation as active for that activity

When a customer unsyncs:
1. Deletes event listeners
2. Marks the activity as inactive (but keeps the record for reactivation)

[Erik Andersson]:
> If you select to unsync it, we just mark it as like not active, but you can reactivate it at any given time if needed.

---

## Outbound Flow Responsibilities and Lead Creation Boundary

### What Outbound Flow Does

[Lukasz Grabowski]: Can we say outbound flow has two responsibilities: sending consents and events, and delivering data for potential leads?

[Erik Andersson]: Yes, we could frame it that way. Consent is purely one-directional. Events are also outbound, but when we send events, we simultaneously provide data hints (mapped fields) that **could** be used for lead creation. The CRM decides what to do with that data.

### What Outbound Flow Does NOT Do

Outbound flow **never requests creation** of new CRM records. It only:
1. Sends events with optional hints (mapped fields)
2. Sends consent updates to existing profiles

### Lead Creation: The Response Handling

When a CRM responds to an event saying "I created a new record from this" or "I matched this to an existing record," that is the beginning of the **lead creation flow**—a separate session topic.

[Erik Andersson]:
> The logic to create or not create fully resides inside the CRM system depending on how the developers have implemented it. Apsis is never making any request to create new resources in the CRM system.

Each CRM implements different logic:
- Some CRMs always create a new record (manually reviewed later)
- Some have automated validation (accept/reject based on data quality)
- Others use hybrid approaches

---

## Recommended Learning Path

### Two Challenges Identified

[Lukasz Grabowski]: We face two challenges: understanding numerous components and how they fit together, and learning to write Go code.

### Immediate Focus: Go Programming

[Erik Andersson]: I would really suggest you try to understand Go better first because that will be so essential for everything you do.

**Priority learning areas for Go**:
- Programming paradigm and design patterns
- Static typing concepts
- Pointer mechanics
- Common data types (strings, arrays, maps, slices)
- Basic patterns (input → process → output, loops, iterations)

**Why first?** Understanding Go fundamentals will make reading and modifying the codebase far easier than trying to learn Go alongside service architecture.

### Documentation Review: Selective and Staged

Rather than reading all wiki pages immediately:
1. **Skip for now**: Pages flagged as potentially outdated (various connectors) until Erik reviews them
2. **Recommended**: Squid proxy wiki if interested, but not essential before the dedicated session
3. **Better alternative**: Focus on Go learning while Erik updates documentation

---

## Unresolved Questions and Action Items

### Questions Deferred to Future Sessions

1. **Lead Creation and Gathering** – Separate session needed to cover:
   - How CRM responses are processed
   - Merging and duplicate profile handling
   - Two-way interaction semantics

2. **Microsoft Dynamics Integration** – Separate session needed to cover:
   - Azure resource management
   - Microsoft Entra configuration
   - Power Platform user management
   - Customer instance debugging via Azure credentials

3. **Squid Proxy Deep Dive** – Separate session planned to cover:
   - Reverse proxy mechanisms
   - Configuration details
   - Whitelist management
   - Troubleshooting

4. **Generic Connector API Specification** – Separate review session to clarify:
   - Endpoint contracts
   - Request/response formats
   - Feature support declarations

5. **Incident and Troubleshooting** – Shravidya has been documenting:
   - Common error patterns and resolutions
   - Log file navigation in CloudWatch
   - Root cause analysis workflows
   - To be enriched over coming months

### Knowledge Base Organization

**Action items for team**:
- Create shared folder structure for knowledge transfer artifacts:
  - `/Diagrams` – Save flow diagrams created during sessions (e.g., draw.io files)
  - `/Presentations` – Store slide decks and visual references
  - `/Transcripts` – Clean transcripts (ongoing)
  - `/Documentation` – Links to wiki and Confluence pages
- Clean up legacy Lambda log groups in CloudWatch (no longer used)
- Update and validate wiki pages as consultancy period continues

### Recommended Homework

**For Lukasz and Tomasz**:
1. **Primary**: Learn Go fundamentals independently:
   - Consider online courses, tutorials, or books on Go
   - Practice simple programs to understand syntax and patterns
   - This will accelerate future code-based learning

2. **Secondary** (if time permits):
   - Review Squid proxy wiki page for context
   - Skim Lime connector documentation if curious (but not essential)

---

## Key Takeaways

1. **Outbound flow is a unidirectional pipeline**: Apsis → SNS → SQS → Validation → Kafka → Batching → SQS → CRM API, with consent as a fast-path bypass.

2. **Profile identification is mandatory**: Every message must identify the profile via CRM ID, email, or phone. Messages without these are discarded.

3. **Batching solves scale**: Messages are batched (200 KB or 30-second timeout) to reduce API handshakes. File imports are the worst case for load.

4. **Auto-scaling is gradual**: CPU-based auto-scaling helps but isn't instant. Processing delays up to 5 minutes are expected during heavy load.

5. **Validation is aggressive**: Five validation gates prevent sending invalid, outdated, or unsupported data to CRMs.

6. **Mapped fields enable smart lead creation**: By providing the CRM with explicit field mappings, CRMs can intelligently create or match records. This is purely optional and CRM-driven.

7. **Security layers are transparent**: Broker service (authentication) and Squid Proxy (authorization) protect credentials and prevent unauthorized domains from receiving data, but require no explicit business logic.

8. **Generic Connector is the contract**: All newer CRM integrations use standardized endpoints and payloads defined in the Swagger spec. Changes to this spec must be communicated to all partners.

9. **Consent bypasses batching**: Real-time consent propagation is critical, so consent messages skip the Kafka/batch pipeline and go direct to the Outbound Worker.

10. **Outbound flow ends at delivery**: CRMs respond with "I created a record" or "I matched an existing record"—this response handling is lead creation, not outbound flow.

---

## Additional Resources Mentioned

- **Generic Connector API Specification** (Swagger document) – The contract between Apsis and CRM vendors
- **Squid Proxy Wiki Page** – Extensive documentation (planned for detailed session)
- **Troubleshooting and Incident Document** – Shravidya's work in progress; will be enriched over coming months
- **Confluence Pages** – Various architecture and design documentation (quality varies; some outdated)