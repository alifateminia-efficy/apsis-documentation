```yaml
source_file: Benjamin - Intergrations Overview.txt
domain: Apsis One Integrations
topics:
  - Architecture and design principles of Apsis One Integrations
  - Inbound synchronization (profile data, field mappings, consent)
  - Outbound synchronization (activities, events, marketing automation)
  - Integration infrastructure (Justin middleware, connector libraries)
  - Full sync and delta sync operations
  - Partner and supplier management models
  - Unified data for advanced CRM queries
  - Historical CMS integrations
speakers:
  - Benjamin Nouhaag (Integration Lead)
  - Lukasz Grabowski
  - Michal Rosikiewicz
  - Premanand Thangamani
  - Speaker 1 (unidentified)
  - Eric (mentioned, not speaking)
key_components:
  - Justin (integration middleware)
  - Integration Manager (service)
  - Mappings Manager (service)
  - Delta Sync Worker (ECS task)
  - Full Sync Manager (ECS task)
  - Outbound Manager
  - Audience Subscription Queue (SQS)
  - Batch Production Worker (Kafka/ECS)
  - Outbound Worker
  - Connector libraries (Microsoft Dynamics, Efficy Enterprise, generic)
  - Audience (profile store)
  - Athena (data warehouse)
  - React (real-time profile store)
session_type: knowledge-transfer
subdomains:
  - Microsoft Dynamics
  - Efficy Enterprise 12.0
  - Efficy Enterprise 12.1
  - Lead creation
  - Duplicate profiles in Apsis
---

## Session Overview

This knowledge transfer session provided a comprehensive conceptual and architectural overview of the Apsis One Integrations domain. Benjamin Nouhaag, the integration lead, walked through the design philosophy behind integrations (the "Justin" middleware), explained inbound and outbound synchronization flows, demonstrated the configuration UI, and discussed the strategic shift from supplier-based to partner-based integration models. The session also covered advanced features like unified data for complex CRM queries and historical context on deprecated integrations.

---

## Mission and Design Philosophy

### Why Integrations Exist

The original mission of Apsis One Integrations was born from a business problem observed in the 2013–2014 era. Companies ran numerous disconnected systems, and IT departments had to manually integrate them to provide a holistic customer view. Data was frequently out of sync, delayed, and missing when needed.

**Profile Cloud** (later renamed **Audience** in Apsis One) was positioned as middleware to solve this. The integrations team's mandate was to:

1. Ensure bidirectional communication arrows exist for as many external systems as possible
2. Provide the same Service Level Agreement (SLA) as Apsis One offers for its core platform
3. Maintain 24/7 pager duty and uptime guarantees despite external systems being outside direct control

> "At least that's the sales pitch of Profile Cloud. And the idea was, of course, to put something in the middle, which first was Profile Cloud and then became audience in Appsis One. It's basically the same thing. It's just a nice little upgrade."

### Standardization Strategy and Its Evolution

The original strategy was to standardize use cases by system type (CRM, ecommerce) and work with suppliers who understood individual systems deeply, keeping external system logic minimal. This allowed monitoring and ensuring uptime for as much of the chain as possible.

**Example**: The abandoned cart infrastructure ran hourly, calling standardized functionality to fetch carts from all integrated systems and throw abandoned cart events into Audience for marketing automation to react to. Only a small connector piece spoke to each specific system.

However, this approach proved "very unwieldy." External systems had undocumented quirks. For example:

> "In lime CRM you don't. You you can opt out of contact of marketing, but you can also just remove the relationship between contact and marketing, which should be an opt out but which is not covered by the works. So we ran into a lot of these crazy little situations very often like one or two days before launch and they required like a three-week fix."

**Current strategy**: Instead of the supplier building a system-specific solution, Apsis provides a **generic connector interface specification** and requires external systems to implement it. Apsis builds only a configuration file for each system. This puts responsibility on the external party to conform to the contract.

---

## The Justin Middleware

### Core Purpose

**Justin** is the integration middleware named after Justin Timberlake ("the greatest integration system you'll ever hear on the radio"). The name encapsulates its job: keeping everything in sync.

The design principle is **modularity**. Justin provides:
- Generic infrastructure for sync operations
- Connector libraries to ensure easy communication with each system

> "So if we have a UI where we can map fields on the profile or attributes on the profile in audience to fields on the contact in Microsoft Dynamics, then the only thing in the connector library that should be needed is a HTTP request that gets all the fields from Dynamics. Translates that to however Justin wants them to look, gives them back to Justin, and then Justin serves it to the front end."

### Evolution of Working Models

**Legacy approach (pre-2020)**: Apsis worked with suppliers like **Geran Kosnokana** (for Microsoft Dynamics 365) who built plugins with webhooks and provided API guidance. Connectors for Shopify, Lime CRM, and others were built based on API documentation without supplier involvement.

**Current approach**: System implementers (partners or suppliers) provide an API that conforms to Apsis specifications. Apsis creates a connector library to communicate with that API.

---

## Inbound Synchronization

### High-Level Flow

Inbound synchronization brings profile data, attributes, and consent information from external systems into Apsis One. The flow is initiated through the **Integrations UI** accessible at `/integrations` in the Apsis One interface.

### Field Mappings

**Integration Manager** (formerly a Lambda, now an ever-running ECS task) serves the field mapping UI. When rendering the field mappings dropdown:

1. User specifies an external system (e.g., `dynamics`, `fc_enterprise`, or generic connector)
2. Integration Manager fetches the schema from the external system using the appropriate connector library
3. Schema is returned to the front end UI, which displays available fields for mapping

Users can map any field from the external system to a profile attribute in Audience. The mapping is stored with its own database user and access to the mappings table, referred to as part of **Justin's laws** (consistency rules ensuring mappings don't create loops).

### Real-Time Synchronization (Delta Sync)

When an external system creates or updates a contact, it sends data via webhook:

1. Some systems send updates to an official endpoint exposed by Audience
2. Others send to custom webhook endpoints provided by Apsis (without the user being directly aware)
3. Updates are placed on a queue and consumed by the **Delta Sync Worker** (ECS task)
4. The worker queries the Mappings Manager (with caching) to determine which mappings apply to the account
5. A message is sent directly to Audience with the mapped data

**Key point**: When mappings are saved, webhooks are updated to reflect changes.

### Consent and Subscription Mappings

Consent mapping works similarly to field mapping:

1. Integration Manager asks what consent basis or topics exist in the external system
2. Apsis asks what topics exist in Audience
3. A mapping is created in the Mappings Manager
4. When processing webhooks, this mapping is used to translate external consent status to Audience topics

Consent resources can be:
- **Dedicated consent resources** (native to the system)
- **Virtual consent** (fields on the contact record, legacy terminology)
- **Native consent** (dedicated resource, modern terminology)

The generic connector doesn't distinguish; it returns whatever the system provides.

### Full Synchronization

[Benjamin: "Here we have our full sync, which we would usually recommend you to run after you've done your initial setup."]

Full sync downloads and syncs everything according to current mappings. The process:

1. User clicks "Start new sync" in the Full Sync UI
2. Infrastructure is spun up on-demand for this specific operation
3. A **Full Sync Producer** (ECS task) paginates all contacts from the external system using the connector library and places them in the expected Justin format
4. A **Full Sync Queue** buffers paginated data
5. A **Full Sync Consumer** (ECS task) processes the queue, queries the Mappings Manager for active mappings, and sends messages to Audience

**Important state management**: During a full sync, **delta sync messages are buffered** to maintain eventual consistency. A separate queue with visibility timeout handling delays real-time messages until the full sync completes. Messages are then processed in the order they arrived.

The UI displays three states:
- **Downloaded**: Profiles downloaded and processed
- **Finalizing**: Profiles are being processed, not yet in Audience
- **Successful**: The system confirms ingestion delay has caught up with the final write to Audience

### Sync Conditions

Sync conditions filter which records should be synchronized based on field values. For example:

```
active = true
```

This ensures only active contacts are synced. If a condition fails during processing:
- The record is ignored during full/delta sync
- If an inbound message indicates the condition is no longer met, the profile is deleted (feature flag controlled)

### Profile Lists (Queries and Segments)

External systems have two types of contact lists:

- **Static lists**: Manually maintained by users
- **Dynamic lists**: Automatically maintained by queries

Apsis abstracts these as **static profile lists** and **dynamic profile lists**.

**Example terminology issue**: In Efficy Enterprise, a dynamic list is called a "query" and a static list is called a "profile"—an unfortunate naming collision with Apsis profile objects.

**What importing a list does**: Importing a profile list does **not** create profiles; it tags existing ones. For example, importing a "Newsletter" list adds a `Newsletter` tag to all members. If a subsequent import removes someone, the tag is removed.

The import can be run once or on a recurring schedule (currently 6:00 AM via CloudWatch event triggering a Lambda). The system tracks which contacts received tags per operation to enable clean removal.

**Known issue**: These workers hold all contacts in memory (no queue), which has caused failures in the past due to size and lack of retry logic. Now more stable but remains on the improvement list.

---

## Outbound Synchronization

### Overview

Outbound sync sends activities and events created in Apsis One back to external systems. Supported activity types:

- Emails
- SMS
- Forms (subset of data)
- Marketing automation flows
- Events (invitations, registrations, confirmations)
- Marketing automation notes (custom notes)

**Not synced**: Surveys

### Email Outbound Flow (Example)

1. User creates an email in Apsis One
2. Front end asks Integration Manager whether an installed integration can sync emails
3. User checks the "Sync to [CRM]" option
4. When the email is sent, an **Outbound Manager** (service not shown in full diagram) bridges the front end and external system, instructing it to create a corresponding activity record
5. As delivery events arrive (sent, delivered, opened, clicked), they flow through the outbound processing pipeline

### Event Processing and Batching

Real-time events from Audience flow to the **Audience Subscription Queue (ASQ)**, an SQS FIFO queue:

1. **Audience Subscription Worker** (ASQ Worker, ECS task) validates:
   - Whether an integration is installed for this account
   - Whether this specific activity should be synced
2. Valid messages are placed on a **Kafka partition** (partitioned by integration/installation)
3. **Batch Production Worker** (ECS task) consumes the Kafka partition and creates batches:
   - Groups messages by integration
   - Batches are sent when either:
     - Largest batch exceeds 200 KB, or
     - 1 second of polling has elapsed
   - The 200 KB threshold is chosen to stay well below SQS's 256 KB message limit
4. **Outbound Worker** (ECS task) picks batches from the SQS FIFO queue, uses the connector library to send them to the external system

**Why batching**: Processing one-by-one could generate excessive traffic; batching reduces load. A single customer could generate 1000+ batches for a million events, but this has not caused problems in practice (though it remains on the improvement list).

### Dead Letter Queue and Retry Logic

If an outbound worker fails to send a batch:

1. After retries, it goes to a **Dead Letter Queue**
2. A **Retry Driver Lambda** runs at 6:00 AM daily and re-queues everything
3. Messages are retried **indefinitely** (current limitation)

[Premanand Thangamani notes: "We keep the violent" — context unclear, but Benjamin confirms infinite retries.]

### Consent Writing (Bidirectional)

When a profile opts out in Apsis One:

1. An opt-out message is placed on the **Audience Subscription Queue**
2. It bypasses batch production (added later, before batching existed)
3. It's picked up as a singular consent message and sent to the Outbound Worker
4. It goes through the retry carousel until successful

**Important caveat**: During redrive operations, **only opt-out messages (`opt_in = false`) are retried**, not opt-in messages. This is because redriving loses eventual consistency guarantees. The assumption is that opting out is the safer direction to re-sync. 

**Batches also lose consistency on redrive**, but the contract with external systems requires them to handle this using batch IDs—they must track which batches have already been processed.

### Consent Directionality

> "As then just said, like we are syncing the consent changes back to the CRM, but we do not sync attribute changes back to the CRM like the consent is bidirectional but the attributes. Only come to Apsis, but not from Apsis."

This is intentional and historical. Customers want the CRM to be the system of truth for attributes; only consent changes flow both directions.

### Form Data in Outbound

Forms present a special case. Users can map form fields to profile attributes on intake. However, for outbound, form submission data is **not** directly mapped to CRM fields.

Instead:
- An **Outbound Mappings** tab (feature-flag controlled) exists as the reverse of field mappings
- When a form is submitted and triggers outbound sync, the system uses these pre-defined outbound mappings (not the form-specific mappings)
- Data is pulled from the profile attributes, not directly from the form submission

This is a known limitation born from political/organizational reasons in 2022 and remains unresolved.

### Marketing Automation Notes and CRM Integration

Users can create marketing automation flows that interact with the CRM. Example:

**Use case**: Listen for event invitations, wait 2 days, check if the attendee confirmed. If not, create a task in the CRM for a salesperson to follow up.

Flow:
1. Event invitation received → wait 2 days
2. Check: `profile.event_confirmed = true` (or false)
3. If false, create CRM task with fields like urgency, task type, assigned owner
4. When the flow reaches the CRM integration node, a message is placed on the Audience Subscription Queue
5. The message goes through batch production and is sent to the external system

---

## Feature Flags and Integration Capabilities

### Feature Flags and Spec-Based Configuration

Different integrations support different features. Apsis controls what is enabled via:

1. **Integration specs** (stored in code/config) defining what CRM systems are allowed to do
2. **Instance-level checks** when a user installs, verifying the external system instance actually supports the feature
3. **Feature flags** (for per-account overrides, though less common)

Example: Microsoft Dynamics might support outbound mappings, but Salesforce might not. The UI hides tabs based on these capabilities.

For on-premise systems (e.g., E-deal), different customer instances may run different versions with different APSIS connector versions, so runtime checks confirm actual support.

### The CRM ID Field Problem

A major limitation prevents multiple CRM integrations per section:

> "We always map to the ******* CRM lighting field and we hate this. We hate this so much, but we've never had time to to fix it and that's why we won't let you install one integration. Per section."

**Root cause**: All inbound mappings historically mapped to a single `CRM ID` field in Audience, which created a key space collision when multiple CRM integrations existed on the same account. The field mapping UI maps all integrations to this field, preventing safe concurrent operation.

**Current workaround**: Only one CRM integration per section.

**Exception**: Third-party integrations (e.g., Playable) don't rely on the CRM ID field. They have their own bootstrapped key space and don't interact with CRM-specific infrastructure, so multiple can coexist.

**Partially resolved for multi-entity systems**: Integrations built after 2020 that sync multiple entity types (e.g., contacts and leads) generate separate ID attributes for each entity (e.g., `lead_id` vs. `contact_id`), avoiding the collision.

**Migration path**: A legacy migration would be needed to fix existing CRM integrations. Benjamin suggests this could be a good project for the fall as a joint effort with the incoming team.

---

## Partner and Supplier Models

### Strategic Shift: From Suppliers to Partners

Historically, Apsis worked with **suppliers** (paid contractors) who built system-specific integrations. This model proved cumbersome:

1. Adding features required modifying the connector library
2. Every feature addition required engaging the supplier, causing bloat and delays
3. Multiple systems (Dynamics, Salesforce, Lime CRM, Shopify, etc.) each required supplier negotiations

**New model**: Work with **partners** who implement the generic connector interface on their side. Partners:
- Implement the generic connector spec independently
- Own support and customer relationships
- Can build custom features outside the generic interface
- Generate their own revenue (not paid by Apsis)
- Submit feature requests to Apsis, which are considered at the platform level (not per-system)

**Current state**:
- **Active suppliers**: None (Apsis chose not to continue paying for support)
- **Active partners**: 
  - **Sideshop** (Microsoft Dynamics 365 CRM integration)
  - **Extramanial** (Loyalty and CRM services)

[Benjamin notes the commercial organization hasn't followed the plan to migrate all Dynamics customers from the legacy supplier model to Sideshop's partner-based model.]

### Supplier and Partner Documentation

#### Supplier Delivery Process

For any future supplier engagement, Apsis uses a standardized process document requiring:

1. **Test coverage**: 75% minimum test code coverage; builds fail without it
2. **Installation walkthrough**: Supplier demonstrates the build process before marking as done
3. **Detailed documentation**:
   - Exact file paths for each created file
   - Database tables and entities created/altered
   - User roles required
4. **Definition of Done**: Ensures future questions can be answered with precision

#### Supplier Service Level Agreement (SLA) Template

Apsis has a standardized SLA template covering:
- Support levels (matching P1, P2, P3 severity definitions)
- Response times
- Escalation procedures
- Maintenance expectations
- Uptime commitments

All active suppliers are expected to sign this agreement.

#### Partner Agreement

Partners implementing the generic connector sign a simpler agreement specifying:
- What they can build and modify
- Rights for Apsis to terminate the relationship
- Ownership of all code and support (theirs)
- Compliance with the generic connector specification

### Relevant Documentation

The following documents govern supplier/partner relationships and are maintained by Benjamin for handover:

- **Supplier delivery process document** (specifies test coverage, installation walkthrough, documentation requirements)
- **Supplier SLA template** (response times, severity definitions, uptime)
- **Partner agreement** (generic connector compliance, rights, support ownership)
- **Generic connector API specification** (latest version kept in an S3 folder)
- **Generic connector implementation guide** (last updated 2022, with 2024 updates pending merge)

All files are hosted in an S3 folder with domain-assigned URLs, allowing partners/suppliers to always access the latest specs.

---

## Historical and Deprecated Integrations

### CMS Integrations (Drupal, Sitecore, FP Server)

Apsis previously built CMS integrations that worked differently from CRM integrations:

1. Users got a dropdown of Audience segments when editing content
2. Users selected which segment(s) should see the content
3. A request was sent to the Audience segment evaluation API
4. There was key space juggling on the CMS side to prevent the web key space from being whitelisted

These integrations have been **rendered open source** and the documentation/repositories are available for reference.

### Ecommerce Integrations

Legacy ecommerce integrations (abandoned cart workflow, etc.) have been discontinued. The architecture diagram showing these has been kept for historical reference but is no longer active.

---

## Unified Data and Advanced Querying

### The Problem: Flattened Data vs. Relational Data

Audience maintains a flat profile model:
- One profile per person with attributes
- No structured relationships or role-based data
- Makes it impossible to model scenarios like "CEOs of companies whose employees attended an event last year"

External CRM systems, by contrast, have structured tables with relationships:
- Event → Attendance → Person → Role → Company

**Use case example** (Paul from AMI Events):
- Paul wants to contact CEOs of European companies whose staff attended his trade show last year
- His personalization request: "Hello [CEO name], I see [Attendee name] from [Company] had a blast at [Event] last year. How do you feel about sending more reps this year?"
- This requires joining across multiple tables and entities, impossible with Apsis's flat profile model

### Solution: Unified Data

Apsis built **Unified Data** for Maxo (product suite project, fall/winter 2024) to solve this. The solution:

1. Apsis hosts a **real-time data export service** that streams CRM query results
2. Users in the email tool create queries in the CRM (e.g., "CEOs of companies with attendees at event X")
3. When sending an email via a segment, the user can join that segment with a CRM query
4. Audience connects to Integrations via **HTTP/2 streaming** and:
   - Provides paginated CRM query results in Apache Columnar format
   - Streams the data back as Integrations fetches it (not all at once)
5. Audience joins the data internally and personalizes the email

**Why HTTP/2 streaming**: CRM queries can return millions of rows. Fetching all at once would be infeasible; streaming allows progressive batching.

**Current state**:
- Implemented and complete (finished on time for fall 2024 project)
- Tested with one Maxo customer
- **Not production-validated at scale** (no 100-customer deployment)
- Full technical documentation exists (payload examples, API contracts, user journey steps)

**Why mention now**: Future customers will ask for this capability. It exists and can be reused for any CRM system (nothing is Maxo-specific except the configuration).

### Implementation Details

The data flow involves:

- **Audience** (profile store with meta and React)
- **React** (real-time store; export too slow for batch use, takes 1 day)
- **Athena** (data warehouse via congestion pipeline; allows fast exports)
- **External CRM** (provides data via pre-made queries)
- **Integrations** (acts as middleware between CRM and Athena)

When a user sets up a unified data email:
1. Integrations opens an HTTP/2 connection to a specific CRM query
2. Integrations paginates data from the CRM, converts to Apache Columnar format
3. Data is streamed back to Audience in real time
4. Audience joins it with its own exports and personalizes the email

---

## Operational Concerns and Known Issues

### Dead Letter Queue Management

Dead letter queues accumulate failed messages. Apsis has monitored these via:

1. **Alerts** on the DLQ (currently in place)
2. **Operational review meetings** (formerly held every Monday) reviewing:
   - Customer integration problems
   - Contents of the DLQ
   - Patterns in recurring failures

Benjamin recommends this practice if operational problems increase.

### Why Messages End Up in DLQ

Messages fail for various reasons:

- **External system bugs or timeouts**
- **Network problems**
- **Incorrect customer data** (e.g., malformed consent attributes)
- **Inconsistent redrive behavior** (consent messages with `opt_in = true` are never retried, only opt-outs)

### Infinite Retries

Messages are currently retried infinitely. This is recognized as "a bit of a room for improvement" and should be addressed.

### Batch Retry Consistency Loss

When batches are redriven:
- They no longer guarantee eventual consistency
- External systems must track processed batch IDs to avoid duplication
- This is explicitly documented in agreements with external systems

### Webhook Node Inefficiency

The webhook node in Marketing Automation sends events one-by-one to customers, generating many requests. It would benefit from using a batching approach similar to the outbound event processing pipeline, but no such mechanism exists today.

### Future Opportunity: Programmatic Event Subscriptions

5–6 years ago, there were requests to subscribe to specific events programmatically via the One API. A generic, reusable batching service (like the current event processing pipeline) could be extended to support this if the requirement resurfaces.

---

## Key Takeaways

1. **Architecture is modular**: Justin middleware uses connector libraries to abstract system-specific logic, enabling scaling across many integrations.

2. **Inbound is attributes + consent**: External systems send profile data, field mappings translate to Audience attributes, consent mappings translate to topics. Full sync and delta sync maintain eventual consistency through buffering.

3. **Outbound is events + consent**: Activities and events are batched and streamed to external systems. Consent is bidirectional; attributes are unidirectional (external → Apsis only).

4. **CRM ID field is a technical debt**: Multiple CRM integrations per section are blocked by a legacy mapping architecture. A migration project is needed.

5. **Partner model is preferred**: Working with partners who implement the generic connector is more scalable than supplier-based, system-specific integrations.

6. **Unified data solves complex queries**: For scenarios requiring joins across CRM tables, Unified Data streams CRM query results via HTTP/2 to personalize emails. Fully implemented but not yet heavily used.

7. **Operational monitoring is essential**: Dead letter queues, regular review meetings, and clear SLA documentation with external parties reduce surprises.

8. **Documentation and specs are critical**: Supplier/partner agreements, generic connector specs, and delivery processes are standardized and should be read by the incoming team to understand the operating model.

---

## Unresolved Questions and Action Items

1. **Merge blocked PR for generic connector**: Benjamin submitted a PR in 2022; Felix provided feedback in spring 2024, but it hasn't been merged. Speaker 1 will ask Felix for review and facilitate the merge.

2. **Handover of supplier/partner agreements and docs**: Benjamin will move these from personal files to shared repositories/OneDrive. Formats: DOCX files initially, then consider GitHub wiki format for easier collaboration.

3. **Feature flag for outbound form mappings**: Michal asked whether a feature flag is needed to enable form-level outbound mapping. Benjamin explained it's per-integration in the spec file and checked at runtime based on instance capabilities.

4. **Structured inventory of active suppliers/partners**: Currently scattered (spreadsheet, email, memory). A comprehensive, maintainable list should be created.

5. **Historical supplier/partner contact information**: Michal requested contact information for companies that historically worked with Apsis (e.g., old Dynamics plugin developers). Benjamin offered to reconstruct from memory or existing notes.

6. **Update Unified Data documentation**: Benjamin noted the 2024 generic connector implementation guide updates haven't been merged into the primary documentation. These should be consolidated.

---

## Appendix: Connector Library and Generic Connector

### Connector Library Concept

When Integration Manager needs schema or data from an external system, it uses a connector library:

```
URL with system name (e.g., "dynamics", "fc_enterprise")
  ↓
Integration Manager selects appropriate connector library
  ↓
Connector library makes HTTP/REST calls to external system API
  ↓
Response converted to Justin format
  ↓
Returned to caller (e.g., UI front end)
```

### Generic Connector Spec

For systems implementing the generic connector, Apsis provides a specification defining:
- What attributes/fields the external system must expose
- What events it must support
- Pagination and error handling requirements
- Batch processing expectations
- The contract for handling idempotency (batch IDs)

The external system (partner) implements this API; Apsis builds a minimal configuration file to connect it.

### Legacy Supplier Dynamics Connector

The legacy Microsoft Dynamics connector was built by **CRM Consultana** (Stockholm contractor, paid hourly) and included:
- A plugin/solution file running on the customer's Dynamics instance
- Webhooks subscribed to contact create/update/delete events
- API guidance provided to Apsis
- SLA maintained by CRM Consultana (no longer paid)

This model is expensive and inflexible. The goal is to migrate all Dynamics customers to **Sideshop's generic connector implementation**, though commercial hasn't fully executed this plan.

---

## File Path and Configuration References

Benjamin mentioned the following notable file/path concepts (though specific paths were not detailed in this session):

- **Integration specs** in code/config defining per-system capabilities
- **Installer options** documentation defining which features are enabled per integration
- **S3 folder** hosting latest generic connector API specs and implementation guides
- **GitHub PR** with merged documentation and specifications (pending review)
- **OneDrive/Shared folder** for ongoing documentation

---

## Technical Terminology Reference

- **Justin**: The integration middleware
- **Audience**: Apsis One's profile store and CDP
- **Athena**: Data warehouse (ingestion from React via congestion pipeline)
- **React**: Real-time profile lookup store
- **Apsis One**: The parent marketing/CDP platform
- **Delta Sync**: Real-time synchronization of changes
- **Full Sync**: Complete download and synchronization of all records
- **ASQ**: Audience Subscription Queue (SQS FIFO queue)
- **DLQ**: Dead Letter Queue (failed messages)
- **SLA**: Service Level Agreement
- **CRM ID**: Legacy field used for CRM-specific profile mapping (source of technical debt)
- **Key space**: Namespace for profile identifiers per integration

---

**End of cleaned transcript**