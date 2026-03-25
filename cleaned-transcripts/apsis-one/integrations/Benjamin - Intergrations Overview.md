```yaml
---
source_file: Benjamin - Intergrations Overview.txt
domain: Apsis One Integrations
topics: 
  - Integration Architecture Overview
  - Inbound Synchronization (Profiles, Consent, Field Mappings)
  - Outbound Synchronization (Activities, Events, Consent)
  - Full Sync and Delta Sync Operations
  - Profile List and Query Management
  - Unified Data for Complex CRM Queries
  - Supplier and Partner Strategy
  - Generic Connector Implementation
speakers:
  - Benjamin Nouhaag (Integration Team Lead)
  - Lukasz Grabowski
  - Michal Rosikiewicz
  - Premanand Thangamani
  - Speaker 1 (likely team lead/manager)
  - Eric (mentioned, technical deep dive owner)
key_components:
  - Justin (Integration Middleware)
  - Integration Manager (ECS task, schema fetching)
  - Delta Sync Worker (SQS consumer)
  - Mappings Manager (mapping validation and storage)
  - Full Sync Producer/Consumer
  - Batch Production Worker (Kafka-based)
  - Outbound Manager
  - Audience Subscription Queue (AllSub Queue)
  - Athena (profile store for Maxo)
  - React (real-time profile store)
session_type: knowledge-transfer
subdomains:
  - Architecture
  - Microsoft Dynamics Integration
  - Efficy Enterprise 12.1 Integration
  - Lead creation
  - Duplicate profiles in Apsis
---

# Apsis One Integrations Overview – Knowledge Transfer Session

## Session Overview

This comprehensive knowledge transfer session covers the conceptual and architectural foundations of the Apsis One Integrations domain. Benjamin Nouhaag, the integration team lead, walks through why the integration layer was built, the core philosophy of standardization and modularity, and how data flows through inbound and outbound synchronization pipelines. The session includes live UI demonstrations of field mappings, full sync operations, and marketing automation integration nodes. Special attention is given to historical challenges, the shift from supplier-controlled connectors to partner-driven generic connectors, and emerging capabilities like unified data for complex CRM queries. This is essential background for teams taking over long-running integration infrastructure.

---

## Context and Historical Mission

### Why Audience (Profile Cloud) Was Built

In 2013–2014, companies had numerous disconnected systems (CRM, ecommerce, etc.) that IT departments were forced to integrate manually. This resulted in:
- Systems perpetually out of sync
- Data lagging or missing when needed
- No holistic customer view

**Profile Cloud** (later rebranded as **Audience** in Apsis One) was the solution: a central middleware to consolidate data from all external systems and provide a single source of truth.

### Integration Team Mission

The integrations team's job is to:
1. **Ensure arrows exist** for as many external systems as possible (maintain bi-directional sync)
2. **Provide the same SLA** (Service Level Agreement) as Apsis One does for the rest of the platform:
   - Uptime guarantees
   - 24/7 pager duty
   - Consistent reliability

This is challenging because logic always resides partly outside Apsis One's control. [Benjamin Nouhaag]

---

## Core Architecture Philosophy

### Standardization and Modularity

Rather than building custom integrations for every system, the team adopted a **standardized use-case approach**:
- Identify common patterns across system types (CRM, ecommerce, etc.)
- Build generic infrastructure for those patterns
- Create lightweight connector libraries for system-specific communication

**Example**: Historically, the team built a standardized **abandoned cart infrastructure** that:
- Ran once per hour
- Called standardized functionality to fetch carts from all connected systems
- Threw abandoned cart events into Audience for marketing automation to react to
- Only required minimal, system-specific logic for each ecommerce platform

This approach minimizes external system dependencies and maximizes what the team can monitor and support. [Benjamin Nouhaag]

### Justin – The Integration Middleware

The core middleware is named **Justin** (after Justin Timberlake – "the greatest integration system you'll ever hear on the radio"). 

Justin provides:
- **Generic infrastructure** for common sync patterns
- **Connector libraries** for easy communication with each external system

**Design principle**: If the UI allows mapping profile fields to CRM contact fields, the connector library should only need to:
1. Make an HTTP request to fetch all fields from the external system
2. Translate them to Justin's expected format
3. Return them to Justin
4. Let Justin serve them to the front end

This modularity means **minimal code duplication** and **easy extensibility**. [Benjamin Nouhaag]

---

## Historical Evolution of Connector Strategy

### Phase 1: Supplier-Controlled, Specification-Driven

The team initially worked with external suppliers (contractors) who built and maintained plugins on customer systems.

**Example: Microsoft Dynamics 365**
- Worked with **CRM Consultana** (Stockholm-based contractor)
- CRM Consultana built a solution file with Webhooks that Apsis could subscribe to
- Paid hourly fees and maintained SLAs with CRM Consultana

**Problems with this approach**:
- Every new feature required coordination with the supplier
- Expensive and slow iteration
- Edge cases emerged constantly (e.g., in Lime CRM, you can opt out of marketing OR remove the relationship – both should be opt-outs, but webhooks don't distinguish)
- One or two days before launch, requests would require three-week fixes
- Unwieldy process when scaling across multiple systems (Dynamics, SuperOffice, Lime, Shopify, etc.)

[Benjamin Nouhaag]

### Phase 2: Generic Connector with Partner Model

The team shifted to a **generic connector specification** model:
- Apsis publishes a standardized **contract/API spec** that external systems must fulfill
- Partners implement this spec on their side
- Apsis provides only a **config file** for integration

**Example: Efficy Enterprise 12.1, SuperOffice**
- Partners implement the generic connector interface
- Apsis maintains the interface; partners maintain their implementations
- Partners can build additional features outside the generic spec

This allows:
- **Faster iteration** (no supplier coordination needed for generic features)
- **Partner revenue model** (they bill customers; they get revenue)
- **Scalability** (partners own support for their implementations)

[Benjamin Nouhaag]

### Active Partners vs. Suppliers

**Current landscape**:
- **Zero active suppliers** with formal contracts (the team chose to stop paying hourly)
- **Two active partners**:
  - **Sideshop**: Provides Dynamics 365 CRM integration via generic connector (planned as full replacement for legacy CRM Consultana integration)
  - **Extramanial**: Provides loyalty-related integrations

**Historical systems** (no longer actively supported but documentation preserved):
- Drupal, EP Server, Sitecore (CMS integrations with segment dropdown functionality)
- Shopify, Relation Plus (ecommerce; now discontinued)

[Benjamin Nouhaag, Michal Rosikiewicz]

---

## Inbound Synchronization

### Overview

Inbound means syncing data **from external systems into Audience** (Apsis profiles). This includes:
- Contact/profile creation and updates
- Field/attribute mapping
- Consent/subscription information
- Segment/list membership (tagging)

### UI Flow: Integration Installation and Field Mapping

When a user installs an integration (e.g., Efficy Enterprise 12.1):

1. **Initial Setup Wizard**
   - User provides URL and API key for the external system
   - System displays a configuration page

2. **Field Mapping Page**
   - User maps external system fields to Audience profile attributes
   - Example: `CRM_Field_Name` → `Audience Attribute Name` (auto-mapped where possible)
   - The field dropdown is rendered by calling **Integration Manager**

[Benjamin Nouhaag]

### Integration Manager Service

**Role**: Serves as the bridge for all basic operations when interacting with external systems.

**How field mapping works** (step-by-step):

1. Front end requests available fields from the external system
2. Front end specifies which integration (Dynamics, Efficy, etc.)
3. Integration Manager **selects the appropriate connector library** based on the integration type
4. Connector library fetches the schema from the external system via its native API
5. Schema is returned to Integration Manager
6. Integration Manager returns it to the front end as a dropdown list

**Implementation**: Currently runs as an **ever-running ECS task** (previously Lambda).

[Benjamin Nouhaag]

### Mappings Manager and Justin's Laws

**Role**: Stores field and consent mappings; enforces consistency rules.

**Database**: Has its own database user with access only to the mappings table.

**"Justin's Laws"**: A set of consistency rules (detailed by Eric in deeper technical sessions) that prevent:
- Circular/recursive mappings
- Invalid mapping configurations
- Other logical errors

**When mappings are saved**:
- Mappings Manager validates them
- Mappings are stored
- Webhooks are subscribed to (or updated if already subscribed) for all relevant entities in the external system
- The external system will now send real-time updates to Audience when records change

[Benjamin Nouhaag]

### Real-Time Data Ingestion: Delta Sync

**Webhook endpoints**:
- Some systems send updates through an **official Apsis endpoint** exposed by Audience
- Others send through a **custom webhook endpoint** provided by Apsis without requiring end-user involvement

**Processing pipeline**:

```
External System → Webhook → SQS Queue (FIFO, grouped by CRM ID)
                      ↓
                  Delta Sync Worker (ECS task, consumes 1-10 messages at a time)
                      ↓
                  - Ask Mappings Manager for active mappings (cached, invalidated on updates)
                  - Apply sync conditions (if configured)
                  - Transform data using field mappings
                  ↓
                  Audience (profile ingestion)
```

**Key behavior**: Messages are consumed one at a time (or in small batches, e.g., 10 at a time) to maintain ordering and eventual consistency. [Benjamin Nouhaag]

### Consent/Subscription Mapping

In the external system, consent can be represented in two ways:

1. **Dedicated consent resource** – A specific entity/table that represents opt-in/opt-out/unsubscribe status
2. **Boolean fields on the contact** – Fields like `OptInMarketing` directly on the contact record

**How it works**:

1. User navigates to the subscription mapping section
2. Integration Manager queries the external system: "What consent basis options do you have?"
3. Audience queries itself: "What topics/subscription lists do we have?"
4. User maps external consent options to Audience topics
5. Mappings Manager stores these mappings
6. During webhook processing, the mappings are used to determine how to update profile consent status

In legacy implementations, these were called:
- **Virtual consent** (for boolean fields)
- **Native consent** (for dedicated consent resources)

In the generic connector model, this distinction doesn't matter. [Benjamin Nouhaag]

### Full Sync Operation

**Purpose**: Download all contacts from the external system and sync them into Audience, respecting all field mappings, consent mappings, and sync conditions.

**When to use**: After initial setup, or when doing a large data migration.

**How it works**:

1. User clicks **Start New Sync** in the UI
2. Request is posted to **Full Sync Manager** (ECS task, formerly Lambda)
3. Full Sync Manager spins up an **entire stack on-demand** for this specific job:
   - One **Full Sync Producer** machine (downloads paginated data from external system)
   - Full Sync Queue (SQS or similar)
   - One or more **Full Sync Consumer** machines

4. **Producer behavior**:
   - Uses connector library to fetch paginated list of all contacts from the external system
   - Returns data in Justin's expected format
   - Puts messages on the full sync queue

5. **Consumer behavior**:
   - Same message processing as delta sync (ask Mappings Manager, apply conditions, transform)
   - Puts profiles into Audience

6. **Completion signaling**:
   - Producer is done when it receives no more paginated results
   - Consumer is done when queue is empty
   - Full Sync Manager waits for both, then waits for **Audience ingestion delay** to catch up before marking sync as **successful**

**UI states**:
- **Downloading**: Profiles have been downloaded from the external system
- **Finalized**: Profiles are being processed (may not yet appear in Audience)
- **Successful**: All profiles processed and ingested into Audience

[Benjamin Nouhaag]

### Buffering During Full Sync

**Problem**: While a full sync is running, real-time delta sync messages arrive from the external system. If processed immediately, they may conflict with full sync data (eventual consistency issue).

**Solution**: 
- When a delta sync message arrives during an active full sync for that integration, it is **buffered in a queue**
- A consumer checks if a full sync is ongoing; if yes, it **sets the visibility timeout** on the message to delay it
- After the full sync completes, buffered messages are processed in the order they arrived

**Rationale**: This ensures that if multiple messages arrive for the same profile during a full sync, they are processed afterward in the correct sequence without risk of overwrites or consistency violations.

**Note**: This is a workaround born from historical reasons; [Benjamin Nouhaag] notes the team wishes it could simply say "wait, full sync is ongoing," but that capability was never built due to time/bandwidth constraints.

[Benjamin Nouhaag]

### Profile Lists and Queries

**Terminology challenge** (Efficy Enterprise calls them "profiles"; Apsis One profiles are different):

In most CRM systems, there are two types of contact lists:
- **Static lists**: Human-maintained membership
- **Dynamic lists** (queries): Automatically maintained by a CRM rule/filter (like Apsis segments)

**How Apsis abstracts this**:
- Static → "Static Profile List"
- Dynamic → "Dynamic Profile List" or "Query"

**What importing a list does** (Efficy calls it "Query" or "Profile"):
- Does **NOT** create profiles in Audience
- **Adds a tag** to each contact in that list
- When the import is re-run, the tag is removed from contacts no longer on the list

**Example workflow**:
1. User configures an import of the "Newsletter" list from Efficy
2. Every contact on the Efficy "Newsletter" list gets tagged with `Newsletter` in Audience
3. Next week, two contacts leave the list
4. Re-run the import → those two lose the `Newsletter` tag

**Recurring imports**:
- Currently trigger at a fixed schedule (e.g., 6:00 AM every morning via CloudWatch event)
- Could be extended to support custom schedules without much effort

**Implementation**:
- CloudWatch event triggers a **Profile List Sync Lambda** (or similar worker)
- Puts sync job entries on a queue
- A consumer spins up one worker per job (sequential, not parallel, to avoid DDOSing customers)
- Worker fetches all contacts on the list and sends tag request to Audience

**Historical issues**: 
- These workers were failure-prone before edge cases were addressed
- No queuing mechanism meant failed syncs had no retry buffer
- [Benjamin Nouhaag] notes these are now "quite stable"

[Benjamin Nouhaag]

### Sync Conditions

**Purpose**: Filter which records get synced based on field values.

**Example**: 
- Only sync active contacts: Add condition `Active == true`
- Only sync from a specific region: Add condition `Region == "EMEA"`

**Behavior**:
- Stored in Mappings Manager
- Checked during every delta sync and full sync message processing
- If condition evaluates to false, record is **skipped**
- If condition evaluates to false and the profile already exists in Audience, the profile can optionally be **deleted** (deactivation feature)

**Configuration**: In the UI, users select a field, choose an operator (equals, contains, etc.), and provide a value.

[Benjamin Nouhaag]

---

## Outbound Synchronization

### Overview

Outbound means syncing data **from Audience into external systems**. This includes:
- Sending activity events (email sends, opens, clicks, deliveries)
- Sending form submissions
- Sending SMS/push delivery status
- Sending consent changes (opt-outs)

**Important limitation**: Attribute/profile changes do NOT sync outbound. Only consent is bidirectional. [Premanand Thangamani, Benjamin Nouhaag]

**Rationale**: Customers historically want the CRM to be the data master; Apsis One is a read-only consumer for attributes but owns consent state.

### Activity Sync to External Systems

When a user creates and sends a marketing activity (email, SMS, form, event, or MA flow):

1. **Sync tab appears** on the activity editor
   - "Sync to [Integration Name]?" checkbox
   - Integration-specific settings (if any)
   - Front end queries Integration Manager to see which integrations can handle this activity type

2. **On send** (for email, SMS) or **on action** (for forms, flows):
   - Outbound Manager creates a corresponding record in the external system
   - As events arrive (opens, clicks, deliveries), they are batched and sent to the external system

[Benjamin Nouhaag]

### Event Batching Pipeline

**Problem**: Without batching, a single email to 1 million people receiving an open would generate 1 million messages to the external system, causing:
- Performance issues on the CRM
- Potential rate-limit violations ("Too many requests" errors)

**Solution**: Batch events before sending.

**Pipeline**:

```
Audience Events → Audience Subscription Queue (AllSub Queue, FIFO SQS)
                     ↓
                Audience Subscription Worker (AllSub Worker, ECS task)
                  - Verify integration is installed for this account
                  - Verify this activity should be synced
                  - Put message on Kafka partition (all messages for all integrations mixed)
                     ↓
                Batch Production Worker (ECS task, consumes Kafka)
                  - Groups messages by integration and activity
                  - Batches are ready when:
                    • Batch size exceeds 200 KB, OR
                    • 1 second has passed since last poll
                  - Puts ready batches on SQS FIFO queue
                     ↓
                Outbound Worker (ECS task, consumes SQS)
                  - Takes a ready batch
                  - Uses connector library to send batch to external system
                  - If success: done
                  - If failure: goes to Dead Letter Queue
```

**Example of batching logic**:
```
Events from Audience (mixed, in order):
  A1, A2, A3, A4  (for integration A)
  B1, B2, B3      (for integration B)
  C1, C2          (for integration C)
  A5, A6          (for integration A again)

After batching:
  Batch 1: [A1, A2, A3, A4, A5, A6]  → Send to Integration A
  Batch 2: [B1, B2, B3]                → Send to Integration B
  Batch 3: [C1, C2]                    → Send to Integration C
```

**250 KB SQS limit**: Batches stay well below the 256 KB SQS message size limit by capping at 200 KB. This is a known limitation of the current approach and is on the team's improvement list.

[Benjamin Nouhaag, Premanand Thangamani]

### Dead Letter Queue and Retry Strategy

**When outbound fails**:
1. Message is put on **Dead Letter Queue** after a few retries
2. **Retry Driver Lambda** runs at 6:00 AM every morning
3. Retry Driver pulls all messages from Dead Letter Queue and puts them back on the outbound queue

**Retry limits**: 
- Originally, the team would infinitely retry failed messages
- [Premanand Thangamani] notes there is a **14-day SQS message limit** and retries happen daily, so messages are eventually expired
- **Area for improvement**: The team has discussed adding an explicit retry limit to prevent infinite redriving

[Benjamin Nouhaag, Premanand Thangamani]

### Alerts and Operational Monitoring

**Questions raised**: [Lukasz Grabowski] asks if there are alerts on the Dead Letter Queue and if there are situations where messages don't get processed.

**Answered**: Yes, unprocessed messages happen regularly. Reasons include:
- CRM system bugs or timeouts
- Network problems
- Incorrectly configured consent data in Apsis (causes processing failures)
- Customer's external system infrastructure limitations

**Consent retry strategy**: 
- When redriving a failed consent message, the team only redrives **opt-out messages** (`OptIn == false`)
- They do **not** redrive opt-in messages (`OptIn == true`)
- **Rationale**: Once messages are redriven, eventual consistency is lost. Playing it safe means only syncing opt-outs (more critical from a compliance standpoint)

**Batch consistency**: 
- Batches also lose consistency on redrive
- **Mitigated by**: External systems in the contract are required to track batch IDs and detect duplicate processing
- Each batch gets a unique ID; external systems should only process each ID once

[Benjamin Nouhaag, Lukasz Grabowski, Premanand Thangamani]

### Operational Best Practice

[Benjamin Nouhaag] recommends:
- Hold a **Monday operational meeting** to review:
  - Customer integration problems
  - Contents of Dead Letter Queue
  - Patterns in failed syncs
- This surfaces chronic issues and prevents small problems from becoming customer pain points

In previous years, the team reported bugs that were ignored for weeks; this meeting helps keep visibility.

[Benjamin Nouhaag]

### Consent Synchronization (Bidirectional)

The team syncs **consent changes back** to the external system when a profile opts out in Apsis One:

**Process**:
1. Profile opts out in Audience
2. Consent change event is put on **AllSub Queue**
3. AllSub Worker passes it through (bypasses batching, since consent changes are low-volume)
4. Message goes directly to **Outbound Queue** (not through Batch Production)
5. Outbound Worker sends the consent change to the external system
6. If it fails, it goes to Dead Letter Queue and is redriven daily

**Why batching is bypassed**: Consent changes are infrequent (thousands per day, not millions), so individual processing is fine.

[Benjamin Nouhaag]

### Outbound Field Mappings (Complex Case)

**Context**: Some integrations allow users to map Audience profile attributes to external system fields for outbound syncing.

**Limitation (as of 2022)**: While users can map profile attributes for outbound, this was not available at the form-submission level. Instead:

1. User creates a form with fields (email, first name, etc.)
2. Form fields are mapped to Audience profile attributes
3. On form submission, the system **does not use form-specific mappings** to the CRM
4. Instead, the system uses pre-configured **Outbound Mappings** (reverse of field mappings)
5. These outbound mappings take data from the Audience profile, not the form submission

**Desired behavior** (not implemented):
- Map each form's fields directly to CRM fields (form-specific)
- So different forms could send different data to the CRM

**Why not implemented**: 
- Described as having "political reasons" for the decision
- Would require significant rework of the outbound processing pipeline

**Workaround**: Use profile-level mappings and ensure form data is written to profile attributes first.

[Benjamin Nouhaag, Lukasz Grabowski, Michal Rosikiewicz]

### Marketing Automation Integration Nodes

In marketing automation (MA) flows, users can create **CRM interaction nodes** that create/update records in the external system.

**Example use case**:
1. Listen for event "Person invited to event"
2. Wait 2 days
3. Check if person has confirmed attendance
4. If no: Create a task in the CRM for a salesperson to call them
5. If yes: Do nothing

**Configuration**:
- Node type: CRM system (e.g., "Efficy Enterprise")
- Record type: What to create (e.g., Task)
- Field values: Pulled from CRM schema (e.g., "urgency," "assigned to")
- All field values are configuration-based on the external system's schema

**Processing**:
1. When a profile reaches the CRM node in a flow, a message is put on **AllSub Queue** (tagged with the node interaction)
2. Message is processed through the full batching pipeline (Batch Production, Outbound Queue, Outbound Worker)
3. Batching is important because MA flows can trigger 500,000 people at a specific time (e.g., noon every day)

**Error handling**: If the external system schema can't be fetched (e.g., integration is misconfigured), an error badge appears on the node.

[Benjamin Nouhaag]

### Webhook Node Issue

[Benjamin Nouhaag] notes that the MA **webhook node** (which lets users call arbitrary external APIs) currently sends requests one-at-a-time without any batching or queuing logic. This has historically caused:
- "Too many requests" errors on customer systems
- No built-in recovery mechanism

**Future opportunity**: The batching infrastructure described here could be extended to webhook nodes, providing customers with more reliable integrations.

[Benjamin Nouhaag]

### One API Request Opportunity

[Benjamin Nouhaag] mentions that 5–6 years ago, there were many requests for **programmatic event subscriptions** via the Apsis One API. This could be solved with a more generic version of the current batching/event subscription infrastructure, if the requirement resurfaces.

---

## Integration Limitations and Architectural Debt

### CRM ID Field Limitation

**Problem**: The system was originally built to allow **one CRM integration per section**. Both the Audience profile attributes and the integration logic rely on mapping to a single **CRM ID field**.

**Why this matters**:
- If you try to connect two different CRM systems to the same section, profiles from both will map to the same `CRM_ID`, causing conflicts
- This violates the team's principle of using system-specific key spaces

**Workaround** (current): Only one CRM integration per section is allowed in the UI.

**How newer integrations work around this**:
- Integrations built after ~2020 bootstrap a **system-specific ID field** for each integration and table
- Example: `Efficy_Contact_ID` for Efficy, `Dynamics_Contact_ID` for Dynamics, etc.
- This avoids the CRM ID collision problem

**Migration opportunity**: For older integrations still using the shared CRM ID field, a migration could be done to establish system-specific ID fields. [Benjamin Nouhaag] suggests this could be a good joint project in the fall.

**Affected integrations**: Dynamics 365 (legacy, pre-2020), and any other old system-specific integration.

**Not affected**: Generic connector integrations (Efficy Enterprise 12.1, SuperOffice via partners) and third-party integrations (Playable, etc.), which can have multiple simultaneous connections.

[Benjamin Nouhaag, Michal Rosikiewicz]

### Outbound Form Mapping Limitation

As discussed above: Forms can't have their fields mapped directly to the CRM; only profile attributes are sent outbound.

[Benjamin Nouhaag]

### No Attribute Changes Outbound

Apsis One does not sync profile attribute changes back to the CRM, only consent. Customers expect the CRM to be the data master.

[Benjamin Nouhaag, Premanand Thangamani]

---

## Supplier and Partner Relationship Model

### Strategy Shift

Historically, Apsis worked with **suppliers** (contractors paid hourly to build integrations). The team realized this was:
- **Slow**: Feature requests required supplier coordination and payment
- **Expensive**: Hourly billing, SLA contracts, ongoing support fees
- **Unscalable**: Couldn't easily add features across multiple systems without supplier involvement

**New model**: Work with **partners** instead.

### Supplier vs. Partner

| Aspect | Supplier | Partner |
|--------|----------|---------|
| **Payment** | Apsis pays hourly | Partner bills customers directly |
| **Feature control** | Apsis owns features, pays for changes | Partner owns features, can innovate freely |
| **Support** | Apsis provides SLA | Partner provides support (or customer agrees to limitations) |
| **Revenue** | Cost to Apsis | Revenue for partner |
| **Scalability** | Hard (adds cost) | Easy (partner handles more customers) |

**Example transition**:
- **Old**: Apsis paid CRM Consultana to maintain Dynamics 365 integration, paid SLAs
- **New**: Sideshop implements Dynamics 365 via generic connector, bills customers directly, gets revenue

[Benjamin Nouhaag]

### Generic Connector Specification

For partners to work independently, Apsis publishes:

1. **Connector API Spec** (technical contract)
   - Describes what the external system must provide
   - Input/output formats
   - Webhook requirements
   - Example payloads

2. **Connector Implementation Guide** (step-by-step for partners)
   - How to implement the spec
   - Code examples
   - Testing guidance
   - 75% code test coverage requirement

3. **Partnership Agreement** (legal)
   - Rights and obligations
   - Support responsibilities
   - Intellectual property

All documents are versioned and stored in an **S3 folder with a domain name** assigned to it. Customers/partners always download the latest spec from there.

[Benjamin Nouhaag]

### Supplier Contracts and SLAs

For situations where Apsis does work with a supplier:

1. **Standardized contract template** (developed with Legal)
   - Negotiated and refined over multiple engagements
   - Covers key protections Apsis discovered it needed

2. **SLA template** (hour-to-hour customizable)
   - Issue severity definitions (map to internal P1, P2, P3)
   - Response time commitments for each severity
   - Uptime guarantees
   - Standard for all suppliers

3. **Supplier Delivery Process** (definition of done)
   - Walk-through of build process with Apsis
   - Proof of 75% test code coverage (enforced at build time)
   - Detailed documentation:
     - Installation instructions
     - User roles required
     - All files created/modified with exact file paths
     - All database tables/entities created/altered
   - This ensures Apsis can answer customer questions about what the integration touches

[Benjamin Nouhaag]

### Current Active Relationships

**Partners** (actively working, generating revenue):
- **Sideshop**: Dynamics 365 CRM integration (generic connector)
- **Extramanial**: Loyalty platform integration (partially custom, partially generic connector)

**Suppliers**: 
- None active (team decided to shift away from paying hourly)

**Historical suppliers** (no longer active):
- CRM Consultana: Dynamics 365 (legacy, being migrated to Sideshop)

**Note**: [Benjamin Nouhaag] mentions that the commercial organization has not been following the migration plan to move all Dynamics customers to Sideshop. This is a gap between the technology/operations strategy and sales reality.

[Benjamin Nouhaag, Michal Rosikiewicz]

### Documentation Handover Items

[Benjamin Nouhaag] commits to handing over:
- Supplier/partner agreement templates
- SLA template
- Supplier delivery process document
- Partnership agreement
- Generic connector API spec (latest version)
- Generic connector implementation guide

All to be uploaded to a shared GitHub repository or OneDrive folder for the new team to maintain.

[Benjamin Nouhaag, Speaker 1]

---

## CMS Integrations (Historical)

### Overview

Before the current focus on CRM and ecommerce integrations, Apsis integrated with CMS platforms: **Drupal**, **EP Server**, and **Sitecore**.

### How CMS Integrations Worked

These were simpler than CRM integrations:
1. CMS user edits content
2. User gets a **dropdown of segments** in the CMS UI
3. Content can be configured to show only to users in that segment
4. When a request comes in, the CMS asks Apsis: "Is this user in segment X?"
5. Apsis evaluates the segment and returns true/false
6. CMS shows/hides content accordingly

### Key Difference from Current Model

- No profile data is synced from CMS to Apsis
- No external data is synced from Apsis to CMS
- It's purely **segment evaluation for content personalization**

### Web Key Space Caution

An important detail: Nothing should ever be whitelisted for the **web key space** in Apsis. The web key space is reserved for real-time web personalization, and allowing CMS (or other external systems) to write to it can cause conflicts.

Instead, a separate key space is created for each CMS integration to avoid cross-system pollution.

[Benjamin Nouhaag]

### Current Status

All CMS integrations have been rendered **open source** and documentation has been provided to the new team. If CMS integration requests resurface, the team has reference implementations to work from.

[Benjamin Nouhaag]

---

## Unified Data – Advanced CRM Querying

### The Problem Unified Data Solves

**Scenario**: Paul, head of sales at AM I Events, wants to send an email to CEOs of companies whose employees attended an event last year.

**Data challenge**: 
- The CRM has **multiple tables with relationships**:
  - Event table (events attended)
  - Attendee table (people who attended events)
  - Person table (individuals)
  - Role table (person's job titles/roles in companies)
  - Company table (organizations)

- To find "CEOs of companies with attendees" requires **joining across tables**:
  ```
  Event → Attendee → Person → Role → Company → (filter to CEO role)
  ```

- **Apsis's flat profile model can't natively represent this**:
  - Apsis syncs one person per profile
  - It can't represent "Person X has multiple roles in multiple companies"
  - It can't represent "Person X attended Event Y"

**Email personalization needed**:
> "Hello [CEO Name], I noticed that [Attendee Name] from your company [Company Name] had a blast at [Event Name] last year. How do you feel about sending more reps this year?"

Apsis's current attribute-based system can't construct this without flattening the data in a way that would explode profile size and Athena (Apsis's data warehouse) costs.

[Benjamin Nouhaag]

### Unified Data Architecture

**High-level idea**: When an email is sent via Maxo (the email marketing tool), Apsis can request **complex data** from the CRM in real-time and merge it with profile data before personalizing the email.

**Components**:

1. **Audience** (profile store): Stores base profile attributes
2. **Athena** (data warehouse): Stores profile history and exports
3. **React** (real-time store): Stores real-time page view events
4. **External CRM system**: Stores complex relational data

**Flow**:

1. User creates email campaign in Maxo with a query:
   - "CEOs of companies whose employees attended [Event] last year"

2. User specifies pre-made **CRM queries** (defined in the CRM) that can be included in email personalization:
   - Query A: "For this person, find their company's CEO"
   - Query B: "For this person, find events they/their company attended"

3. When **Maxo exports profiles** for email sending:
   - Maxo tells Audience to do an export for a specific segment
   - Audience returns paginated profile data in **Apache Columnar (Parquet)** format
   - For **each query**, Audience opens an **HTTP/2 streaming connection** to Integrations
   - Integrations acts as a **middle man** between the CRM and Audience:
     - Fetches paginated data from the CRM query
     - Translates it to Apache Columnar format on-the-fly
     - **Streams it back** to Audience (doesn't buffer everything in memory)
   - Athena **joins** Audience data with the streamed CRM query data
   - Result: A virtual table with base profile attributes + related CRM data

4. Email template uses the joined data:
   ```
   Hello [CEOProfile.FirstName],
   I noticed [AttendeeProfile.FirstName] from [CompanyProfile.Name] 
   had a blast at [EventProfile.Name] last year...
   ```

### Why HTTP/2 Streaming

**Problem**: If you have 2 million potential recipients, you could get 2 million rows back from a CRM query. Loading all of that into memory before streaming would be infeasible.

**Solution**: Use HTTP/2 streaming to push data incrementally:
1. Integrations fetches first page from CRM
2. Translates to Columnar format
3. **Streams** it to Audience immediately
4. Fetches next page
5. Streams next page
6. Continues until done

This keeps memory usage low and query latency reasonable.

[Benjamin Nouhaag]

### Pre-Made Queries

- These are **pre-built queries in the CRM** (defined by the CRM administrator)
- They return specific columns that the CRM admin deems safe for export
- The CRM admin defines the query; Integrations fetches the selected columns
- Partners can define which queries are available for email personalization

**Example query in Efficy**:
```sql
SELECT 
  PersonID, 
  Person.FirstName, 
  Person.LastName, 
  Company.Name, 
  Company.Country
FROM Person
JOIN Company ON Person.CompanyID = Company.ID
WHERE Role = 'CEO'
```

[Benjamin Nouhaag]

### Production Status and Caution

**State**: Feature is implemented and has been tested in the **product suite project** (fall/winter 2024).

**Test coverage**: 
- All functionality completed on time
- Successful test sendings in controlled environments
- One customer (Berto) has used it

**Production readiness**: **Not heavily proven** 
- Only one customer has actually used it in anger
- Not stress-tested with 100+ customers
- Not proven to be stable at scale

**Recommendation**: 
- Good to know about for future requests
- Don't assume it's battle-tested like core integrations
- May need defensive monitoring and support when customers start using it

[Benjamin Nouhaag, Speaker 1]

### Documentation

Comprehensive documentation exists that covers:
- Complete user experience flows
- Technical contract (every step, every API call, example payloads)
- How it works between Justin, Email Tool, Audience, and the CRM

This should allow the team to operate it even without deep hands-on experience.

[Benjamin Nouhaag]

### Scaling Attributes Without Unified Data

**Historical approach** (rejected): Flatten CRM data onto every profile.
- Example: Add `[Company]_[CEO]_[FirstName]` attribute to every person
- **Problem**: Explodes Athena database size and costs
- **Why rejected**: Too expensive

**Unified Data approach**: Keep data in the CRM, query it on-demand during email send.
- **Benefit**: No Athena cost increase
- **Trade-off**: Slightly slower email generation (must query CRM during send)

[Benjamin Nouhaag]

---

## Key Takeaways

### Architecture Principles

1. **Standardization over custom work**: Build generic infrastructure for common patterns; add system-specific logic only at the edges (connector libraries).

2. **Modularity**: Each component (Integration Manager, Mappings Manager, Full Sync, Delta Sync, Outbound Worker) has a specific responsibility and can be maintained/replaced independently.

3. **Minimize external dependencies**: By standardizing how external systems behave and owning validation (Mappings Manager), the team can guarantee Apsis SLAs even when external systems have issues.

4. **Asynchronous, event-driven processing**: Queue-based pipelines (SQS, Kafka) allow the team to buffer spikes, retry failures, and decouple components.

### Inbound = Profiles, Attributes, Consent, Lists

- **Profiles and attributes** are one-way: from external system to Apsis
- **Consent** is bidirectional: Apsis syncs opt-outs back to the CRM
- **Lists/tags** allow external systems to segment profiles without exporting full attribute data
- Sync conditions allow filtering which records get imported

### Outbound = Events, Not Attributes

- Activities (emails, SMS, forms, MA flow steps) generate events
- Events are **batched** for efficiency
- **Consent changes** are sent back immediately (not batched)
- External systems are responsible for deduplicating batch IDs to handle retries

### Partner Model Scales Better Than Supplier Model

- Partners own their integration; Apsis publishes a spec
- Partners can iterate quickly without asking Apsis for permission
- Partners get revenue; Apsis gets lower support costs
- Generic connector allows partners to build on top of a standard interface

### Unified Data Enables Complex CRM Personalization

- When external systems have rich relational data, Unified Data allows Apsis to query and join that data at send time
- HTTP/2 streaming keeps memory usage low
- Still experimental; not widely proven yet

### Known Technical Debt

- **CRM ID field collision**: One CRM per section due to shared ID field (affects pre-2020 integrations)
- **Form outbound mapping**: Can't map form fields to CRM; must use profile attributes
- **Attribute sync**: Only one-way, external system to Apsis
- **SQS batch size limit**: 200 KB cap to fit in 256 KB limit is somewhat arbitrary
- **Infinite retries**: Dead Letter Queue doesn't have an explicit retry limit (relies on SQS 14-day expiry)
- **MA webhook node**: Sends one-at-a-time without batching; could benefit from event infrastructure

### Operational Best Practices

- Run a weekly operational meeting to review Dead Letter Queue and integration problems
- Use standardized contracts and SLA templates with any suppliers
- Require 75% test code coverage and detailed documentation from suppliers
- Test all integrations with multiple instances (different CRM versions may support different features)

---

## Unresolved Questions and Action Items

### Action Items for Benjamin

- [ ] Merge the pending PR for process/policy documentation (needs feedback from Felix, manager to expedite)
- [ ] Move editable versions of supplier/partner agreements and SLA templates to GitHub repo (decision: upload as .docx or use GitHub-native docs)
- [ ] Send the pending PR link to Speaker 1/Mashek for review
- [ ] Provide list of historical suppliers and partners with contact information
- [ ] Update Generic Connector API Spec and Implementation Guide (docs from 2022 and 2024 need merging; target: done by tomorrow)
- [ ] Upload all documentation to shared S3 folder or GitHub repo
- [ ] Provide references to open-source CMS integrations (Drupal, EP Server, Sitecore)

### Action Items for Receiving Team

- [ ] Review supplier/partner agreement templates and SLA template
- [ ] Decide on documentation format (GitHub wiki, .docx in OneDrive, etc.)
- [ ] Review Generic Connector API Spec and Implementation Guide
- [ ] Review Unified Data technical documentation
- [ ] Follow up with Benjamin or Eric for deeper dives on specific components (e.g., Justin's Laws, Mappings Manager validation, Unified Data HTTP/2 flow)
- [ ] Consider whether to migrate legacy (pre-2020) integrations away from shared CRM ID field

### Questions for Future Technical Deep Dives

1. What exactly are "Justin's Laws"? How does Mappings Manager prevent circular mappings?
2. Detailed flow of how sync conditions are evaluated and applied during delta/full sync
3. How are Kafka partitions organized? Why Kafka instead of pure SQS?
4. What triggers the Batch Production Worker to start pulling from Kafka? How is the 1-second timer and 200 KB threshold enforced?
5. Detailed walkthrough of Unified Data HTTP/2 streaming with actual API examples
6. How are feature flags used to control which operations are available per integration?
7. What happens if a full sync is interrupted mid-way? How does recovery work?
8. How are field transformations handled (e.g., date format conversions, string truncations)?

### Gaps Noted by the Team

1. **CRM ID migration**: Pre-2020 integrations still use shared CRM ID field; migration would unlock multiple CRM per section
2. **Commercial/sales misalignment**: Sales has not been migrating Dynamics customers to Sideshop as strategically planned
3. **Unified Data customer adoption**: Only one live customer; needs more proof points before being widely recommended
4. **Documentation version control**: Process and policy docs currently live in personal files; need to be moved to shared repo

---

## Appendix: System Components Summary

### Core Services

| Service | Type | Purpose |
|---------|------|---------|
| **Justin** | Middleware | Central integration platform; routes messages through pipelines |
| **Integration Manager** | ECS Task | Fetches schema, validates operations |
| **Mappings Manager** | ECS Task | Stores and validates field/consent mappings |
| **Delta Sync Worker** | ECS Task | Processes real-time webhook events from external systems |
| **Full Sync Manager** | ECS Task | Orchestrates on-demand full data sync operations |
| **Full Sync Producer** | ECS Task (on-demand) | Fetches paginated data from external system |
| **Full Sync Consumer** | ECS Task (on-demand) | Processes full sync data into Audience |
| **Audience Subscription Worker (AllSub)** | ECS Task | Routes outbound events to Kafka |
| **Batch Production Worker** | ECS Task | Batches events for efficient external system delivery |
| **Outbound Worker** | ECS Task | Sends batched/individual messages to external systems |
| **Retry Driver** | Lambda | Redrives failed messages from Dead Letter Queue (daily) |

### Queuing Systems

| Queue | Type | Purpose |
|-------|------|---------|
| **Delta Sync Queue** | SQS FIFO | Real-time webhook events, grouped by CRM ID |
| **Full Sync Queue** | SQS | On-demand full sync messages |
| **Profile List Sync Queue** | SQS | Static/dynamic list imports |
| **Audience Subscription Queue (AllSub)** | SQS FIFO | Outbound events to be batched |
| **Batch Production (Kafka)** | Kafka | Events grouped by integration, waiting for batching threshold |
| **Outbound Queue** | SQS FIFO | Ready-to-send batches |
| **Dead Letter Queue** | SQS | Failed outbound messages awaiting retry |
| **Delta Sync Buffering Queue** | SQS | Messages buffered while full sync is ongoing |

### Connectors

| Connector | Type | Current Status |
|-----------|------|-----------------|
| **Microsoft Dynamics 365** | Supplier (legacy) + Partner (Sideshop) | Legacy being migrated to partner |
| **Efficy Enterprise 12.1** | Generic Connector (Partner) | Active, partner-maintained |
| **SuperOffice** | Generic Connector (Partner) | Active, partner-maintained |
| **Intermail Loyalty** | Custom (Partner) | Active |
| **Relation Plus** | Generic Connector (Partner) | Coming soon / active |
| **Playable** | Third-party | Active (passive, no Apsis code) |
| **Lime CRM** | Legacy | Historical (discontinued) |
| **Shopify** | Legacy ecommerce | Historical (discontinued) |
| **Drupal, EP Server, Sitecore** | CMS | Historical (open-sourced) |

---

**End of Knowledge Transfer Transcript**

```