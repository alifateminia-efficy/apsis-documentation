---
source_file: Benjamin - Intergrations Overview.txt
domain: Apsis One Integrations
topics:
  - Integrations Architecture Overview
  - Inbound Data Flow (Field Mapping, Subscription Mapping, Full Sync, Delta Sync)
  - Outbound Data Flow (Activity Syncing, Event Processing, Batch Production)
  - Generic Connector vs. Legacy Connectors
  - Partner and Supplier Models
  - Unified Data for Complex Queries
  - Integration Manager and Justin (Integration Middleware)
speakers:
  - Benjamin Nouhaag (Lead presenter, Integrations Architecture)
  - Lukasz Grabowski
  - Michal Rosikiewicz
  - Premanand Thangamani
  - Speaker 1 (unnamed, appears to be project lead)
key_components:
  - Justin (Integration Middleware Layer)
  - Integration Manager (ECS task, formerly Lambda)
  - Mappings Manager
  - Delta Sync Worker (SQS FIFO)
  - Full Sync Manager and Stack
  - Profile List Sync
  - Batch Production Worker (Kafka)
  - Outbound Manager
  - Audience Subscription Worker (All-Sub Worker)
  - Unified Data Provider (HTTP/2 streaming to Athena)
session_type: knowledge-transfer
subdomains:
  - Architecture
  - Different Types of Connectors
  - Generic Connector
  - Inbound Flow
  - Outbound Flow
  - Microsoft Dynamics
  - Efficy Enterprise 12.0
  - Efficy Enterprise 12.1
---

## Session Overview

Benjamin Nouhaag delivered a comprehensive knowledge transfer on the Apsis One Integrations domain, covering the conceptual architecture, historical evolution, and operational mechanics of the platform's integration layer. The session covered the original mission of Profile Cloud (now Audience) as a central hub synchronizing data across multiple systems, the design of Justin (the integration middleware), both inbound and outbound data flows, the shift from supplier-controlled integrations to a generic connector model with partners, and the emerging Unified Data capability for complex multi-table queries. The presentation was designed to give new team members conceptual understanding before deeper technical dives, while also documenting critical operational knowledge and governance processes around supplier and partner relationships.

---

## Historical Context and Design Philosophy

### Original Mission and Problem Statement

The integrations team was formed to address a fundamental business problem: enterprises in 2013-2014 had many disparate systems that were manually integrated by IT departments, resulting in data being out of sync, lagging, and unavailable when needed. The original solution was Profile Cloud, later rebranded as Audience in Apsis One.

> "The idea originally is that if you go to any company, at least in 2013, 2014 when this stuff started being full, you would see a lot of different systems and the IT departments were integrating all of these systems with each other in order to be able to provide some kind of holistic customer view and things were always out of sync, things were always lagging."

### Original Mission Statement

The integration team's mission was to:
1. **Ensure arrows exist** (bidirectional connections) between Audience and as many systems as possible
2. **Provide equivalent SLA** to what Apsis provides for the rest of Apsis One (24/7 pager duty, uptime guarantees)
3. **Manage external dependencies** where some logic inevitably resides outside Apsis

### Design Principle: Standardized Use Cases Per System Type

Rather than building one-off connectors for every system, the team standardized use cases by system category (CRM, ecommerce, etc.) and worked with suppliers who had deep knowledge of individual systems while maintaining minimal scope in external systems for monitoring and uptime assurance.

**Example of standardization**: The abandoned cart infrastructure for ecommerce systems. A centralized process ran once per hour, called standardized functionality to fetch carts from all connected systems, threw abandoned cart events into Audience, and had only minimal system-specific logic to fetch data from each platform.

---

## Justin: The Integration Middleware Layer

### Core Concept and Naming

Justin is the integration middleware layer whose job is to keep everything in sync. The name is "obviously named after Justin Timberlake, the greatest integration system you'll ever hear seeing on the radio."

The design principle is **modularity**: Justin provides generic infrastructure with small connector libraries for each system, so that only system-specific HTTP requests and field translation logic needs to be maintained per connector.

### Architecture Pattern

For a typical integration like Microsoft Dynamics:
- UI provides field mapping between profile attributes and contact fields
- Connector library needs only to: (1) fetch all fields via HTTP, (2) translate to Justin's format, (3) return to Justin
- Frontend consumes the standardized data

---

## Supplier-Controlled vs. Generic Connector Evolution

### Original Approach: Supplier Model (2013-2020s)

Early connectors were built by suppliers with deep system knowledge:

- **Microsoft Dynamics 365**: Contract with Gerran Kostenakana (Stockholm-based CRM consultant), who built a plugin with webhooks, provided API guidance
- **Shopify, Lime CRM**: Built with API docs and webhooks, no supplier involvement
- **Problem encountered**: Systems had edge cases and quirks not covered by webhooks. Example: In Lime CRM, you can remove the relationship between contact and marketing (which is not an opt-out), but the webhook doesn't cover it. These issues often appeared 1-2 days before launch and required 3-week fixes.

### Migration: Generic Connector Model

The team moved toward a **contract-based approach**:
- Provide an API spec/contract that external systems must fulfill
- Each system implements the generic connector interface
- Apsis provides only a config file for each implementation

**Key advantages**:
- External systems own implementation and support
- Scaling features across systems becomes easier (no per-system development)
- Apsis team has less operational burden for understanding external system quirks

**Current state**: Still maintaining legacy Dynamics and Lime CRM on field infrastructure, but new integrations use the generic model.

---

## Integration Page UI and Field Mapping

### Installation Flow

When a user installs an integration (e.g., Efficy Enterprise 12.1), they:
1. Complete a standard wizard
2. Provide URL and API key for the external environment
3. Land on a page with various integration settings

### Field Mapping Mechanism

**Integration Manager** (an ever-running ECS task, formerly Lambda) powers the field mapping UI:

1. When user opens field mapping page, Integration Manager is called
2. It determines which connector library to use based on URL (e.g., `dynamics` → Microsoft Dynamics library, `fc_enterprise` → Efficy library, generic systems → internal mapping)
3. Fetches schema from external system using the appropriate connector
4. Returns schema to frontend for dropdown rendering

**Schema Format**: When Efficy 12.1 renders field mappings, fields are displayed with automatic setup options. For example:
- `Clé` (key) field automatically maps to `Sierra Mighty`
- Users can override and customize mappings

---

## Inbound Data Flow: Profile Synchronization

### Core Inbound Concepts

**Three mechanisms drive inbound sync:**
1. **Real-time delta sync**: External system sends webhooks when contacts are created/updated
2. **Scheduled full sync**: Bulk download and processing of all contacts
3. **Profile list imports**: Tag profiles based on list membership in external system

### Mappings Manager and Justin's Laws

The **Mappings Manager** stores all field and subscription mappings with its own database user and table access. It enforces **Justin's Laws**—a set of constraints to prevent:
- Infinite mapping loops
- Invalid data transformations
- Other consistency violations

[Benjamin indicated Eric would cover Justin's Laws in more detail in a technical follow-up]

### Webhook Subscription and Delta Sync Worker

When installation completes:
1. Integration Manager subscribes to webhooks for all required events
2. When contact is created/updated in external system, webhook sent to **Delta Single Manager** (official endpoint) or custom webhook endpoint (not exposed to users)
3. Updates placed on SQS FIFO queue (message group ID = CRM ID)
4. **Delta Sync Worker** (ever-running ECS task) consumes queue:
   - Asks Mappings Manager which mappings exist for this account (with caching, cache invalidated on updates)
   - Translates message to Audience format
   - Sends to Audience in real-time

### Full Sync: On-Demand Infrastructure

Full sync is accessed via "Start New Sync" button and syncs everything per current mappings.

**Architecture**:
- **Full Sync Manager** (ECS task, formerly Lambda) receives sync operation
- Spins up entire stack **ad hoc** for this specific job (includes producer and consumer)
- **Full Sync Producer**: Fetches paginated list of contacts from external system using connector library, outputs in Justin format
- **Full Sync Queue**: Holds producer output
- **Full Sync Consumer**: Consumes queue, asks Mappings Manager for mappings/consent mappings, sends to Audience

**Rationale for ad-hoc infrastructure**: Historical queuing issues with FIFO-only behavior; spinning up per job gives better parallelism.

### Buffering During Full Sync: Maintaining Eventual Consistency

**Critical mechanism** to prevent data inconsistency:

During a full sync, real-time delta messages for that integration are **buffered** in a separate queue rather than processed immediately. This is necessary because:
- Full sync downloads everything from external system at a point in time
- Real-time delta messages arriving during sync might duplicate or contradict full sync data
- Solution: Buffer messages, set message visibility timeout, delay processing until full sync completes

**Flow**:
1. Full Sync completes → signals completion
2. Messages in buffer queue are then processed in order
3. Prevents broken eventual consistency (e.g., if Benjamin's data changes during full sync, we process both full sync version and delta update, in order)

**Limitation**: This is a workaround the team "often wish we didn't have." Historically they couldn't implement a better solution due to time/bandwidth constraints.

### Full Sync Status Transitions

- **Downloading**: Profiles downloaded and being processed
- **Finalizing**: All producer and consumer threads idle; waiting for Audience ingestion delay to catch up
- **Successful**: Audience ingestion delay caught up with the point in time when full sync wrote its last message

### Profile List Imports (Static and Dynamic Lists)

The UI also allows importing **Static Profile Lists** and **Dynamic Profile Lists**, though the terminology is confusing:
- **Dynamic list** (e.g., "Query" in Efficy): Automatically maintained by a query, evaluated at request time (like Apsis segments)
- **Static list** (e.g., "Profile" in Efficy): Manually maintained list

**How it works**:
1. User selects a list and clicks "Start New Import"
2. Request goes to Full Sync Manager → added to Profile List Sync Queue
3. Consumer spins up one worker per job (serially, not parallel, to avoid DDOS-ing customer infrastructure)
4. Worker fetches every contact on the list (held in memory, no intermediate queue)
5. Sends tag request to Audience for that list

**Historical issue**: Before edge case handling, these failed frequently with no queue recovery mechanism. Now more stable.

**Recurring imports**: Can be scheduled (e.g., 6:00 AM daily) via CloudWatch event triggering a Profile List Sync Lambda, which also puts items on the queue.

### Sync Conditions (Field Filtering)

Users can create **conditions** (e.g., "Active = true") to filter which contacts are synced:

**During processing** (full sync or delta sync):
- Check if condition is met
- Only sync if condition is true
- If inbound message shows condition is false and profile exists, optionally delete profile (feature flag controlled)

---

## Subscription Mapping and Consent Handling

Consent is managed via **Subscription Mappings**, which map external system consent data to Audience consent topics/bases.

**Two formats**:
1. **Native consent**: Dedicated consent resource in external system (preferred)
2. **Virtual consent**: Mapped from boolean fields on contact (legacy approach in historical connectors)

**Process**:
1. Integration Manager fetches available consent structures from external system
2. User maps to Audience topics via Mappings Manager
3. During webhook processing or full sync, consent mappings applied

**Key note**: Consent is **bidirectional**—both inbound and outbound. This differs from attributes, which flow only inbound.

---

## Outbound Data Flow: Activities and Events

### Overview of Syncable Activities

Users can sync **any activity created in Apsis One** to the external system:
- Email activities
- SMS activities
- Form submissions
- MA (Marketing Automation) flows
- Event registrations (invitations, registrations)
- **NOT**: Event surveys, though this could be added

When activity is created:
1. Signal sent to external system to create corresponding record
2. As events occur (opens, clicks, deliveries), batches of events transferred to external system

### Outbound Manager and CRM Sync Tab

**Frontend workflow**:
1. User creates email activity
2. Frontend queries Integration Manager: "Is there an integration installed that can sync emails?"
3. If yes, CRM Sync tab appears with option to sync
4. User checks "Sync with [CRM Name]"
5. **Outbound Manager** (service not shown in early diagram) acts as bridge between frontend and CRM

### Event Batching Architecture

Sending one event per external system request would generate massive traffic. The batching system is designed to handle this efficiently.

**Flow**:

```
Audience generates events
    ↓
All-Sub Queue (SQS FIFO)
    ↓
All-Sub Worker (ECS task)
    ├─ Verifies activity should be synced
    └─ Puts messages on Kafka partition
    ↓
Batch Production Worker (ECS task)
    ├─ Pulls from Kafka partition
    ├─ Groups messages by integration
    ├─ Batches until: (200KB limit OR 1 second elapsed)
    └─ Puts ready batches on SQS FIFO queue
    ↓
Outbound Worker (ECS task)
    ├─ Takes batch
    ├─ Uses connector library
    └─ Sends to external system
    ↓
Dead Letter Queue (on failure, after retries)
    ↓
Retry Driver Lambda (runs 6:00 AM daily)
    └─ Re-queues messages from DLQ
```

**Why HTTP/2 over other protocols** [related to unified data, see below]:
- Allows streaming responses
- Necessary for paginated data without blocking

**Batch size decisions**:
- 200KB limit: Stays below SQS 256KB message limit
- 1 second polling interval: Keeps latency low
- For 1 million events: Could result in 1000+ batches (acceptable, though noted as area for improvement)

### Idempotency and Retry Behavior

**Dead Letter Queue handling**:
- Messages that fail after retries go to DLQ
- Retry Driver Lambda runs daily and re-queues messages
- **Issue**: Infinite retries not currently capped (noted as room for improvement)
- **Workaround**: Only retry opt-out consent messages on redrive (not opt-in), because redrive loses eventual consistency

**Batch idempotency**:
- Each batch assigned a unique ID
- External systems (at least Efficy) expected to track already-processed batch IDs
- This responsibility is contractually specified and documented; Apsis cannot enforce
- If external system doesn't track and reprocesses, Apsis considers it their problem

**Benjamin's note**: 
> "Worth noting here is that when we do a redrive, we don't redrive any consent message saying opt in equals true. We only redrive consent messages saying opt in equals false because once we do a redrive, we no longer have eventual consistency."

### Operational Monitoring: Dead Letter Queue Management

Team established a **Monday operational meeting** practice during high-problem periods to:
- Review dead letter queue contents
- Identify recurring failures
- Understand why messages weren't being processed
- Plan improvements

**Failure causes** (non-exhaustive):
- External system bugs
- Timeouts
- Network problems
- Customer-imported consent data that can't be processed
- External system non-compliance with agreed process

---

## Outbound Mappings and Form Submission Handling

### Directional Attribute Flow Limitation

**Critical constraint**: Attributes flow **inbound only** to Apsis; they do NOT sync back to external systems.
- **Consent**: Bidirectional (both directions)
- **Attributes**: Unidirectional (CRM → Apsis only)

**Reason**: Historically, customers wanted the CRM to be the data master and requested no back-sync of Apsis-modified attributes.

### Form Submission Outbound Mapping

**Problem**: In early versions (pre-2022), form field mappings couldn't map directly to external system fields. Only profile attributes could be synced outbound.

**Workaround**: Created **Outbound Mappings** tab—a reversed version of field mappings.

**Current behavior**:
1. Form submitted with field values
2. All-Sub Worker receives form submit event
3. Worker does NOT map form fields directly
4. Instead, worker maps profile attributes (via Outbound Mappings) into outgoing payload
5. Sends to external system

**Problem with this approach**: 
- Form-specific data doesn't sync—only profile attributes
- If customer wants form field X to sync to CRM field Y, they can't do it per-form
- They must ensure profile attribute is updated and use Outbound Mappings
- CRM team has complained, but this is current architecture

### Feature Flags and Capability Detection

**Two-level capability checking**:

1. **Integration-level** (Apsis side): Does this CRM type support outbound mappings at all?
   - Configured in integration spec
   - Feature flag controls visibility

2. **Instance-level** (external system side): Does this specific customer's environment/version support it?
   - Integration Manager checks external system schema/capabilities
   - Some features available in production but not in dev instances

**Examples of variation**:
- E-deal: Customers often run on-premise with different versions (different APSIS connector versions)
- Customer A might support a feature; Customer B might not
- Apsis checks both levels before showing tabs/options to user

---

## Integration Installation Limitation: Single CRM per Section

### The CRM ID Problem

**Severe architectural limitation**: Only ONE native CRM integration can be installed per section at a time.

**Root cause**: All integrations map to a single field `CRM_ID` in Audience (established field space).

**Why this exists**: 
- When system was designed (super urgent release), used this field space
- Would require major migration to fix
- CRM ID field space was spec'd by Jonas for Apsis use but not actually used by CRM team

**Workaround introduced**: Bootstrap unique key space per integration:
- Integration has its own key space (e.g., `EFFICY_ENTERPRISE_ID`, `DYNAMICS_ID`)
- Never use shared CRM key space
- But all still mapped to profile's `CRM_ID` field for profile linking

**Impact**:
- Cannot install Dynamics AND Efficy on same section simultaneously
- Each must be installed per-section
- Prevents multi-CRM strategies within a section

**Possible future fix** [Benjamin suggests]: Fall migration project to map each integration's unique key to its own profile attribute instead of shared `CRM_ID`. This requires downstream changes to ensure profile identity resolution works correctly.

### Exception: Third-Party Integrations

Some integrations (Playable, etc.) are completely third-party and work differently:
- User provides no credentials to Apsis
- Apsis bootstraps a unique key space for the third party
- Provides API key only
- User copies/pastes setup to third party
- No dependency on CRM ID field

**These can be multiple per section** because they don't compete for the CRM ID field.

### Multiple Entity Types (Contacts, Leads, etc.)

For some integrations, users can create and manage profiles from multiple entity types (tables):

**Example**: 
- Default mapping for contacts with unique `CONTACT_ID` attribute
- Additional mapping for leads with unique `LEAD_ID` attribute

**Current state**:
- Integrations built after 2020 properly support this with separate identity attributes
- Pre-2020 legacy integrations depend on shared CRM ID field
- Requires significant migration to fix historical ones

---

## Marketing Automation Integration Nodes

### Use Case: MA Flow with CRM Node

Benjamin demonstrated an example MA flow:

1. **Listen** for event: "Person invited to event"
2. **Wait** 2 days
3. **Decision**: Did person confirm attendance?
   - **YES**: Do nothing
   - **NO**: Create task in CRM (e.g., call to confirm)

**CRM Node Configuration**:
- Select integration (Efficy Enterprise, etc.)
- Select action (create task)
- Map fields based on external system schema (urgency, assignee, etc.)
- Values are config values from external system

### How MA Integration Nodes Work

**Same outbound path as activities**:
1. When person reaches MA node, message generated
2. Put on All-Sub Queue (as node interaction event)
3. Goes through Batch Production Worker
4. Sent via Outbound Worker to external system

**Why batching matters for MA**:
- MA triggers based on segments and schedules
- Could send 500,000 people into a flow at 12:00 daily
- One-by-one sending would overwhelm external system
- Batching essential for scale

**Schema validation issue**: Integration error shown when schema can't be fetched (e.g., broken integration). Would benefit from being filterable like incoming event activities.

---

## Unified Data: Complex Multi-Table Query Capability

### Problem Statement: Flattening Complex Relationships

**Use case**: Paul runs a trade show (Expo) and wants to:
- Find **CEOs of companies** whose **employees attended last year's event**
- Send personalized email: "Hello Alfonso, seems like Imashek had a blast at Expo Poland last year..."

**Why this is hard with current architecture**:
- Current model syncs **flat profile documents** from CRM
- But real CRM data is **relational**: Event → Attendee → Person → Role (CEO) → Company
- Can't trivially query "get CEOs of companies who had attendees"
- Would require flattening (replicating company role/title on every person profile), which bloats Athena and increases costs

### Unified Data Solution (Built for Maxo, Reusable for Other CRMs)

**Architecture Overview**:

- **Audience**: Meta profile store (Athena) with React for real-time personalization
- **Athena**: Long-term storage for exports, segment evaluation
- **External CRM**: Has the complex relational data

**How it works**:

```
Email Tool triggers export with Unified Data query
    ↓
Audience opens HTTP/2 streaming connection to Integrations
    ↓
Integrations calls Integration Manager
    ↓
Integration Manager translates query into CRM-specific query
    ↓
Connector library fetches paginated data from CRM
    ↓
Integrations converts data to Apache Columnar format
    ↓
Stream data back to Audience over HTTP/2 connection
    ↓
Audience joins streamed CRM data with Athena export
    ↓
Email Tool personalizes and sends
```

### Why HTTP/2 Streaming

Regular HTTP/1.1 request/response won't work because:
- CRM might return 2,000,000 entries
- Can't fetch all at once (memory, timeout constraints)
- Need to stream paginated results as they arrive
- HTTP/2 supports multiplexing and server push, enabling true streaming

**Process**:
1. Integrations fetches paginated data from CRM
2. Translates each batch to Apache Columnar format on-the-fly
3. Pushes through HTTP/2 connection
4. Fetches next batch and repeats
5. Continues until CRM provides all data

### Pre-Made Queries in External System

- External system (CRM) has pre-defined queries with specific data exports (e.g., "CEOs of companies with event attendees")
- Query specifies which attributes are exposed in selected fields
- Integrations acts as middleware, fetching from query and providing to Audience

### Current Status and Production Readiness

**State**: Implemented for Maxo, successfully completed fall/winter 2024 project on time.

**Production usage**: 
- One Bertio customer has used it
- Not proven at 100-customer scale
- No severe issues reported, but not heavily battle-tested

**Recommendation**: Treat as experimental but reusable pattern. Will definitely be requested for other CRMs in future; foundation is solid.

### Historical Attempt: Flattening Approach

Team previously considered flattening complex data (e.g., replicating company title on every person) but rejected because:
- Audience keeps full history of every attribute
- Replicating company info on every person profile would massively increase Athena bill
- Unified Data approach avoids this by querying at send time instead

---

## Partner and Supplier Relationship Models

### Shift in Strategy: From Supplier to Partner

**Historical approach (2013-2020s)**: Suppliers controlled integration implementation
- Example: Gerran Kostenakana built Dynamics plugin
- Apsis paid them hourly, maintained SLA
- Apsis owned feature additions, architecture decisions
- Slow, unwieldy, hard to scale

**New approach (2020+)**: Partners implement generic connector, own revenue
- Example: Sideshop (for Dynamics), Intermail (for Loyalty)
- Partners implement Apsis generic connector spec
- Partners own support and customer billing
- Apsis provides only config file
- Partners can build custom features outside generic spec

### Active Partners and Suppliers

**Active Partners** (don't pay, get revenue):
1. **Sideshop**: Dynamics 365 CRM integration
2. **Intermail**: Loyalty integration (custom), Relation Plus (generic connector)

**Active Suppliers**: None currently (deliberate, to reduce costs)

**Historical Suppliers**:
- Gerran Kostenakana (Dynamics plugin, no longer under contract)
- Various others for Shopify, Lime CRM, etc.

### Why This Model Works Better

**For Apsis**:
- No ongoing supplier costs
- Feature additions integrated into generic spec (benefits all partners)
- Partners own support burden and customer complaints
- Scaling easier (add feature to generic spec, all partners get it)

**For Partners**:
- Own customer relationship
- Can charge whatever they want
- Can build differentiated features outside generic spec
- Benefit from Apsis' integration infrastructure

### Governance Documents and Contracts

Benjamin referenced several standardized documents (developed after trial and error):

#### 1. Supplier Delivery Process
- Specifies acceptance criteria before delivery
- Requires 75% test code coverage (enforced by build)
- Requires detailed walk-through of build
- **Definition of Done**: Detailed installation instructions, user roles, file paths, DB tables created/altered

#### 2. SLA Template
- Standardized response times and severity levels
- Maps partner severity to Apsis internal (P1/P2 = critical, P3+ = non-critical)
- Defines uptime expectations
- Tested and refined over multiple partnerships

#### 3. Supplier Agreement
- Legal contract template developed with Apsis legal
- Standardized contract to ensure key terms are covered
- Should be used or at least reviewed if entering new supplier relationship

#### 4. Partnership Agreement for Partners
- Specifies what partners can do with generic connector
- Defines Apsis termination rights
- States partners own everything (support, revenue, feature development)
- References generic connector spec and implementation guide

#### 5. Generic Connector API Spec
- Technical contract for partners
- Specifies exact API endpoints, request/response formats
- Versioned and maintained
- Updated manually and hosted on S3 with domain name
- Partners can always access latest version

#### 6. Generic Connector Implementation Guide
- Lengthy technical document for partners implementing generic connector
- Describes every step from partner's perspective
- Last updated in 2022 with major updates from 2024
- Benjamin planned to merge 2024 updates into main document (PR pending since 2022, awaiting review)

### Documentation and Knowledge Transfer

**Current location of documents**: 
- S3 folder with domain assignment (links to all supplier/partner materials)
- Personal files that need to be transferred to GitHub repository (action item)

**Outstanding PR**: Benjamin has a pull request pending since 2022 with merged 2024 updates. Needs review by Felix and merge approval from Mashek (noted as needed immediately).

**Handover process**: Benjamin created spreadsheet of active supplier/partner agreements as part of parental leave handover to Felix (location unclear to Mashek).

---

## Legacy CMS Integrations

### VMS (Virtual Marketing Systems) Integrations

Before current approach, Apsis had integrations with content management systems:

**Historical systems**:
- Drupal
- FP Server
- Sitecore

**How they worked** (different from profile-sync model):
1. CMS user editing content gets dropdown of Apsis segments
2. User specifies: "Show this content only to this segment"
3. CMS sends request to Audience segment evaluation API
4. Key space juggling on CMS side (nothing whitelisted for web key space)

**Current status**: All rendered open source. Documentation and repositories available (Benjamin to provide to team).

**Why different from current model**: These were about personalizing web content in real-time, not syncing customer data. Very different data flow and use case.

---

## Key Architectural Components Summary

### Services and Their Roles

| Service | Type | Purpose |
|---------|------|---------|
| **Integration Manager** | ECS task (formerly Lambda) | Fetches schemas, serves field mappings, checks capabilities |
| **Mappings Manager** | Database + service | Stores and validates field/consent/subscription mappings; enforces Justin's Laws |
| **Delta Sync Worker** | ECS task | Consumes real-time webhook messages; processes against mappings; sends to Audience |
| **Full Sync Manager** | ECS task | Orchestrates on-demand full syncs; spins up producer/consumer stack |
| **Full Sync Producer** | On-demand ECS | Fetches paginated contacts from external system |
| **Full Sync Consumer** | On-demand ECS | Processes contacts; applies mappings; sends to Audience |
| **Profile List Sync** | Lambda (scheduled) + ECS | Imports static/dynamic profile lists; tags profiles |
| **All-Sub Worker** | ECS task | Verifies outbound activities should sync; routes to Kafka |
| **Batch Production Worker** | ECS task | Consumes Kafka; groups events by integration; creates batches; posts to SQS |
| **Outbound Worker** | ECS task | Takes batches; sends via connector to external system |
| **Retry Driver** | Lambda (6 AM daily) | Re-queues dead letter messages |

### Data Queues and Flows

**Inbound**:
- Webhooks → Delta Single Manager (or custom endpoint) → SQS FIFO → Delta Sync Worker → Audience
- Full Sync: External system → Full Sync Producer → Full Sync Queue → Full Sync Consumer → Audience
- Buffering during Full Sync: Real-time messages → Buffer Queue → delayed until Full Sync completes

**Outbound**:
- Activity/node creation → All-Sub Queue → All-Sub Worker → Kafka → Batch Production Worker → SQS FIFO → Outbound Worker → External System
- Consent changes → All-Sub Queue → All-Sub Worker → SQS FIFO → Outbound Worker → External System (bypasses batch production)
- Failures → Dead Letter Queue → Retry Driver (daily) → back to SQS

### Key Design Decisions and Tradeoffs

| Decision | Rationale | Downside |
|----------|-----------|----------|
| **Batch processing for events** | Handle 500K MA flows daily | 1000+ batches for 1M events; complex retry logic |
| **Ad-hoc full sync stack** | Better parallelism than FIFO queue | Operational overhead; one machine per job |
| **Buffering during full sync** | Maintain eventual consistency | Workaround for fundamental design issue; lost consistency on retries |
| **Shared CRM ID field** | Early decision for profile identity | Can't install multiple CRM integrations per section; migration needed |
| **Attributes inbound only** | Customer preference (CRM as master) | Forms can't sync form-specific data; must use profile attributes |
| **Infinite retries** | Ensure no data loss | Can retry failed messages forever; noted as area for improvement |
| **HTTP/2 streaming for Unified Data** | Handle large paginated result sets | Requires HTTP/2; increases complexity |

---

## Known Limitations and Future Improvements

### Acknowledged Issues (from transcript)

1. **CRM ID field dependency**: Can't install multiple CRM integrations per section. Potential fall migration project needed.

2. **Form submission mapping**: Form fields don't sync directly; must map through profile attributes. CRM team has complained.

3. **Infinite retries on dead letter queue**: Messages could theoretically retry forever. Should add cap (e.g., N retries or X days).

4. **Batch size and queue choice**: For 1M events = 1000+ batches. Could optimize with different queue type.

5. **Outbound mappings feature flag complexity**: Two-level capability checking (Apsis-level and instance-level) is complex; documentation could be clearer.

6. **Profile List Sync robustness**: Holds contacts in memory; pre-2022 versions failed frequently with no queue recovery (now improved).

7. **Unified Data production readiness**: Only tested by one customer; not at 100-customer scale.

8. **CMS integrations open sourced but undocumented**: Drupal, FP Server, Sitecore integrations exist but not well documented; Benjamin to provide repositories.

### Suggested Future Projects

1. **Migrate from shared CRM ID field** (noted as potential fall migration with new team)
2. **Implement per-form outbound field mappings** (more flexible than current profile attribute approach)
3. **Add retry caps** to dead letter queue processing
4. **Webhook node improvements**: MA's webhook node sends one-by-one; could use batch processing like outbound activity sync
5. **Generic programmatic event subscription**: 5-6 years ago there were requests for programmable event subscriptions via One API; could reuse outbound batch infrastructure

---

## Operating Procedures and Monitoring

### Operational Meetings

Team established a **Monday operational meeting** practice to:
- Review dead letter queue contents
- Identify patterns in failures
- Track recurring issues reported by customers
- Understand root causes (CRM bugs, timeouts, network, customer data issues)
- Plan improvements

**Benefit**: Proactive monitoring; catch systemic issues before they escalate.

### Feature Flag Configuration

Uses feature flags to:
- Control visibility of tabs/options per integration
- A/B test new functionality
- Disable features on-the-fly if issues arise

### Monitoring and Alerting

Alerts exist for dead letter queue (mentioned by Lukasz). No explicit mention of alerting thresholds or escalation procedures, but this is implied to be in place.

---

## Unresolved Questions and Action Items

### From Transcript

1. **Redis vs. DLQ alerting**: Lukasz asked "Do you have some alert on that letter Q?" Benjamin confirmed yes but didn't specify alert conditions.

2. **Message retry cap**: Premanand mentioned 14-day SQS limit; Benjamin clarified they retry infinitely (not capped by days, but by SQS message visibility timeout). Noted as area for improvement.

### Action Items (from end of session)

**Benjamin to deliver**:
1. Send PR link to Speaker 1 (Mashek) for review/merge (2024 updates to generic connector implementation guide; pending since 2022)
2. Share list of all documentation (presentations, processes, policies, specs)
3. Provide list of active/historical suppliers and partners
4. Upload doc files to shared GitHub folder (needs Speaker 1 to clarify folder location and move files; Benjamin providing source docs)
5. Merge partner/supplier agreement documents into GitHub repository
6. Update generic connector implementation guide with 2024 changes (after PR is merged)
7. Provide documentation for Unified Data capability
8. Share links to open-sourced CMS integration repositories (Drupal, FP Server, Sitecore)

**Team to address**:
1. **Speaker 1 (Mashek)**: Review and merge Benjamin's 2024 updates PR; clarify shared folder location for documentation; request feedback from Felix on PR
2. **Lukasz**: Coordinate documentation handover; ensure all GitHub repositories are accessible
3. **Michal**: Decide on documentation format (GitHub wiki vs. .docx vs. other) and create supplier/partner inventory

---

## Architectural Diagrams Referenced

Benjamin shared several diagrams (to be provided to team post-session):

1. **Inbound flow**: Covers Integration Manager → Mappings Manager → Delta Sync Worker / Full Sync stack → Audience
2. **Outbound flow**: Covers activity creation → All-Sub Queue → Batch Production → Outbound Worker → external system
3. **Subscription/consent mapping**: Shows bidirectional consent, unidirectional attributes
4. **Unified Data architecture**: Shows HTTP/2 streaming between Audience and Integrations, with CRM query integration

---

## Key Takeaways

1. **Apsis Integrations is a sophisticated middleware platform** built over 10+ years to solve the problem of keeping customer data synchronized across disparate systems while maintaining Apsis SLAs.

2. **Justin is the core abstraction**: Provides generic infrastructure with pluggable connector libraries. Modularity is the design principle.

3. **Inbound is syncing profiles**: Field mappings, subscription/consent, full sync (on-demand), delta sync (real-time), and profile list imports. Buffering during full sync maintains eventual consistency.

4. **Outbound is syncing activities and events**: Batching is critical; handles 500K daily messages by grouping into batches of 200KB with 1-second polling. Consent is bidirectional; attributes are not.

5. **Supplier-to-partner model shift**: New integrations use a generic connector spec. Partners own implementation, support, and revenue. Apsis provides spec and config. This scales better than early supplier model.

6. **CRM ID field is a legacy bottleneck**: All native CRM integrations compete for same profile field; prevents multiple CRMs per section. Migration possible but requires coordination.

7. **Unified Data is a powerful emerging capability**: Enables complex multi-table queries from CRMs without flattening data. HTTP/2 streaming supports large datasets. Only early-stage customer usage but solid foundation.

8. **Governance documents exist**: Supplier agreements, SLA templates, generic connector spec and implementation guide. Should be reviewed and adapted as-needed for future partnerships.

9. **Known issues and improvements**: Infinite DLQ retries, form field mapping limitations, shared CRM ID field. Some addressed in recent updates; others documented for future roadmap.

10. **Operational discipline matters**: Monday ops meetings, feature flags, monitoring dead letter queue, and clear issue tracking help catch problems early and scale reliably.

---

## Related Documentation (To Be Provided)

- Generic Connector API Specification (updated 2024)
- Generic Connector Implementation Guide (merge pending)
- Supplier Agreement Template
- SLA Template
- Partnership Agreement
- Process and Policies Document
- Unified Data Technical Architecture Document
- Open-Source CMS Integration Repositories (Drupal, FP Server, Sitecore)
- List of Active/Historical Suppliers and Partners
- Integrations Architecture Diagrams (inbound, outbound, consent, unified data)