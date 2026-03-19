---
source_file: Benjamin - Integrations Overview.txt
domain: Integrations
topics: [architecture-overview, inbound-sync, outbound-sync, delta-sync, full-sync, webhook-processing, consent-management, integration-patterns, supplier-partnerships, generic-connector, unified-data, field-mapping]
speakers: [Benjamin Nouhaag, Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1/Eric]
key_components: [Justin (integration middleware), Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager, Outbound Manager, Batch Production Worker, Unified Data, Generic Connector, SQS queues, Kafka, ECS tasks, Lambda functions]
session_type: knowledge-transfer
---

## Session Overview

This is a foundational knowledge transfer session on the **Integrations domain** within the APSIS One platform. Benjamin Nouhaag, the departing integrations architect, provides a conceptual and technical overview of how the integrations system works—specifically how data flows between external CRM systems and the APSIS One audience profile store. The session covers the original mission and evolution of the integrations strategy, detailed walkthroughs of inbound (sync from external systems) and outbound (sync to external systems) data flows, the architecture supporting these flows, supplier and partner working models, and an advanced capability called Unified Data for handling complex relational queries.

---

## Original Mission and Strategic Context

### Why the Integrations System Exists

In the early 2010s (2013–2014), enterprise customers faced a fragmented data landscape: many disconnected systems managed by IT departments, inconsistent data across systems, and no single holistic view of customers. The original **Profile Cloud** product (later renamed **Audience** in APSIS One) was designed to solve this by acting as a central profile store that integrates with all external systems.

> "The idea originally is that if you go to any company, at least in 2013, 2014 when this stuff started being full, you would see a lot of different systems and the IT departments were integrating all of these systems with each other in order to be able to provide some kind of holistic customer view and things were always out of sync, things were always lagging."

### The Integrations Team's Mission

The integrations team's responsibility is to:

1. **Ensure connectivity arrows exist** for as many external systems as possible
2. **Provide the same SLA (Service Level Agreement) as the rest of APSIS One**, including uptime guarantees and 24/7 pager duty
3. **Manage operational complexity** by standardizing use cases across system types (CRM, ecommerce, loyalty, etc.) rather than building ad-hoc connectors

The challenge is that some logic always resides outside of APSIS, making it difficult to guarantee uptime. To mitigate this, the team built a **modular connector strategy** where standardized functionality is centralized in APSIS, and system-specific logic is isolated in small connector libraries.

#### Historical Example: Abandoned Cart Infrastructure

Before this system was abandoned, the abandoned cart workflow exemplified the standardized approach:
- A standardized process ran once per hour
- It called standardized functionality to fetch carts from all integrated ecommerce systems
- It generated abandoned cart events into Audience
- System-specific logic was minimal (just the data fetch piece)

---

## Justin: The Integration Middleware Layer

### Purpose and Design Philosophy

**Justin** (named after Justin Timberlake—"the greatest integration system you'll ever hear on the radio") is the generic integration infrastructure that connects APSIS One to external systems. Its core design principle is **modularity**.

The architecture follows this pattern:
- A **generic infrastructure** in Justin handles common operations
- **Connector libraries** for each external system handle system-specific communication
- Each connector library is responsible only for translating between the external system's API and Justin's expected format

For example, if a user maps a Profile field to a Microsoft Dynamics contact field:
1. The front end calls Integration Manager
2. Integration Manager uses the Dynamics connector library to fetch the field schema from Dynamics
3. The connector makes an HTTP request to Dynamics, translates the response to Justin's format
4. The result is returned to the front end

### Evolution of Integration Approach

The team's strategy has evolved based on operational experience:

**Phase 1: Supplier-Controlled Connectors (Early Days)**
- Worked with suppliers (e.g., **CRM Consultana** for Microsoft Dynamics) who built plugins with webhooks
- Suppliers provided API guidance and custom solutions
- **Problem**: Frequent surprises in external system behavior just before launch (e.g., in LimeCRM, removing a contact-to-marketing relationship is different from opting out but wasn't covered by webhooks); required 3-week fixes

**Phase 2: Generic Connector with Partner Implementation (Current)**
- Instead of owning the connector logic, APSIS publishes a **contract/spec** that external systems must implement
- Partners implement the generic connector interface on their side
- APSIS only needs a simple config file per installation
- **Example**: Relation Plus (a loyalty platform by Intermail) implements the generic connector on their side; APSIS just maps configuration

This shift removes the need for APSIS to understand every nuance of external systems while maintaining operational control through clear contracts.

---

## Inbound Sync: Getting Data from External Systems into APSIS

### High-Level Flow

Data flows from external systems into APSIS One Audience through these stages:

1. **Installation & Schema Discovery** → Integration Manager fetches available fields/attributes from external system
2. **Mapping Configuration** → User maps external fields to Audience profile attributes
3. **Real-Time Sync** → Webhooks deliver changes as they happen (Delta Sync)
4. **Bulk Sync** → Full Sync downloads all data initially and on demand

### Components and Flow

#### Integration Manager Service

**Purpose**: Acts as the interface between the front end and external systems
- **Status**: Converted from Lambda to long-running ECS task
- **Responsibilities**:
  - Fetches schema from external systems (triggered when user opens field mapping UI)
  - Routes requests to appropriate connector library based on integration type (Dynamics, FC Enterprise, etc.)
  - For generic connector implementations, uses internal mapping to determine how to call the external system

**Example**: When rendering a field mapping dropdown for FC Enterprise 12.1:
1. User opens the integrations page and clicks on an installation
2. Front end requests available fields from Integration Manager
3. Integration Manager calls the FC Enterprise connector library
4. Connector makes HTTP call to FC Enterprise API, gets schema, returns formatted response
5. Front end displays dropdown of available fields

#### Mappings Manager Service

**Purpose**: Manages field mappings and ensures data consistency
- Stores mappings in its own database with a dedicated user account
- Enforces **Justin's Laws** (validation rules to prevent circular mappings and data conflicts)
- Updates webhooks whenever mappings change
- Maintains mapping cache accessed by workers (invalidated on updates)

#### Webhook Subscription and Delta Sync

When a user installs an integration and configures mappings:

1. **Automatic Webhook Subscription**: The system subscribes to webhooks for all entities that have mappings
2. **Webhook Endpoints**: 
   - Some systems send webhooks to an official endpoint in the Delta Sync Manager (preferred)
   - Others send to custom endpoints provided by APSIS (less ideal but necessary for some systems)
3. **Delta Sync Queue**: Updates arrive as messages in an SQS FIFO queue
   - Message group ID = CRM ID (ensures ordering per contact)
4. **Delta Sync Worker**: 
   - Long-running ECS task consuming the queue
   - Processes messages one-by-one or in batches of 10
   - Queries Mappings Manager for applicable mappings (with caching)
   - Sends profile updates to Audience in real time

**Real-Time Processing**: When a contact is created or updated in the external system, the workflow is:
```
External System → Webhook → Delta Sync Queue → Delta Sync Worker → Audience
```

### Consent/Subscription Mapping

Beyond field mappings, the system handles **consent mapping** (mapping external consent/subscription states to Audience topics):

1. **Consent Basis Discovery**: Integration Manager queries external system for available consent concepts
   - Could be a dedicated consent resource (e.g., explicit "Consents" table)
   - Could be individual fields on the contact record
   - In legacy connectors, designated as "virtual consent" (multiple fields) or "native consent" (dedicated resource)
   - Generic connectors don't distinguish; they just expose what's available

2. **Topic Mapping**: User maps external consent basis to Audience topics
3. **Real-Time Sync**: Processed alongside field mappings during delta sync

### Full Sync: Bulk Data Download

The **Full Sync** process downloads all contacts and their data from the external system to ensure initial consistency and periodic reconciliation.

#### Architecture

When a user clicks "Start" on a full sync operation:

1. **Full Sync Manager** (ECS task) receives the request and spins up an **on-demand stack** for this specific operation:
   - Dedicated **Full Sync Producer** (fetches paginated data from external system)
   - Dedicated **Full Sync Queue** (SQS FIFO)
   - Dedicated **Full Sync Consumer** (processes queue entries into Audience)

2. **Full Sync Producer**:
   - Uses connector library to call external system API
   - Fetches paginated contact list
   - Converts response to Justin's expected format
   - Puts each batch on the full sync queue

3. **Full Sync Consumer**:
   - Consumes from full sync queue
   - Asks Mappings Manager for applicable field and consent mappings
   - Sends records to Audience

#### Eventual Consistency During Full Sync

A critical issue: If the external system sends real-time delta sync messages while a full sync is in progress, there's risk of processing inconsistency (e.g., syncing the same update twice or missing an intermediate state).

**Solution**: Delta Sync Buffering

```
External System → Delta Sync (Buffered)
                     ↓
              If Full Sync Running:
              Set Message Visibility Timeout
              (delay in queue until full sync completes)
                     ↓
              After Full Sync Complete:
              Process All Buffered Messages in Order
```

The system maintains a **Delta Sync Buffer Queue** that:
- Accepts all real-time messages while a full sync is running
- Uses SQS message visibility timeout to delay messages
- After the Full Sync Manager signals completion and Audience ingestion catches up, replays buffered messages in order

This ensures eventual consistency: if multiple messages arrive for the same profile during full sync, they're processed after the full sync with their original order preserved.

#### Finalization and Success States

Full Sync UI displays states:

- **"Finalizing"**: Producer and consumer have drained (no more messages), waiting for Audience ingestion to catch up with the final message timestamp
- **"Successful"**: Audience ingestion has caught up; all data is live

The Full Sync Manager waits for:
1. Producer threads see nothing in paginated requests → producer "done"
2. Consumer threads see nothing in queue → consumer "done"
3. Full Sync Manager signals both done
4. Audience ingestion delay has caught up with the final message write timestamp → "Successful"

### Profile Lists and Queries

External CRM systems typically have **list management** features (dynamic and static lists). APSIS abstracts these as:

- **Static Profile List**: Manual list maintained by humans
- **Dynamic Profile List**: Auto-generated list based on a query

Different systems use different terminology:
- FSC Enterprise: "Profile" (static) and "Query" (dynamic)
- Microsoft Dynamics: "Static List" and "Dynamic List" / "Market"

#### Importing Profile Lists

When a user imports a profile list:
- **Does NOT create new profiles** (that happens via delta sync or full sync)
- **Adds a tag** to existing profiles that are members of that list
- Tracks membership per import operation; if re-imported and profile is no longer on list, tag is removed

**Flow**:
1. User selects a list from external system (e.g., "Newsletter")
2. Clicks "Start New Sync"
3. Sync request goes to Full Sync Manager → Profile List Sync Queue
4. **Profile List Consumer** spawns one worker per import job
5. **Profile List Worker** fetches all contacts on list, sends tag request to Audience

**Recurring Imports**: Currently hardcoded to 6:00 AM every morning via CloudWatch Event → Lambda trigger
- Can be extended to other schedules with minimal effort

**Note**: These syncs have historically been less robust than full sync (no queue for streaming results, all data held in memory). Failures occurred before edge case handling was added, but system is now stable.

### Sync Conditions (Filtering)

Users can define **conditions** that filter which records are synced:

**Example**: Only sync active contacts
```
Condition: field "active" == true
```

**Behavior**:
1. Stored in Mappings Manager
2. Evaluated during delta sync and full sync
3. If condition not met: record is not synced
4. If record was previously synced and condition becomes false: record is marked for deletion

This allows selective sync based on external system state without having to create filters in Audience.

---

## Outbound Sync: Sending Data from APSIS to External Systems

### High-Level Flow

When users create **activities** (emails, SMS, forms, marketing automation flows) in APSIS One, the system can sync these activities and their engagement events back to external systems.

**Supported Activity Types**:
- Emails ✓ (sync creation + opens, clicks, deliveries)
- SMS ✓
- Forms ✓
- Marketing Automation flows ✓
- Events ✓

**Not Supported**:
- Surveys ✗

### Outbound Manager and Activity Syncing

When a user creates an activity and opts to sync it to an external system (e.g., "Sync with FC Enterprise 12.1"):

1. **Front End** checks Integration Manager: "Is there an integration installed that can sync emails?"
2. If yes, shows "Sync to [System]" checkbox
3. User checks it and creates/sends the activity
4. **Outbound Manager** (not shown in original architecture diagram, but mentioned) bridges the front end and external system
5. External system creates a corresponding record (e.g., email campaign, task)

### Event Processing and Batching

When an email is sent, APSIS generates many engagement events (sent, delivered, opened, clicked, etc.). Sending these one-by-one to external systems would create excessive traffic.

**Solution**: Intelligent batching via Kafka and SQS

#### Architecture

```
Audience Events → Audience Subscription Queue (SQS FIFO)
                       ↓
                  All Sub Worker (ECS)
                       ↓
              [Verify integration installed]
              [Verify activity should be synced]
              [Put on Kafka partition]
                       ↓
            Kafka Partition (mixed events)
            Example:
              A1, B1, A2, C1, A3, B2, C2, B3
                       ↓
          Batch Production Worker (ECS)
          [Poll Kafka until: batch > 200KB OR > 1 second elapsed]
                       ↓
          Grouped Batches (per integration)
          Example:
            Batch 1: [A1, A2, A3]
            Batch 2: [B1, B2, B3]
            Batch 3: [C1, C2]
                       ↓
        Outbound Queue (SQS FIFO, 256KB limit)
                       ↓
          Outbound Worker (ECS)
          [Use connector library to send batch to external system]
```

**Why This Architecture**:
- Kafka allows interleaving of many messages from many integrations
- Batch Production Worker groups by integration and waits for either:
  - Batch size > 200KB (SQS 256KB limit; keeping well below)
  - 1 second timeout (keep things moving)
- Connector library on external system receives batch as one request
- For a million events, could result in >1000 batches, but no issues seen so far
- **Potential Improvement**: Alternative queuing mechanism (noted for future work)

#### Dead Letter Queue and Retry

If a batch fails:

1. **Dead Letter Queue**: Batch is moved to DLQ after retries exhausted
2. **Retry Driver Lambda**: Runs daily at 6:00 AM
   - Pulls everything from DLQ
   - Puts it back on Outbound Queue for retry
3. **Infinite Retry**: Currently no limit on retry attempts (improvement noted)

**Contract with External Systems**: Batches include a unique batch ID; external systems must track processed IDs to handle retries idempotently (no duplicates even if same batch is sent twice).

### Consent Bidirectionality

An important asymmetry in the current design:

**Inbound**: Field data AND consent both sync from external system → Audience

**Outbound**:
- Consent changes SYNC back (if profile opts out in APSIS, that's sent to external system)
- Attribute changes DO NOT sync back (only consent is bidirectional)

**Reason**: Customers historically prefer the CRM to be the "source of truth" for customer attributes. Attributes should flow one direction: CRM → APSIS.

**Consent Flow Back**:
1. User opts out in APSIS (e.g., unsubscribes)
2. Consent message goes to Audience Subscription Queue
3. All Sub Worker validates and routes to outbound
4. **Bypass**: Does NOT go through Batch Production (added later, before batching was mature)
5. Consent messages are sent individually (not batched) to external system via Outbound Worker
6. Goes through same DLQ/retry cycle

**Retry Caveat**: When redriving consent from DLQ:
- **Do NOT redrive "opt in = true"** (could create false positives)
- **Only redrive "opt in = false"** (opt-outs) to be safe
- Reason: After redrive, eventual consistency is lost; playing it safe

---

## Outbound Mapping: Forms and Custom Attributes

### The Problem

When a user creates a **form** in APSIS One, they configure field mappings (e.g., form field "Email" → profile attribute "Email"). These mappings are also used for form submission events.

**Desired Behavior**: Each form could have its own mapping to external system fields.

**Current Implementation** (pre-2022): This wasn't possible due to technical limitations.

### Current Solution: Outbound Mappings

An additional **Outbound Mappings** tab (available as feature flag per integration) allows field mappings in the reverse direction: APSIS attributes → External system fields.

**Important caveat**: Outbound mappings are NOT form-specific. They are account-level mappings that apply to all form submissions.

**Flow for Form Submissions**:
1. Form is submitted in APSIS
2. Form submit event goes to Audience Subscription Queue
3. All Sub Worker receives it
4. **Instead of mapping form submission data directly**, worker queries Outbound Mappings
5. Takes mapped attributes from the profile (not form submission event)
6. Constructs external system payload with those attributes
7. Sends via Outbound Worker

**Why This Design**:
- Centralized attribute mapping (not form-specific)
- Reuses Outbound Mappings infrastructure

**Known Issue** [CRM Teams Have Complained]: 
- Ideal behavior would be form-specific mappings
- Current design applies account-level mappings to all forms
- Would require significant rework to make form-specific

### Feature Flags and Connector Specifications

Whether Outbound Mappings are available depends on:

1. **Integration-Level Feature Flag**: Is this integration allowed to support outbound mappings at all?
   - Configured in APSIS back office
   - Set per integration type (Dynamics, FC Enterprise, etc.)

2. **Connector Spec Check**: Does this specific system instance support it?
   - Integration Manager calls external system to check available features
   - A customer running FC Enterprise v5 might support it, but v4 might not
   - System checks both: "Is this allowed for FC Enterprise in APSIS?" AND "Does this specific customer's FC Enterprise instance have it?"

3. **Dynamic Feature Discovery**: Some integrations may have different capabilities by environment
   - Example: Customer runs FC Enterprise on-premise with custom connector version
   - Production instance supports outbound mappings, but development doesn't
   - System detects this per instance

---

## Limitation: One CRM Integration Per Section

### The Core Issue

Currently, **only one CRM integration can be active per section at a time**. This is a significant limitation for customers wanting to use multiple CRM systems.

**Root Cause**: 

All CRM integrations map to a single **"CRM"** attribute key space in Audience (the `CRM_ID` field).

```
Integration 1 (Dynamics) → Profile.CRM_ID
Integration 2 (Salesforce) → Profile.CRM_ID (conflict!)
Integration 3 (FC Enterprise) → Profile.CRM_ID (conflict!)
```

If two integrations map to the same key space, profiles get mixed up: a profile synced from Dynamics would have the same CRM_ID as a profile from Salesforce, breaking isolation.

### Why This Happened

- **Historical Decision**: An old high-priority release required quick implementation
- **"Jonas Super Urgent Release"** situation: CRM_ID was specified once in Apsis specs
- **APSIS Never Changed This**: CRM built the system expecting this unique key space, but it was never implemented correctly

### The Proper Solution (Not Yet Implemented)

Each integration should have its own key space:

```
Dynamics Integration → Profile.Dynamics_ID
Salesforce Integration → Profile.Salesforce_ID
FC Enterprise Integration → Profile.FC_Enterprise_ID
```

This would allow multiple CRM integrations per section without conflicts.

### Current Workaround

- One CRM integration per section maximum
- Other integrations (non-CRM, like ecommerce or loyalty) CAN be multiple per section
  - Example: Can have both Shopify AND WooCommerce on same section
  - These don't use CRM_ID; they use their own key spaces

**Migration Path** [Potential Fall Project]:
- Migrate all CRM integrations to system-specific key spaces
- Requires data migration in Audience
- Could be a collaborative project between integrations team and Audience team
- Would unblock multiple CRM integrations per section

---

## Working with External Systems: The Evolution

### Phase 1: Supplier-Built Connectors

**Example**: Microsoft Dynamics via **CRM Consultana** (Swedish consulting firm)

**Model**:
- Consultana builds a custom plugin with webhooks
- APSIS pays them hourly
- Previously had SLA commitments (no longer maintained)
- Problem: Every feature request requires payment and coordination with external supplier

**Limitations**:
- Slow iteration (external company must build changes)
- Expensive (hourly billing + SLA costs)
- Difficult to scale features across multiple CRM systems
- Surprises in system behavior require expensive fixes

---

### Phase 2: Supplier with Service Level Agreement (Not Currently Active)

Documents prepared for this model (available in shared folders):

1. **Supplier Agreement**: Standard contract covering obligations, support, deliverables
2. **SLA Template**: Defines response times, severity levels, uptime targets
   - Severity mapping: P1/P2 = Critical, P3+ = Non-critical
   - Example response times: P1 = 1 hour, P2 = 4 hours, P3 = 24 hours
3. **Delivery Process**: Supplier must provide:
   - Complete build walkthrough
   - Minimum 75% test code coverage (enforced in CI/CD)
   - Detailed installation instructions
   - List of user roles required
   - Complete file inventory with paths
   - Database table/entity inventory
   - Change log

**Note**: No active suppliers under this model currently. If needed in future, templates are in shared documentation.

---

### Phase 3: Partnerships with Generic Connector

**Current preferred model**. Partners implement the **generic connector interface** on their side.

**How It Works**:

1. **APSIS Publishes Specs**:
   - Generic Connector API Spec (latest version published to S3 folder)
   - Generic Connector Implementation Guide (detailed walkthrough)

2. **Partner Implements**:
   - Builds their own connector that matches the spec
   - Handles all external system-specific logic
   - Can build additional features outside the spec
   - Supports their own customers

3. **APSIS Provides**:
   - A config file per customer (minimal)
   - Integration Manager calls to fetch schema (using the spec)
   - No ownership of external system understanding

4. **Revenue Model**:
   - Partner charges customers for the integration
   - APSIS may get revenue share (commercial terms vary)
   - Example: Sideshop charges €200/month per integration; customers pay Sideshop directly

**Active Partners**:
- **Sideshop**: Implements Microsoft Dynamics 365 CRM integration (branded as "Dynamics 365 CRM by Sideshop")
  - Customers migrated from CRM Consultana to Sideshop
  - Plan: Eventually deprecate CRM Consultana's Dynamics connector
  - Sideshop also builds Super Office (separate CRM)
  
- **Intermail**: 
  - Relation Plus (loyalty platform)
  - Intermail Loyalty (custom-built integration, not generic connector)

**Advantages of Partnership Model**:
- Partners own feature development
- APSIS focuses on generic connector stability
- Customers benefit from partner expertise in their system
- Easier to scale across multiple systems (each partner maintains their own)

### Generic Connector Specification

The **Generic Connector API Spec** defines:

- **Required Endpoints**: Schema fetch, data fetch, update, etc.
- **Expected Payloads**: Request/response formats
- **Idempotency Requirements**: How to handle retries
- **Feature Flags**: What capabilities a system can declare support for

Example: A partner implementing a generic connector for System X must:
```
GET /schema → return available fields/attributes
GET /contacts?page=1 → return paginated contacts
POST /batch → process batch of contact updates
POST /consents → process consent changes
```

**Version Management**: 
- Latest spec is published to S3 folder with domain name
- Partners can always fetch the current spec
- Spec updates are versioned

### Documentation and Handover

**Critical Documents** [To be shared with new team]:

1. **Partner Agreement**: Legal terms for generic connector partnerships
2. **Supplier Agreement**: Legal terms for supplier relationships (if needed)
3. **SLA Template**: Service level definitions
4. **Delivery Process**: Requirements for supplier deliverables
5. **Generic Connector API Spec** (latest version)
6. **Generic Connector Implementation Guide**: Step-by-step walkthrough for implementing partners
7. **Process and Policies Document**: Complete overview of supplier/partner management

**Action Items** [Benjamin noted]:
- Need to merge a GitHub PR with comprehensive supplier/partner documentation (pending since 2022, feedback received spring 2024)
- Pull request requires review and merge approval
- Upload .docx files to shared repository
- Create inventory of active and historical agreements

**Historical Integrations** [Open Source, Available for Reference]:
- Drupal (CMS)
- FP Server (CMS)
- Sitecore (CMS)
- All built before current partnership model
- Different architecture: Provide segment dropdown to CMS; CMS calls segment evaluation API

---

## Unified Data: Advanced Relational Queries

### The Problem

APSIS One's profile model is **flat**: each profile is a document with attributes and tags. However, many CRM systems store data relationally across multiple tables.

**Use Case Example**: Paul from AM I Events wants to send a personalized email to CEOs of companies whose employees attended his event last year.

```
Query Logic:
- Find all attendees of Event X from last year
  ↓
- For each attendee, find their Person record
  ↓
- For each Person, find their Company
  ↓
- For each Company, find its CEO
  ↓
- Send personalized email from CEO to CEO
```

**Data Structure in CRM**:
```
Event
  └─ Attendee (rel: Event → Person)
       └─ Person
            ├─ Role (rel: Person → Role → Company)
            │    └─ Company
            │         ├─ CEO (rel: Company → Role with type=CEO)
            │         └─ Other Roles
            └─ Other Attributes
```

**Current System Limitation**: APSIS only syncs flat profile attributes. It cannot represent:
- A person having multiple roles at different companies
- Different email addresses for different roles
- Complex joins across multiple tables

### The Solution: Unified Data

**Unified Data** architecture allows APSIS to query CRM data on-demand during email generation, joining it with Audience profile data.

**Components**:

1. **Audience (Meta Profile Store & Analytics)**:
   - Maintains full customer data and history
   - React: Real-time profile store (full lookup only, slow export)
   - Athena: Analytics warehouse (supports segment exports)

2. **Integration Manager** (Updated for Unified Data):
   - Provides HTTP/2 streaming endpoint
   - Accepts query requests from Email Tool
   - Fetches data from CRM (paginated)
   - Streams results in Apache Columnar format

3. **Email Tool (Personalization)**:
   - Defines a segment/audience
   - Joins Audience export with CRM query results
   - Personalizes email with joined data

4. **CRM System**:
   - Pre-built **queries** that expose relational data
   - Example Query: "CEOs of companies with attendees at Event X last year"
   - Returns structured fields that can be joined with profiles

### Architecture Flow

```
Email Tool: "Generate emails for Event X attendees"
  ├─ Trigger Audience export (via React or Athena)
  │   "Get all people who attended Event X"
  │
  └─ Query CRM via Integration Manager (HTTP/2 stream)
     "Get CEO info for each attendee's company"
     
     Integration Manager:
     ├─ Parse CRM query request
     ├─ Fetch from CRM (paginated):
     │   Person 1 → Company → CEO Name, CEO Email
     │   Person 2 → Company → CEO Name, CEO Email
     │   (continues until all fetched)
     │
     ├─ Convert to Apache Columnar format
     └─ Stream response back to Email Tool
     
Email Tool:
  ├─ Has Audience data: [attendee profiles]
  ├─ Has CRM data: [CEO details per attendee company]
  ├─ Join on: attendee.company_id = CEO.company_id
  └─ Template personalization:
     "Hello {{ceo_name}}, your employee {{attendee_name}} 
      attended {{event}} last year. Would you like to send 
      more reps this year?"
```

### Why HTTP/2 and Streaming?

- **HTTP/2 Streaming**: Allows continuous flow of data over a single connection
- **Why Not Regular Requests**: 
  - Example: 2 million attendees → fetching all in one request would be massive
  - Risk of timeouts, memory issues, single connection saturation
- **Streaming Approach**:
  - Fetch paginated results from CRM
  - Convert each page to columnar format
  - Stream to Email Tool immediately
  - Continue fetching next page
  - No need to hold entire result set in memory

### Implementation Status and Limitations

**Status**: 
- Built for Maxo (FSC's Marketing Automation suite)
- Product suite project completed on time (fall/winter 2024)
- **Limited real-world testing**: Only one Berto customer used it
- No failures reported, but not validated at scale with 100+ customers
- **Not production-hardened** (yet)

**Recommendation**: 
- Know this capability exists and can be reused
- Documentation available for deep dive if needed in future
- When customers request "send emails based on related data," this is the solution
- More technical walkthrough available separately

### Connector Integration

Like other APSIS features, Unified Data is **connector-agnostic**:
- Generic Connector implementations can expose pre-built queries
- Any CRM system can publish queries in columnar format
- Integration Manager handles the translation/streaming
- All system-specific logic lives in connector

---

## Operational Management and Support

### Handling Dead Letter Queue Issues

Outbound messages occasionally fail and end up in DLQ. Investigation and resolution:

**Common Reasons for Failure**:
- External system bugs or downtime
- Network timeouts
- Invalid credentials
- Customers' incorrectly configured consent data
- Rate limiting on external system API

[Premanand Thangamani] provided clarification on retry limits:
- Messages can stay in SQS queue up to **14 days**
- Retry Driver Lambda runs **daily** (6:00 AM)
- Messages are retried every day for 14 days (not a single retry window)
- Currently: **Infinite retry** after 14 days (room for improvement)

**Alerts**: Yes, alerts are configured on DLQ growth

### Operational Meetings

**Recommended Practice**: 
When integration problems are frequent, hold a weekly (Monday) operational review meeting:
- Review customer integration issues
- Examine what's in DLQ
- Identify patterns (e.g., "We've reported issue X to Supplier Y for 2-4 weeks with no resolution")
- Make decisions (escalate, workaround, deprecate, etc.)

**Historical Example**: 
- Team had operational problems, started Monday meetings
- Often ping vendors about issues reported weeks/months prior
- Eventually decide to stop doing something if unresolved

---

## Limitations and Known Issues

### Attribute Sync Direction (One-Way)

Field data only syncs **inbound** (CRM → APSIS). Customers' requested bidirectional attribute sync is not supported because:

> "Historically customers have not wanted that. They feel like the CRM should be the data master."

**Only exception**: Consent changes sync back to CRM (opt-in/opt-out)

### Form Submission Mapping

Forms in APSIS can be mapped to external system fields, but:
- Mapping is **account-level**, not form-specific
- All form submissions use the same outbound mappings
- **Desired behavior** (form-specific mappings) would require rework

### Multiple CRM Integrations Per Section

See earlier section: **Limitation: One CRM Integration Per Section**

### Batching Potential Improvements

- Currently: Batch size capped at 200KB (SQS limit is 256KB)
- Potential improvement: Use alternative queuing (e.g., Kafka for outbound as well)
- Concern: "For a million events, could be over 1000 batches" — not a problem yet, but could be optimized

### API Specification Updates

- Generic Connector API Spec last updated: **2024**
- Implementation Guide last updated: **2022** (needs updates from 2024 spec changes)
- Action: Benjamin noted these need to be merged by next day and published

---

## Unresolved Questions and Action Items

### Documentation and Handover

1. **GitHub PR Pending** (since 2022): Supplier/Partner documentation merge
   - Feedback received spring 2024
   - Requires review and approval
   - Benjamin to send PR link to new team lead

2. **Upload Documents** [Benjamin committed]:
   - All supplier/partner templates (.docx files)
   - Generic Connector specs
   - Implementation guides
   - Partner agreement template
   - Supplier agreement template
   - Process/policy documents

3. **Format Preference** [Discussed]:
   - New team prefers: GitHub Markdown documents
   - Fallback: OneDrive shared folder (will be moved by new team lead)

4. **Update Implementation Guide**:
   - Merge changes from 2024 generic connector spec
   - Expected completion: next day of session

### Supplier/Partner Inventory

[Michal Rosikiewicz asked]: Do we have structured documentation of active and historical supplier/partner agreements?

**Current State**: 
- Spreadsheet exists (created by Benjamin for Felix before parental leave)
- Location: Currently with Felix or in Benjamin's personal docs
- Content: List of suppliers/partners (very short list currently)

**Active Agreements**:
- **Partners**: Sideshop (Dynamics), Intermail (Relation Plus + Loyalty)
- **Suppliers**: None active (no active SLA agreements at this time)

**Action**: Benjamin to provide inventory or recreate from memory

### Integration Manager Schema Fetching

[Implicit from architecture]: Integration Manager handles schema discovery, but there may be system-specific edge cases not fully documented. More technical walkthrough with Eric (development team lead) recommended.

---

## Key Takeaways

### Strategic Understanding

1. **APSIS One integrations exist to solve the "fragmented systems" problem**: Customers have many disconnected systems; Audience provides a unified profile store.

2. **SLA and operational stability are paramount**: The integrations team commits to 24/7 pager duty and uptime guarantees, requiring architectural choices that isolate and monitor risk.

3. **Modularity through connectors**: System-specific logic is isolated in small, reusable connector libraries. All connectors follow a contract to enable scaling.

4. **Evolution from ownership to partnership**: 
   - Phase 1 (Supplier-owned) was expensive and slow
   - Phase 2 (APSIS-owned connector, suppliers paid) still had scaling issues
   - Phase 3 (Generic connector, partners implement) is the current direction

### Architectural Highlights

1. **Inbound**: Field mappings + consent mappings + delta sync + full sync with eventual consistency buffering = reliable data import

2. **Outbound**: Event batching + SQS/Kafka + intelligent retry = handling high-volume engagement data without overwhelming external systems

3. **Unified Data**: HTTP/2 streaming + connector libraries + Apache columnar format = complex relational queries without flattening data model

### Operational Reminders

1. **One CRM integration per section** is a current limitation (architectural debt from 2014–2022 era, needs migration)

2. **Dead letter queues need monitoring**: Alerts exist, but operational meetings help identify patterns and make escalation decisions

3. **Consent is bidirectional, attributes are not**: Intentional design reflecting customer preference for CRM as data master

4. **Testing at scale is limited**: Unified Data, for example, is production-ready but not battle-tested by 100+ customers

### Handover Notes for Incoming Team

- **Documentation is being consolidated** in shared repo (GitHub or OneDrive)
- **PR approvals needed**: Benjamin has pending contributions that need review
- **Supplier/partner inventory** to be provided
- **Contact relationships for historical integrations** (Shopify, LimeCRM, Drupal, etc.) should be maintained
- **Feature requests** for multiple CRM integrations or form-specific mappings should reference architectural debt and migration path

---

## Supporting Resources to be Shared

Benjamin committed to providing:

1. All presentation slides
2. Architecture diagrams (inbound, outbound, unified data flows)
3. Supplier/Partner management templates and agreements
4. Generic Connector API specification (latest)
5. Generic Connector Implementation Guide (2024 version with updates merged)
6. Process and policy overview document
7. List of active/historical supplier and partner contacts
8. Links to open-source historical integrations (Drupal, FP Server, Sitecore)
9. Unified Data technical documentation and contract details

---

**Session End Time**: ~1h 31m

**Facilitators**: Benjamin Nouhaag (departing architect), Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1 (Eric, development team lead)