---
source_file: Benjamin - Intergrations Overview.txt
domain: Integrations
topics: [Architecture Overview, Inbound Sync (CRM to Audience), Outbound Sync (Audience to CRM), Field Mappings, Consent/Subscription Mappings, Full Sync, Delta Sync, Profile Lists, Webhooks, Partner and Supplier Strategy, Unified Data, Historical Context]
speakers: [Benjamin Nouhaag, Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1 (Eric)]
key_components: [Justin (Integration Middleware), Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager, Outbound Manager, All Sub Worker, Batch Production Worker, Audience (Profile Store), Athena (Query Engine), React (Real-time Profile Store)]
session_type: knowledge-transfer
---

## Session Overview

Benjamin Nouhaag delivered a comprehensive conceptual overview of the Integrations domain within APSIS One, covering both historical context and current architecture. The session explained the original mission to synchronize customer data across disparate enterprise systems, the core middleware solution called **Justin**, and the complete flow of data in both inbound (CRM-to-Audience) and outbound (Audience-to-CRM) directions. Key discussion included the shift from supplier-controlled implementations to a partner-based generic connector model, the challenges of maintaining operational SLAs with external systems, and advanced capabilities like unified data querying for complex multi-table scenarios.

---

## Why the Integrations Domain Exists

### Historical Context and Original Problem

When the integration platform was being built in 2013–2014, enterprise IT departments faced a critical problem: **numerous disconnected systems with no holistic customer view**. Data was frequently out of sync, lagging, or unavailable when needed. The original solution, **Profile Cloud**, and its successor **Audience** in APSIS One, aimed to place a middleware layer in the center to synchronize all these systems and provide a unified customer profile.

### Mission and Service Level Commitments

The Integrations team's mission is to:

1. **Ensure connectivity** ("arrows exist") to as many systems as possible
2. **Provide the same SLA (Service Level Agreement)** that APSIS One provides for the rest of the platform—including guaranteed uptime and 24/7 pager duty support

This is operationally challenging because some logic always resides in external systems outside APSIS's control. To address this, the team adopted a **strategy of standardization**: create standardized use cases for different system types (e.g., CRM, eCommerce) and work only with suppliers who understand their specific system well, keeping external integration logic minimal to allow monitoring and SLA enforcement.

**Example of standardization:** The abandoned cart infrastructure once ran hourly, fetching carts from all integrated eCommerce systems via standardized functions and emitting events to Audience for marketing automation to act upon. Only a small connector piece was system-specific.

---

## Core Architecture: Justin and the Integration Middleware

### Design Principles

**Justin** (named after Justin Timberlake, "the greatest integration system you'll ever hear on the radio") is the core middleware layer. Its design is based on **modularity**:

- A **generic infrastructure** that handles common integration patterns
- **Connector libraries** for each external system, keeping system-specific logic isolated and minimal
- Clean separation: if there's a UI to map Audience profile attributes to Microsoft Dynamics contact fields, the connector library should only need to fetch those fields from Dynamics, translate them to Justin's internal format, and return them

### Evolution of Connector Strategy

**Early approach (Legacy):** The team worked directly with suppliers on system-specific implementations.
- Example: Microsoft Dynamics 365 had a custom solution file developed by CRM Consultana, with webhooks and API guidance
- Problems emerged: edge cases and undocumented system behaviors (e.g., LimeCRM's distinction between contact opt-out and removing the marketing relationship) were discovered 1–2 days before launch, requiring three-week fixes
- This became operationally unviable

**Current approach (Generic Connector):** Rather than APSIS owning the external system logic, partners implement a standardized contract (the generic connector API spec).
- APSIS provides only a configuration file for each partner
- Partners handle their system-specific implementation details
- This decouples APSIS releases from partner development cycles

---

## Inbound Sync: CRM-to-Audience Data Flow

### Field Mapping and Schema Discovery

When a user installs an integration and navigates to the **field mappings** page:

1. The **Integration Manager** (originally a Lambda, now an ever-running ECS task) is invoked
2. It uses the appropriate **connector library** based on the external system identifier in the request
3. For a standard CRM like Dynamics, it fetches the schema; for a generic connector system, it uses an internal mapping
4. The schema is returned to the front end, displaying available fields as a dropdown

### Mappings Manager and Justin's Laws

When the user saves mappings:

- The **Mappings Manager** validates that mappings don't create loops and enforces other constraints called **Justin's Laws**
- Mappings are stored in a dedicated database with its own user and table access
- Mappings are cached in the Delta Sync Worker and invalidated when updated

### Webhook Subscription and Real-Time Delta Sync

During installation:

1. The system subscribes to webhooks from the external system for contacts created/updated
2. Some systems send updates through an official APSIS endpoint (**Delta Sync Manager**); others use custom webhook endpoints provided by Integrations
3. Updates land on an **SQS FIFO queue** (message group: CRM ID)
4. The **Delta Sync Worker** (an ever-running ECS task) consumes messages:
   - Queries the Mappings Manager for active mappings
   - Applies field mappings to translate external data to Audience format
   - Sends mapped profiles to Audience in real-time

### Consent and Subscription Mappings

Consent is handled separately through **subscription mappings**:

- Maps consent basis/topics from the external system to Audience subscription topics
- Consent basis can be:
  - A dedicated consent resource (native to the external system)
  - Mapped to fields on the contact card (called "virtual consent" in legacy connectors)
- The generic connector approach doesn't distinguish—it accepts whatever the external system provides

### Full Sync: Bulk Contact Import

After initial setup, users run a **full sync** to import all existing contacts.

**Architecture:**

- **Full Sync Manager** (ECS task) spins up infrastructure on-demand for each sync job
- **Full Sync Producer** fetches paginated contact lists from the external system using the connector library, translating to Justin's expected format
- **Full Sync Queue** buffers the paginated results
- **Full Sync Consumer** applies the same mapping logic as delta sync, emitting profiles to Audience

**Complexity: Eventual Consistency During Full Sync**

A critical challenge: ensuring eventual consistency when delta sync messages arrive during a full sync. Because the system cannot pause incoming webhooks, it uses a buffering strategy:

1. **All Sub Queue** (Audience Subscription Queue) buffers incoming delta messages during full sync
2. An **All Sub Worker** checks: is a full sync ongoing for this integration?
3. If yes, it delays the message using SQS visibility timeout
4. After the full sync completes (producer and consumer both finish), the full sync manager waits for the **ingestion delay** to catch up with the last message timestamp
5. Only then does it mark the sync as "Successful"
6. Buffered delta messages are then processed in order

> This is a workaround the team wishes it didn't need. Ideally, they would halt delta processing during full sync, but historical constraints prevented that implementation.

### Full Sync States

- **Downloading:** Profiles have been downloaded from the external system
- **Finalizing:** Profiles are being processed; waiting for Audience ingestion delay to catch up
- **Successful:** All data written to Audience and ingestion is complete

### Profile Lists (Dynamic and Static)

Different CRM systems have different terminology for contact lists:
- **Static list:** Human-maintained (e.g., "Newsletter" list)
- **Dynamic list:** Auto-generated by a query (evaluated on-demand)

APSIS abstracts both as **profile lists** (static/dynamic). In FSC Enterprise, dynamic lists are called "queries" and static lists are called "profiles"—which is unfortunate terminology overlap with Audience profiles.

**Import functionality:**

- Importing a list does NOT create new profiles
- It adds a **tag** to profiles already synced via full sync or delta sync
- Recurring imports are scheduled via CloudWatch events (currently fixed at 6:00 AM)
- The **Profile List Sync Consumer** spins up one worker per job (sequential per integration to avoid DDOSing the customer)
- Workers fetch all contacts on a list and send a batch tag request to Audience

### Sync Conditions

Users can define conditional logic (e.g., "only sync active contacts"):

- Stored in the Mappings Manager
- Applied during message processing in both full sync and delta sync
- If a condition is not met, the message is ignored
- **Inbound delete:** If an inbound message violates a condition (e.g., `active == false`), the profile is also deleted from Audience (controlled by feature flag)

---

## Outbound Sync: Audience-to-CRM Data Flow

### Activity and Event Syncing

Users can sync the following outbound activities to external systems:
- Emails
- SMS
- Forms  
- Marketing Automation flow state changes
- Event tool registrations
- Marketing automation notes (CRM-specific)

**Cannot currently sync:** Surveys, generic events

### Outbound Manager and UI Integration

When a user creates an email in the campaign editor:

1. The front end queries the **Integrations Backend** to check if any installed integrations can receive emails
2. If yes, a **"Sync to CRM"** tab appears
3. User confirms sync; the **Outbound Manager** acts as a bridge between the front end and the external system
4. A message for the email activity is placed on the **All Sub Queue**

### Event Processing and Batching

Once an email is sent, Audience begins emitting events (sent, delivered, opened, clicked).

**Problem:** Sending events one-by-one to external systems creates excessive traffic.

**Solution:** Batch processing with Kafka.

**Flow:**

1. All events go to the **All Sub Queue** (Audience Subscription Queue)
2. **All Sub Worker** (ECS task) validates:
   - Is there an integration installed for this account?
   - Should this activity be synced?
3. Valid messages are placed on a **Kafka partition** (unordered globally, but partitioned by integration)
4. **Batch Production Worker** (ECS task) continuously polls the Kafka partition:
   - Groups messages by integration
   - Creates batches when either:
     - Batch size reaches 200K (staying safely below the 256K SQS limit)
     - Polling time exceeds 1 second
5. Ready batches are placed on an **SQS FIFO queue**
6. **Outbound Worker** (ECS task) consumes batches and uses the connector library to send to the external system

### Retry Logic and Dead Letter Queue

If a batch fails to send:

1. After a few retries, it goes to a **dead letter queue**
2. A **Retry Driver Lambda** runs every morning at 6:00 AM, re-queuing all messages from the dead letter queue
3. Messages are retried indefinitely (noted as a potential area for improvement)

> Current SQS limit is 14 days for message retention in the queue, but the retry system keeps redoing every day, so messages remain in the system indefinitely.

### Consent Bidirectionality

The team syncs consent changes back to external systems:

- If a profile opts out in APSIS One, an **opt-out message** is sent via the same All Sub → Outbound Worker path
- Messages bypass the batch production worker (consent is low-volume, one at a time)
- Consent sync is **bidirectional** (in ↔ out)

**Important caveat:** Attribute changes are **NOT synced back to the CRM**.

> This was a historical customer preference: CRM systems were considered the "data master," and attributes should flow from CRM to APSIS, not the reverse. Only consent is bidirectional.

### Redrive Consistency Caveat

During redrive operations from the dead letter queue:

- **Opt-in messages are NOT redriven** (only opt-out)
- **Batches lose eventual consistency** during redrive
- External systems are contractually required to handle idempotency by tracking batch IDs they've already processed
- This is documented in an agreed-upon process; if systems don't follow it, APSIS cannot guarantee consistency

### Outbound Field Mapping (Limited)

Users can configure **outbound field mappings** (attributes flowing from APSIS to the external system), but with limitations:

- These mappings are **not form-specific**; they apply globally to all form submissions for that integration
- When a form submit event arrives at the All Sub Worker, it doesn't use form-specific field values; instead, it pulls the already-mapped profile attributes from Audience and includes those in the outbound payload
- This is a historical limitation that CRM teams have complained about but was never addressed

---

## Integration Installation and CRM ID Limitation

### The CRM ID Problem

All integrations map to a single field in Audience profiles: **CRM ID**.

**Why this is problematic:**

- Only one CRM integration is allowed per Audience section
- If a customer wants to integrate both Microsoft Dynamics and Salesforce, they must use separate sections (which creates key space separation issues)
- After 2020, new integrations use system-specific ID attributes (e.g., dynamics_id, salesforce_id), but legacy integrations still depend on the shared CRM ID

**Historical reason:** A rushed release around 2020 ("super urgent Jonas release situation") that was never fully addressed.

**Migration path:** A data migration project could fix this, potentially combining with Eric and Prem's team as a joint effort in fall 2024, but it's significant work.

### Feature Flags and Capability Discovery

Installed integrations expose capabilities through feature flags and configuration:

1. **In APSIS backend code:** Specify what this CRM type supports (e.g., does it support outbound field mappings?)
2. **On the external system instance:** Check if the specific customer's instance supports a feature (different environments may have different versions)
3. **In the UI:** Show/hide tabs (field mappings, outbound mappings, profile lists, etc.) based on both checks

This is especially important for on-premise systems (e.g., eDeal, older Sitecore versions) where customers run different versions with different APSIS connector versions.

---

## Multiple Entities and Table Support

Some integrations can manage profiles based on **multiple entities** in the external system (not just contacts):

- Example: Contacts and Leads as separate tables
- Each gets its own dedicated ID attribute (lead_id, contact_id)
- This approach is standard for integrations built after 2020
- Legacy integrations fall back to the shared CRM ID, creating the limitation described above

---

## Operational Challenges and Monitoring

### Dead Letter Queue Monitoring and Operational Meetings

The team experienced high volumes of failed messages in the dead letter queue, especially when external systems had bugs, timeouts, or network issues. Reasons for failures:

- External system bugs
- Timeouts
- Network problems
- Customer misconfiguration (e.g., incorrect consent data)

**Mitigation:** Regular Monday operational meetings to review:
- Customer integration problems
- Dead letter queue contents
- Patterns in failures
- Actions to prevent recurrence

> Example: The team would follow up on issues reported 2, 3, and 4 weeks earlier, only to find the problems had still not been addressed by external partners.

### Infinite Redrive Without Limits

Currently, messages are redriven indefinitely without a configurable limit. This has been on the team's improvement list but hasn't been prioritized.

---

## Partner and Supplier Strategy

### Shift from Supplier-Controlled to Partner-Controlled

**Legacy approach (Supplier):**
- APSIS contracts with a vendor (e.g., CRM Consultana for Dynamics)
- Vendor develops and maintains a custom solution/plugin
- APSIS pays hourly; provides SLAs; owns release cycles
- Problems: Slow feature development, expensive, difficult to scale across multiple systems

**Current approach (Partner):**
- External partner implements the **generic connector API spec**
- Partner owns the integration; APSIS only manages the generic contract
- Partner bills customers directly (or charges APSIS a revenue share)
- Partner controls feature development and can build proprietary extensions beyond the generic spec

### Active Partners

1. **Sideshop** (Microsoft Dynamics 365)
   - Formerly: customers used the supplier-based Dynamics 365 CRM integration
   - Currently: migration plan to move all customers to Sideshop's partner-controlled implementation
   - Status: Commercial organization not fully executing on this migration strategy (as of the session date)
   - Benefit: Sideshop gets direct customer relationships and revenue

2. **Intermail**
   - Manages the **Intermail Loyalty** integration (custom implementation, not generic connector)
   - Also provides **Relation Plus** (generic connector-based CRM/loyalty platform)
   - Brings their own customers to APSIS; customers pay Intermail; Intermail and APSIS share revenue

### Generic Connector Interface

Partners implementing the generic connector must:
- Accept an **API spec** defining the contract (fields, events, operations)
- Provide a configuration to APSIS (spec file listing key space, attributes, events)
- Handle all system-specific logic on their side
- Implement idempotency for batch operations (track batch IDs to avoid duplicates)
- Support feature discovery (report what their instance can do)

### Standardized Contracts and Supplier Agreements

**Partner Agreement:**
- Defines what the partner can do
- Specifies APSIS termination rights
- Clarifies that the partner owns support and development

**Supplier Agreement (for any future supplier contracts):**
- Standardized contract template (developed with legal)
- Defines SLAs, response times, and issue severity levels
- Maps partner severity definitions to APSIS's internal P1/P2/P3 scale
- Requires detailed supplier delivery documentation:
  - Build process walkthrough
  - Minimum 75% test code coverage (enforced by CI/CD)
  - Detailed installation instructions
  - User roles required
  - Complete file paths and database tables created/altered
  - Overview of all entities created by the integration

**Supplier Delivery Process:**
- Partner must walk APSIS through the build before acceptance
- Ensures APSIS can answer customer questions (e.g., "What does this integration create on my system?")
- Prevents surprise side effects or hidden dependencies

### Benefits of This Approach

- **Scalability:** Add new systems without expanding APSIS engineering
- **Flexibility:** Partners can innovate beyond the generic spec (e.g., Intermail's loyalty features)
- **Reduced operational burden:** Partners handle their system's uptime; APSIS only guarantees the integration layer
- **Customer choice:** Partners compete on features and service

### Documentation and Handover

Key documents exist (though some are outdated or in personal storage):

- Generic Connector API Spec (latest version to be uploaded to S3)
- Generic Connector Implementation Guide (lengthy; updated as of 2024)
- Partnership Agreement template
- Supplier delivery process checklist

All documents should be versioned and hosted on S3 with domain names assigned so customers/partners can always access the latest versions.

---

## Advanced Feature: Unified Data (For Complex Multi-Table Scenarios)

### Problem Statement

Standard inbound sync imports a flat profile document with attributes. But many use cases require joining data across multiple tables in the external system.

**Example use case:** Paul, head of sales for an event company, wants to reach CEOs of companies whose employees attended his event last year. This requires:
- Event table (which employees attended?)
- Person table (who are they?)
- Role table (what is their role?)
- Company table (what company do they work for?)
- CEO query (is this person the CEO?)

These relationships cannot be flattened into a single profile attribute easily, especially when one person might have multiple roles at different companies with different emails.

### Solution: Unified Data via HTTP/2 Streaming

**Architecture:**

1. **Pre-made queries in the CRM:** Partners define queries that return structured data (e.g., "all CEOs at companies with attendees last year")
2. **Audience email tool:** When composing an email, user selects a unified data query
3. **Streaming connection (HTTP/2):**
   - Audience opens an HTTP/2 connection to Integrations
   - Integrations fetches paginated data from the CRM query (via connector library)
   - On-the-fly translation to Apache Columnar format
   - Streaming response back to Audience (no buffering entire result set in memory)
4. **Athena joins:** Audience's Athena query engine joins the streamed external data with its own profile exports
5. **Personalization:** Email tool can now reference fields from the joined data (e.g., "Hello [ceo_first_name]")

**Why HTTP/2 streaming is necessary:**
- External systems may return millions of records
- Buffering all results in memory is infeasible
- Streaming allows continuous pagination and translation without loading everything at once

### Current State and Production Readiness

**Status:** Built for the Max product suite (fall/winter 2024), completed on time.

**Testing:** Limited real-world validation.
- One customer used it successfully
- No 100+ customer validation (vs. other features with wider adoption)
- Not heavily exercised in production

**Recommendation:** Treat as production-capable but not battle-tested; monitor closely if expanded to more customers.

### Reusability

Though designed for Max with Maxo, the implementation is **not system-specific**. It can be reused for any CRM system as long as:
- The CRM partner defines appropriate queries
- The connector library fetches and translates data

### Alternative Approaches Considered

Flattening data into profile attributes (e.g., storing "company_title" on every CEO profile) was considered but rejected because:
- Audience keeps full historical attribute values for every profile
- Replicating company data per profile would explode the Athena bill
- Unified Data's streaming approach is more efficient

### Documentation

Comprehensive documentation exists covering:
- User experience flows
- Persona workflows
- Complete technical contract between Justin, Audience, Email Tool, and CRM
- Example API payloads for every step
- Easy to operate even if not heavily used

---

## Historical Context: Discontinued and Legacy Integrations

### CMS Integrations (Drupal, FP Server, Sitecore)

These worked differently than modern CRM integrations:
- No profile sync
- Users got a **segment dropdown** in the CMS content editor
- Content visibility was controlled by segment selection
- CMS would call **Segment Evaluation API** to check segment membership at render time
- Special key space juggling to ensure web key space was never whitelisted for CMS (security/data separation requirement)
- These integrations have been open-sourced; repositories available

### Discontinued CRM Integrations

- Shopify (eCommerce)
- LimeCRM
- Various others

These taught the team valuable lessons about the complexity of supporting multiple systems and drove the shift toward the generic connector model.

---

## Key Takeaways

1. **Justin is the modular middleware:** Generic infrastructure + system-specific connector libraries. New integrations should follow the generic connector pattern, not legacy supplier-dependent patterns.

2. **Inbound sync is comprehensive:** Field mappings, subscription mappings, full sync, delta sync (with eventual consistency buffering), and profile list tagging all work well at scale. Sync conditions allow users to filter which records sync.

3. **Outbound sync is event-driven and batched:** Activities and consent changes are streamed to Audience, batched via Kafka/SQS, and sent to external systems. Batching is essential to avoid DDOSing customers and to maintain reasonable SQS message sizes.

4. **Consent is bidirectional; attributes are not:** Only consent opt-in/opt-out flows back to CRM, not profile attribute changes. This was a historical customer preference that hasn't been revisited.

5. **The CRM ID limitation affects legacy integrations:** Modern integrations (post-2020) use system-specific IDs, but legacy ones depend on a shared CRM ID field, limiting each Audience section to one CRM integration. A data migration could fix this.

6. **Partner model is the future:** New CRM integrations should be implemented by partners using the generic connector spec. APSIS provides the contract; partners own implementation and customer relationships.

7. **Unified Data solves complex multi-table queries:** For use cases requiring joins across external system tables, unified data streaming (HTTP/2 to Athena) provides a scalable solution without exploding profile attribute storage.

8. **Operational health requires monitoring:** Regular Monday meetings to review dead letter queues, failed syncs, and emerging issues prevent silent failures.

9. **Documentation is critical for handover:** All supplier agreements, partner templates, and technical specs should be centralized and versioned (currently scattered in S3 and personal storage).

10. **Historical context matters:** Understanding why decisions were made (CRM ID field, supplier vs. partner, consent bidirectionality) helps avoid repeating mistakes.

---

## Unresolved Questions and Action Items

### From the Session

1. **PR Review and Merge:** Benjamin has a pull request (submitted 2022, feedback received spring 2024) that needs to be merged. Speaker 1 (Eric) will ask for feedback and help get it merged.

2. **Document Handover:** All supplier agreements, partner templates, generic connector specs, unified data documentation, and a list of active/historical partners need to be:
   - Collected into a centralized location (GitHub wiki or shared OneDrive folder)
   - Updated with any 2024+ changes (especially unified data and generic connector guides)
   - Made accessible to the team

3. **CRM ID Migration:** Moving legacy integrations from the shared CRM ID to system-specific IDs could be a joint project in fall 2024, but needs prioritization and scoping.

4. **Feature Expansion Opportunities (mentioned but not committed):**
   - Generic batching infrastructure (currently only used for outbound events) could be reused for webhook node in MA
   - Generic event subscription API (requested 5–6 years ago) could leverage the batching infrastructure

5. **Infinite Redrive Limit:** Add a configurable retry limit instead of infinite redrives.

6. **CMS Integration Documentation:** Open-source repositories for Drupal, FP Server, and Sitecore integrations should be provided to the team.

7. **Partner and Supplier List:** Michal requested a structured inventory of active and historical agreements with contact information. Benjamin provided a brief mental list; a spreadsheet should be formalized.

### Outstanding Clarifications

- Exact state and location of unified data implementation details (to be covered in future technical deep dive)
- Complete feature flag documentation for each integration type (mentioned but not fully detailed)
- Detailed retry logic configuration and limits (infinite today, needs scoping)

---

## Documents to be Provided

Benjamin committed to sending the following after the session:

1. All presentation slides (inbound and outbound architecture diagrams)
2. Generic Connector API Spec (latest version)
3. Generic Connector Implementation Guide (updated 2024)
4. Partnership Agreement template
5. Supplier delivery process checklist
6. List of active and historical partners/suppliers (spreadsheet)
7. Unified Data documentation and technical contract
8. CMS integration repositories (Drupal, FP Server, Sitecore)
9. S3 folder links for versioned documents

Format: GitHub wiki or shared OneDrive folder (to be determined; Speaker 1 will advise on location).