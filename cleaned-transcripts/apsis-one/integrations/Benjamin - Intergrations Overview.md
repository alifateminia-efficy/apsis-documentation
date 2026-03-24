---
source_file: Benjamin - Intergrations Overview.txt
domain: Apsis One - Integrations
topics: [Integration Architecture, Inbound Syncing, Outbound Syncing, Justin Middleware, Field Mappings, Consent Management, Full Sync Operations, Delta Sync, CRM Connectors, Partner/Supplier Models, Unified Data]
speakers: [Benjamin Nouhaag, Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1 (Eric)]
key_components: [Justin (Integration Middleware), Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager, Outbound Manager, All Sub Queue, Batch Production Worker, Connector Libraries, Audience (Profile Store), Athena (Data Warehouse)]
session_type: knowledge-transfer
---

## Session Overview

Benjamin Nouhaag delivered a comprehensive conceptual overview of the Apsis One Integrations domain to the incoming team. The session covered the historical context and mission of the integrations platform (originally Profile Cloud, now Audience), the architecture of the Justin middleware system, and detailed walkthroughs of inbound and outbound data synchronization flows. Key topics included the standardization strategy for different CRM and e-commerce systems, the shift from supplier-owned connectors to partner-driven generic connectors, and emerging capabilities like unified data for complex cross-system queries.

---

## Historical Context and Original Mission

### Why Audience/Profile Cloud Exists

The integrations team was originally created to solve a critical business problem. In 2013-2014, organizations used many disconnected systems managed by IT departments, resulting in:

- Systems constantly out of sync with each other
- Data missing when needed
- No holistic customer view across the enterprise

The original solution was Profile Cloud (rebranded as Audience in Apsis One), which sits in the middle of all external systems to provide a unified customer view.

### Original Integration Mission and Strategy

The integrations team's mission was twofold:

1. **System Coverage**: Ensure integration arrows exist for as many systems as possible
2. **Service Level**: Provide the same SLA (uptime guarantees, 24/7 pager duty) that Apsis provides for its own platform

**The Central Challenge**: Some logic will always reside outside Apsis, making it hard to guarantee uptime. To address this, the team adopted a **standardization strategy**:

- Identify standardized use cases for different system types (CRM, e-commerce)
- Create generic, reusable infrastructure for those use cases
- Work with suppliers/partners who specialize in each system
- Keep the scope of external system logic minimal so Apsis can monitor and support as much of the chain as possible

**Example**: For e-commerce integrations, they built standardized abandoned cart infrastructure that:
- Runs once per hour
- Calls standardized functionality to fetch carts from all connected systems
- Throws abandoned cart events into Audience for marketing automation to react to
- Requires only minimal system-specific code per connector

This approach prioritizes operational reliability over feature richness.

---

## Justin: The Integration Middleware System

### Overview and Design Principles

**Justin** is the generic infrastructure layer that sits between Apsis One and external systems. The name is a reference to Justin Timberlake — "it's job is to keep everything In Sync."

**Core Design Principle**: **Modularity**

The architecture separates concerns into:
- **Generic infrastructure** in Justin itself
- **Connector libraries** for easy communication with each system

### How Justin Modularity Works

The ideal flow for any integration:

1. **Integration Manager** (the Justin gateway service) receives a request
2. Determines which connector library to use based on the system identifier (e.g., "dynamics", "fc_enterprise", or generic)
3. Connector library makes HTTP request to external system
4. Connector translates response to Justin's standard format
5. Justin serves it back to the frontend/caller

This design allows adding new systems by implementing only the connector library, not changing core Justin logic.

### Evolution of Connector Strategy

**Early Approach (2010s-2020s): Supplier-Owned Connectors**

- **Example**: Microsoft Dynamics 365 via CRM Consultana
  - Company built a plugin with webhooks
  - Provided API guidance
  - Apsis maintained connector library in Justin
  - Apsis paid CRM Consultana by the hour with SLA agreements

- **Problem**: This approach became unwieldy
  - Every system had unexpected edge cases (Benjamin gave the example of Lime CRM, where you could either "opt out of marketing" or "remove the relationship between contact and marketing" — two different things, not covered by webhooks)
  - Issues often discovered 1-2 days before launch
  - Fixes required 3 weeks of coordination and payment
  - Adding features meant paying external contractors each time
  - Difficult to scale new features across multiple systems

**Current Approach: Generic Connector with Partner Implementation**

- **Apsis now provides**: An API spec/contract that external systems must fulfill
- **External system responsibility**: Implement the generic connector interface on their side
- **Apsis responsibility**: Create a config file for the partner
- **Result**: Apsis controls the interface, partners own implementation details

**Example Partners**:
- **Sideshop**: Implements the generic connector for Dynamics 365. No hourly fees; they bill customers directly (e.g., €200/month) and retain integration revenue. Can build custom features beyond the generic interface.
- **Intermail**: Implements generic connector for their Relation Plus loyalty platform. Drives customers to Apsis and takes whatever integration revenue they want.

**Legacy Systems Still Running Old Model**:
- Dynamics 365 (old supplier contract still active, though Apsis no longer pays for SLA)
- Various historical connectors

---

## Inbound Synchronization: Syncing Data from External Systems to Apsis

### User Journey: Integration Setup and Field Mapping

When a user installs an integration (demonstrated with FC Enterprise 12.1):

1. **Installation wizard** provides URL and API key for the external environment
2. **Field Mapping page** allows mapping external system fields to Apsis profile attributes
3. **Backend service**: Integration Manager (ECS task, formerly Lambda)
   - Fetches schema from external system using appropriate connector library
   - Returns field list to frontend for user to map
   - Stores mappings in a dedicated mappings manager database

### Three Types of Inbound Mappings

**1. Field Mappings**
- Maps attributes from external system (e.g., "Email" in CRM) to profile fields in Audience
- Determined by user configuration during setup

**2. Subscription/Consent Mappings**
- Maps consent basis from external system (either a dedicated consent resource or fields on the contact)
- **Legacy terminology**: "Virtual consent" (fields) vs. "native consent" (dedicated resource)
- **Current approach**: Generic connector doesn't distinguish; just gets what external system provides

**3. Sync Conditions**
- Boolean conditions that determine whether a record should sync
- Example: Only sync if `active == true`
- If inbound message fails condition, record is not synced
- If outbound and condition fails, profile may be deleted (per feature plan)

### Real-Time Synchronization: Delta Sync

When an external system has webhooks enabled and mappings are saved:

1. **External system** sends real-time updates when contacts are created or updated
2. **Two possible endpoints**:
   - Official Apsis endpoint (rare) — goes through Delta Single Manager
   - Custom webhook endpoint provided by Apsis (common) — Apsis manages the URL without involving customers

3. **Processing pipeline**:
   - Updates → **SQS FIFO queue** (message group by CRM ID)
   - Consumed by **Delta Sync Worker** (ECS task)
   - Worker asks Mappings Manager for applicable mappings (with caching)
   - Worker sends mapped message to Audience

### Full Synchronization: Bulk Syncing All Records

**When used**: Initial setup after configuration; can also be triggered manually or on schedule

**Architecture**: Spins up ad-hoc infrastructure for the sync job (considered overkill, but necessary due to historical queuing issues)

**Components**:
- **Full Sync Manager** (ECS task, formerly Lambda)
  - Receives sync request
  - Spins up producer and consumer infrastructure
  - Monitors job progress

- **Full Sync Producer**
  - Uses connector library to fetch paginated list of contacts from external system
  - Translates to Justin's expected format
  - Places data on full sync queue

- **Full Sync Consumer**
  - Consumes queue messages one by one (or 10 by 10)
  - Asks Mappings Manager for field and consent mappings
  - Sends mapped profiles to Audience

### Handling Data Consistency During Full Sync

**The Problem**: If a contact is updated in the CRM while a full sync is running, how do we ensure eventual consistency?

**Solution**: 
- During full sync, all real-time delta sync messages for that integration are **buffered** in a separate queue
- The buffer checks: Is a full sync currently running for this integration?
- If yes, delay the message by setting its visibility timeout
- Once full sync completes, process all buffered messages in order

**Why This Approach**: 
- Cannot simply reject messages ("send this later") due to historical reasons that were never addressed
- Ensures eventual consistency: if Benjamin is updated multiple times during full sync, all updates are processed afterwards in correct order
- Prevents scenarios where a newer version of the data is processed before an older version

**Progress Reporting**:
- Job shows "Finalizing" until Audience's ingestion delay catches up to the point in time when the last message was written
- Only then shows as "Successful"
- This ensures data is actually written before UI reports completion

### Static and Dynamic Profile Lists

In external CRM systems, there are two types of contact lists:

- **Static lists**: Manually maintained by humans
- **Dynamic lists**: Automatically maintained by queries (like Apsis segments)

Apsis abstracts both as:
- **Static profile list** (terminology comes from external system; FC Enterprise calls it "profile")
- **Dynamic profile list** (FC Enterprise calls it "query")

**Important Note**: This terminology is confusing because "profile" has a different meaning in Apsis context. No time has been allocated to fix this terminology issue.

**How list import works**:
- Import does **not** create profiles; that requires full sync or real-time sync
- Import **adds tags** to existing profiles based on list membership
- Example: "Newsletter" list import adds "Newsletter" tag to all profiles on that list
- Tag is removed if profile is no longer on the list in subsequent imports
- Can be set to run on a recurring schedule (currently 6:00 AM via CloudWatch events)

**Infrastructure**:
- **Profile List Sync Manager** receives requests
- Puts jobs on profile list sync queue
- **Consumer** spins up one worker per job (one at a time per integration to avoid DDOSing customer systems)
- **Worker** fetches all contacts on the list, holds in memory, sends tag request to Audience
- **Note**: This has caused problems in the past (failures with no queue). Now more stable.

---

## Outbound Synchronization: Syncing from Apsis to External Systems

### Activities That Can Be Synced Outbound

Nearly every activity type in Apsis can be synced to external systems:

- **Email** (opens, clicks, deliveries, etc.)
- **SMS** (deliveries, etc.)
- **Forms** (submissions)
- **Marketing Automation flows** (entry, exit events)
- **Event tool activities** (invites, registrations, confirmations)
- **Marketing Automation notes** (custom notes created in workflows)

**Cannot sync**: Events themselves, surveys

### Creating and Syncing an Email Activity

**User Flow**:

1. User creates email activity in Apsis
2. **Frontend checks**: Are there installed integrations that can sync emails?
3. If yes, shows **CRM Sync tab** with checkbox option
4. User clicks "Sync to [CRM System]"
5. When email is sent, system begins:
   - Creating corresponding record in external system
   - Transferring opens, clicks, delivered events to external system in batches

**Frontend Decision Logic**:
- Already asks integrations backend: "Is there an integration installed that can sync emails?"
- If yes, displays sync option
- User can check/uncheck per activity

### Outbound Processing Pipeline

**Challenge**: Avoid overwhelming external systems with one-by-one event transfers

**Solution**: Batch processing with buffering

**Flow**:

1. **Audience sends events** (sent, delivered, opened, clicked) to Apsis
2. **All Sub Queue** (Audience Subscription Queue)
   - FIFO SQS queue
   - Collects all event messages from Audience

3. **All Sub Worker** (Audience Subscription Worker, ECS task)
   - Verifies integration is installed and activity should be synced
   - Checks if this activity's sync is enabled for this integration
   - Places validated messages on **Kafka partition**
   - All messages mixed together for all installations

4. **Batch Production Worker** (ECS task)
   - Continuously polls Kafka partition
   - Groups messages by target integration
   - Waits until either:
     - Largest batch reaches **200KB** (stays well below SQS 256KB limit)
     - **1 second** has passed
   - Places ready batches on **SQS FIFO queue** (outbound queue)

5. **Outbound Worker** (ECS task)
   - Takes ready batches from outbound queue
   - Uses connector library to send to external system
   - If failed, places in **dead letter queue**
   - **Retry Driver Lambda** runs at 6:00 AM daily, redrives all dead-lettered messages

**Batching Rationale**: 
- Could generate 1000+ batches for a million events, but no problems observed
- However, switching to a different queue type has been on the improvement list
- 200KB chosen because it stays well below SQS 256KB message limit

### Consent Synchronization (Bidirectional)

Unlike attributes (one-directional from CRM to Apsis), **consent is bidirectional**:

**Inbound**: CRM consent maps to Audience subscription status
**Outbound**: When user opts out in Apsis, sync back to CRM

**Outbound consent flow**:
- Opt-in/opt-out messages go to All Sub Queue (bypasses batching)
- All Sub Worker processes them one-by-one (low volume)
- Sent to outbound queue
- Outbound Worker sends singular messages to external system
- Goes through dead-letter/retry carousel if failed

**Important Caveat** [Benjamin Nouhaag]: Attributes are **one-directional only** — from CRM to Apsis, not vice versa. This is by historical customer preference; they view the CRM as the data master.

### Dead Letter Queue and Message Retries

**Current behavior**:
- Failed outbound messages go to dead letter queue
- Retry Driver Lambda redrives **daily at 6:00 AM**
- Retries happen **indefinitely** (no limit — room for improvement)

**Gotcha with consent retries**:
- When redriving dead-lettered messages, **do NOT redrive opt-in (true) messages**
- Only redrive opt-out (false) messages
- Reason: Redrive loses eventual consistency guarantees. Playing it safe.

**Gotcha with batch retries**:
- Batches also lose consistency when redriven
- **External systems are contractually required** to handle this
- Mechanism: Each batch gets a unique ID; external system tracks processed batches
- If batch is redriven and already processed, external system should deduplicate

**Operational Practice** [Benjamin Nouhaag, citing experience]:
- When integration problems were frequent, team held **weekly operational meetings on Mondays**
- Reviewed customer integration problems and dead-letter queue contents
- Identified patterns and prevention strategies
- Recommended continuing this if problems accumulate

**Historical note**: Team has followed up on issues reported 2-4 weeks prior, then eventually stopped reporting them.

### Outbound Consent Writing

When a profile opts out in Apsis and consent must be written back to CRM:

- Consent message placed on All Sub Queue
- Bypasses batch production (opt-out messages are low-volume)
- Singular messages processed by outbound worker
- Sent directly to external system
- Goes through dead-letter/retry carousel

### Field Mappings for Outbound Data

**Current limitation** [Benjamin Nouhaag]: Only inbound field mappings are fully supported.

**The Gap**: Forms and some outbound activities
- Users create forms in Apsis and map form fields to profile attributes
- Ideally, could also map form fields directly to CRM fields
- **Current behavior**: Form submission events only sync profile attributes that have outbound mappings
- Data directly from form submission is **not** synced to CRM

**Why this gap exists**: 
- Implemented in 2022
- Various political and organizational reasons prevented full bidirectional mapping
- **CRM teams have complained** about this limitation
- Would require UI and logic to support form-specific outbound mappings

**Feature Flag Control**:
- Outbound mappings support depends on integration capabilities
- **Integration Manager checks**:
  - Does Apsis support this feature for this CRM system?
  - Does this specific CRM instance support it?
- Different instances of same CRM might support different features (e.g., production vs. development environment)
- Feature visibility and availability is determined by:
  - Connector specification (what Apsis allows per CRM system)
  - External system response (what this specific instance supports)

### Marketing Automation Notes and CRM Integration

Users can create MA notes that trigger CRM activities. [Benjamin demonstrated a workflow example]:

**Scenario**: Event invitation follow-up
- Listen for someone being invited to an event
- Wait 2 days
- Check: did they confirm attendance?
- If yes: do nothing (they confirmed)
- If no: create a CRM task for sales to call them

**Implementation**:
- Marketing Automation note uses **integration node** to create CRM record
- Each person reaching the node generates a message on All Sub Queue
- Goes through batch production dance (necessary because MA can trigger for 500,000 people at once)
- Sent to external system in batches
- External system creates corresponding CRM record (task, opportunity, etc.)

**Configuration**:
- All fields and settings are based on external system's schema
- Can set attributes like urgency, assignment, type, etc.
- These vary by external system

---

## Architectural Services and Components

### Core Services Overview

**Integration Manager** (ECS task, formerly Lambda)
- Fetches and translates schemas from external systems
- Routes requests to appropriate connector library
- Serves schema dropdown data to frontend

**Mappings Manager**
- Stores and validates field mappings, consent mappings, sync conditions
- Ensures mappings don't create loops ("Justin's laws")
- Has own database with dedicated user access to mappings table
- Caching layer with invalidation on updates

**Delta Sync Worker** (ECS task)
- Consumes real-time messages from All Sub Queue
- Fetches applicable mappings for account
- Sends mapped profiles to Audience

**Full Sync Manager** (ECS task)
- Manages full sync jobs
- Spins up producer and consumer infrastructure on-demand
- Monitors progress
- Tracks finalizing state until Audience ingestion catches up

**Outbound Manager**
- [Mentioned but not detailed in this session]
- Acts as bridge between frontend and CRM system
- Coordinates with Integration Manager for validations

**All Sub Worker** (ECS task)
- Validates that integrations are installed
- Verifies activities should sync
- Routes messages to Kafka for batching

**Batch Production Worker** (ECS task)
- Polls Kafka partition
- Groups messages by integration
- Implements 200KB or 1-second batching logic

**Outbound Worker** (ECS task)
- Takes ready batches from queue
- Uses connector library to send to external system
- Routes failed messages to dead-letter queue

**Retry Driver Lambda**
- Runs daily at 6:00 AM
- Redrives all dead-lettered messages (with caveats for consent)

---

## Multiple Integrations Per Account and Single CRM Limitation

### The Core Limitation

**Only one CRM integration can be active per section at a time.**

[Michal Rosikiewicz asked about multiple integrations on one account]

**Why this limitation exists**:
- All integrations, regardless of system, map to a single field: **CRM ID** (the "CRM_ID" key space in Audience)
- This was decided early on and is now deeply baked into the system
- [Benjamin emphasizes]: "We hate this so much, but we've never had time to fix it"

**Historical context**: This was a "super urgent release something now" situation (nicknamed "Jonas")

**Workaround in place**:
- Apsis bootstraps a **unique key space per integration** (e.g., FSC_Enterprise_ID, Dynamics_ID)
- Profiles from different integrations don't mix
- But still internally maps to the CRM_ID field (a hack)

**Impact on UI**:
- Integration page shows "You can only have one CRM integration at a time"
- This prevents users from syncing email activity to multiple CRM systems simultaneously
- Each section can only choose one CRM as the target for outbound syncing

### Which Integrations Are Not Affected

**Non-CRM integrations are not subject to this limitation**:
- Playable (third-party marketing platform)
- Other SaaS platforms with no CRM ID dependency

**Why they're unaffected**:
- They use **generic connector** with custom key spaces
- No dependency on CRM_ID field
- Each integration gets its own dedicated key space
- User just copies/pastes API credentials and key space to the external system

**Migration Path**:
- Anything implemented after 2020 was designed with this limitation in mind
- Integrations made in 2019 or earlier are dependent on CRM_ID
- A **big migration project** would be needed to fix this
- [Benjamin suggests]: Could be a good joint project with the new team to tackle in the fall

---

## Unified Data: Advanced Cross-System Querying

### The Problem Unified Data Solves

**Use Case** [Benjamin presented Paul from "AM I Events"]:

Paul wants to reach out to **CEOs of European companies whose employees attended last year's expo.**

**Why this is hard with current Apsis model**:
- Apsis profiles are flat documents (single attributes per profile)
- External CRMs have **relational data structures** with multiple tables and relationships
- To find CEOs of companies with attendees, need to traverse:
  - Event → Attendee → Person → Role → Company → CEO
  - Each step is a join in CRM, but Apsis has no way to represent this

**The gap**:
- Apsis can sync individual person records
- But cannot sync relationships, roles, or company hierarchies
- Cannot represent "Elon Musk is CEO of Tesla AND CTO of The Boring Company" with different emails for each role

### Solution: Unified Data (HTTP/2 Streaming)

**Implemented for**: Maxo (email marketing tool in Apsis One), but reusable for other systems

**How it works**:

1. **Marketing team** creates advanced query in CRM (e.g., "CEOs of companies with expo attendees")
2. **Email tool** in Apsis needs to send to this audience
3. **Audience initiates HTTP/2 connection** to Integrations service
4. **Integrations** calls the connector library to fetch paginated results from CRM
5. **Data is streamed** (not all at once) from CRM to Integrations
6. **Integrations translates** to Apache Columnar format on-the-fly
7. **Data is streamed back** to Audience over HTTP/2 connection
8. **Audience joins** the external data with its internal exports
9. **Email personalization** can reference fields from both systems

**Why HTTP/2 is important**:
- Streaming protocol allows continuous data flow
- Prevents fetching millions of records at once
- Integrations fetches paginated data, converts to columnar format, pushes, repeats
- Only possible with HTTP/2 streaming, not regular HTTP request/response

**Architecture**:
- Pre-made queries in CRM system list which data they expose
- Each query has a "selected class" defining available fields
- Integrations acts as middleware between CRM and Athena (Apsis data warehouse)
- Athena can join external table data with internal profile exports

### Production Status and Limitations

**Current status**: 
- Implemented in fall/winter 2024
- Finished on time
- No customers have used it yet (Maxo product didn't bring in customer base)
- One beta customer was used, but not verified in large-scale production

**Assessment** [Benjamin]: 
- "I would not say it is really tried in actual usage"
- Successful sendings have been made
- One customer has used it
- Not yet proven stable across 100+ customer deployments

**Not heavily used means**:
- Should be operationally possible if needed
- Technical contract is documented with every step
- But real-world issues may not be discovered until customer usage

### Alternative Approaches Considered

**Historically**: Team attempted to flatten CRM data and send events to Apsis
- Example: Replicate "person's title at company" as an attribute on every profile
- **Problem**: Audience keeps full history of every attribute on every profile
- Replicating company data across millions of profiles would massively increase Audience bill
- **Why unified data is better**: Joins happen in Athena (data warehouse), not in profile storage

### Documentation and Future Use

- Comprehensive documentation exists describing full flow
- User experience steps detailed
- Technical contract between Justin, email tool, Audience, and CRM documented
- Every step includes API calls and example payloads
- Should be operationally maintainable if needed in future

---

## Partner and Supplier Models

### Shift from Supplier to Partner Strategy

**Historical approach**: Apsis paid suppliers (agencies, consultants) to build integrations
- Example: CRM Consultana for Microsoft Dynamics
- Paid by the hour
- Apsis funded SLA agreements
- Each feature addition required renegotiation and payment

**Current approach**: Partners implement and own integrations; Apsis provides interface
- Partners can charge customers directly
- Partners fund their own operations
- Apsis benefits from feature requests without paying for implementation
- More scalable for Apsis

### Definitions

**Supplier**: 
- Apsis pays them by the hour
- Apsis has SLA contracts
- Example: CRM Consultana (historical)
- Current status: No active supplier contracts (Apsis chose not to continue paying)

**Partner**:
- No payment from Apsis
- They bill customers directly (if at all)
- They implement Apsis's generic connector interface
- They own support and maintenance
- Examples:
  - **Sideshop**: Implements Dynamics 365, charges customers ~€200/month, retains revenue
  - **Intermail**: Implements generic connector for Relation Plus (loyalty platform) and Relation Plus CRM, drives customers to Apsis

### Contractual Documentation

**Standardized Supplier Agreement**:
- Benjamin worked with legal to develop
- Defines terms, SLAs, service levels
- Issue severity definitions mapped to Apsis internal P1-P4 system:
  - P1-P2: Critical
  - P3+: Non-critical
- Response time requirements
- Specifies how supplier should handle issues
- Available in documented template; all active suppliers should sign version or tailored variant

**Supplier Delivery Process**:
- Supplier walks Apsis through build process before delivery
- Must demonstrate:
  - 75% test code coverage (failing builds if not met)
  - Code quality standards
  - Documentation
- **Definition of Done** includes:
  - Detailed installation instructions
  - List of user roles required
  - Overview of all files created/modified (with exact paths)
  - Overview of database tables and entities created/altered
  - This ensures Apsis can answer customer questions

**Partnership Agreement**:
- Separate agreement for partners using generic connector
- Defines what partners can do
- Specifies Apsis termination rights
- Partners own everything (support, development, intellectual property)

### Key Documents and Resources

**Generic Connector Specification**:
- API contract that external systems must fulfill
- Latest versions maintained in S3 folder with domain name
- Manually updated as spec evolves

**Generic Connector Implementation Guide** (lengthy document):
- Detailed instructions for implementing connector
- Last updated: 2022 (outdated)
- Updated version exists from 2024 with "pretty big change"
- Benjamin committed to merging updates into main document by "tomorrow" (relative to session)
- Very useful for onboarding new partners

**Historical CMS Integrations** (Open Source):
- Drupal, FP Server, Sitecore
- Worked differently than current CRM model
- Provided segment dropdowns for content personalization
- Evaluated segments at render time
- Different architecture worth studying
- Repositories available to share

### Current Active Partners

[Michal Rosikiewicz asked for inventory of active agreements]

**Partners**:
1. **Sideshop**: Dynamics 365 CRM integration
2. **Intermail**: Relation Plus and Relation Plus CRM

**Suppliers**: None (Apsis discontinued paying for SLA agreements)

**Historical note**: Benjamin provided a spreadsheet to Felix before parental leave with full list; can be recreated from memory if needed.

### Managing Multiple Versions and Instances

**Challenge**: Same CRM system can have different versions across customers

**Example**: eDeal (often on-premise)
- Different customers run different versions
- Different versions of Apsis connector
- Customer A's version may support a feature that Customer B's version doesn't
- Apsis cannot just check "is this CRM type supported"

**Solution**:
- **Integration Manager checks both**:
  - Does Apsis support this feature for this CRM type? (integration code level)
  - Does this specific customer instance support it? (external system level)
- Feature visibility determined by both checks passing

**Feature Flags**:
- Apsis still uses feature flags for account-level feature enabling
- But no longer relies on back-office flags for availability
- Determination happens in:
  - Connector specification (what Apsis allows per CRM)
  - External system response (what this instance supports)

---

## Multiple Entity Types in External Systems

### Extending Beyond Single Contact Table

Current implementation has been shown with single mapping (contacts), but some integrations support multiple entity types:

**Example**: Could map both:
- **Contacts** table (primary)
- **Leads** table (secondary)

Each would appear as separate table in UI with its own mappings.

**Key Difference**: 
- Pre-2020 integrations: All entities map to single CRM_ID field (the limitation)
- Post-2020 integrations: Each entity gets **dedicated ID attribute** (e.g., Lead_ID, Contact_ID)
- This is the proper approach, but migration of older systems would be required

### Benefits of Multi-Entity Design

- Allows flexible handling of different record types
- Avoids forcing all records through single ID space
- Proper foundation for future feature development

---

## Infrastructure and Implementation Details

### Queueing and Message Processing

**SQS FIFO Queues**: Used throughout
- Delta sync queue (message group: CRM ID)
- Full sync queue
- Profile list sync queue
- Outbound queue
- Benefits: Ordering guarantees, exactly-once processing semantics

**Kafka**: Used for intermediate message batching
- Kafka partition collects messages from All Sub Worker
- Batch Production Worker polls this partition
- Messages mixed together across all installations
- Kafka provides efficient buffering before batching

**SQS 256KB Limit**: Design constraint
- Batch Production Worker keeps batches to 200KB
- Provides safety margin below SQS limit
- Single 1MB message could not be handled

### Infrastructure Patterns

**Ad-hoc vs. Always-On Services**:
- Full Sync Manager spins up producer/consumer infrastructure on-demand
- Considered "overkill" but necessary due to historical queuing issues
- Other services (Delta Sync Worker, Batch Production Worker, etc.) run as always-on ECS tasks
- Retry Driver runs on schedule (daily at 6:00 AM) as Lambda

**Caching Strategy**:
- Delta Sync Worker caches mappings from Mappings Manager
- Cache invalidated when mappings are updated
- Reduces repeated database queries during high-volume sync

### Connector Library Pattern

All communication with external systems goes through **connector libraries**:
- Isolated per external system type
- Contains system-specific API calls, field translations, error handling
- Swappable implementation
- Same interface across all connectors (from Justin perspective)

---

## Operational Concerns and Known Issues

### Dead Letter Queue and Retries

[Premanand Thangamani contributed]: "14 days" is the limit based on how long a message can stay in the queue before being dropped, but Apsis redrives **every day** (infinite retry attempts)

**Issue**: Infinite retries without a maximum attempt count creates unbounded retries

### Webhook Node in Marketing Automation

[Benjamin mentioned]: MA webhook node has similar problems to old integration approach
- Sends one-by-one to customers
- Frequently receives "too many requests" errors
- No batching or backoff logic
- Could benefit from unified batching like outbound event processing
- **Potential future improvement**: Reuse event batching infrastructure for MA webhook node

### Programmatic Event Subscription

[Benjamin noted]: 5-6 years ago, many requests for ability to subscribe to specific events via Apsis One API
- Current outbound event batching infrastructure could be generalized
- Could be reused if requirement comes back
- Worth considering for future API design

### Inconsistencies and Historical Debt

**Field mapping terminology**:
- "Profiles" in FC Enterprise context means "static profile list" (not same as Apsis profiles)
- Confusing and never resolved due to time constraints

**CRM ID field dependency**:
- Most problematic piece of historical technical debt
- Blocks single-section multi-CRM support
- Affects outbound syncing choices
- Would require significant migration

**Outbound form field mapping**:
- Only profile attributes can be synced from forms, not form submission data
- Feature requested by CRM teams
- Remains unimplemented

---

## Documentation and Handover

### Documents to Be Shared

Benjamin committed to providing:

1. **All presentation slides** (in multiple formats)
2. **Process and policy documents** (partner/supplier agreements, templates)
3. **Documentation for unified data** (technical contract, implementation guide)
4. **List of suppliers and partners** (with historical context)
5. **GitHub pull request** with generic connector documentation updates
6. **Docx files** to be uploaded to shared repository (pending storage location finalization)

### Outstanding Tasks

- [Benjamin]: Merge 2024 updates into Generic Connector Implementation Guide (deadline: "tomorrow")
- [Benjamin]: Provide GitHub PR for review and merge (pending help from team member to get Felix's review)
- [Benjamin]: Move editable documentation files to GitHub repository or OneDrive shared folder
- [Team member assigned]: Request feedback from Felix on pending PR

### Resources Available

- S3 folder with domain name containing latest API specs
- GitHub repositories for legacy open-source CMS integrations (Drupal, FP Server, Sitecore)
- Standardized supplier contract and SLA template
- Supplier delivery process checklist
- Partnership agreement template

---

## Key Takeaways

1. **Integration architecture is built on modularity**: Justin middleware separates generic infrastructure from system-specific connector libraries. This allows scaling to new systems without reimplementing core logic.

2. **Standardization is central to operations**: The team deliberately chose standardized use cases (e.g., abandoned cart) over feature richness to make monitoring and SLA guarantees viable when external systems are involved.

3. **The strategic shift to partners is fundamental**: Moving from paying suppliers hourly to partnering with systems-specific experts (who charge customers directly) reduces Apsis's operational burden and accelerates feature development.

4. **Inbound and outbound are fundamentally asymmetric**: Inbound syncs attributes and consent; outbound syncs events and consent only. Attributes are intentionally one-directional (from CRM to Apsis) per customer preference.

5. **Batching and buffering prevent operational disasters**: Buffering delta sync during full sync, batching outbound events into 200KB chunks, and daily retries of failed messages are what keep the system stable at scale.

6. **The CRM_ID field is legacy debt with ongoing impact**: All CRM integrations map to a single CRM_ID field, blocking multi-CRM setups per section. Migration would require significant engineering effort but should be considered.

7. **Unified data is a powerful but unproven capability**: The HTTP/2 streaming approach to joining external relational data with Apsis profiles exists and is documented, but hasn't been stress-tested in production at scale. Could be valuable for future use cases.

8. **Documentation, agreements, and processes matter**: The standardized supplier/partner agreements, delivery process checklists, and generic connector specs are the foundation for sustainable partnerships. These should be maintained and enforced going forward.

---

## Unresolved Questions and Action Items

### For the Incoming Team to Decide

1. **CRM_ID migration**: Should the team prioritize fixing the single-CRM-per-section limitation? This would require migrating all pre-2020 integrations to use per-entity ID fields.

2. **Unified data investment**: Should unified data be promoted and tested with new customers, or remain a specialized feature?

3. **Documentation location**: Should handover documentation go on GitHub wiki, shared OneDrive folder, or both?

4. **MA webhook node batching**: Should the outbound event batching infrastructure be refactored and reused for the MA webhook node?

5. **Operational cadence**: Should the team resume weekly operational reviews of integration problems and dead-letter queue contents?

### Immediate Action Items

1. **Benjamin to provide**: PR for generic connector documentation (needs review/merge help)
2. **Benjamin to provide**: Merged updates to Generic Connector Implementation Guide
3. **Benjamin to provide**: Supplier/partner inventory spreadsheet
4. **Benjamin to provide**: All presentation slides and handover documents
5. **Team to determine**: Where to store editable versions of handover documents
6. **Team member assigned**: Request Felix review of pending GitHub PR

### Items Benjamin Noted as Potential Future Projects

- **Fall project candidate**: Joint migration of CRM_ID dependency to per-entity ID fields
- **Ongoing**: Monitor and update generic connector spec and implementation guide as requirements evolve