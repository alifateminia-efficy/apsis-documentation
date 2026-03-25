---
source_file: Benjamin - Intergrations Overview.txt
domain: Apsis One Integrations
topics:
  - Integration Architecture and Strategy
  - Justin Integration Middleware
  - Inbound Data Synchronization
  - Outbound Event Processing
  - Field and Subscription Mappings
  - Full Sync and Delta Sync Operations
  - Profile List Management
  - Unified Data Feature for CRM Queries
  - Partner and Supplier Relationships
  - Generic Connector Pattern
speakers:
  - Benjamin Nouhaag (Integration Strategy/Architecture Lead)
  - Lukasz Grabowski (Host/Facilitator)
  - Michal Rosikiewicz (Participant)
  - Premanand Thangamani (Participant)
  - Eric (referenced, technical lead)
  - Speaker 1 (unidentified)
key_components:
  - Justin (Integration Middleware)
  - Integration Manager (ECS task, formerly Lambda)
  - Mappings Manager
  - Delta Sync Worker (SQS FIFO, ECS task)
  - Full Sync Producer/Consumer (on-demand infrastructure)
  - Outbound Manager
  - All Sub Queue (Audience Subscription Queue)
  - All Sub Worker (Audience Subscription Worker)
  - Batch Production Worker (Kafka-based)
  - Outbound Worker
  - Full Sync Manager
  - Profile List Sync Lambda
  - Retry Driver Lambda
  - Athena (Profile Store in Audience)
  - React (Real-time profile store)
session_type: knowledge-transfer
subdomains:
  - Architecture
  - Inbound Flow
  - Outbound Flow
  - Generic Connector
  - Different Types of Connectors
  - Microsoft Dynamics
  - Lead creation
  - Duplicate profiles in Apsis
---

## Session Overview

Benjamin Nouhaag delivered a comprehensive conceptual overview of the Apsis One Integrations domain to incoming team members (Lukasz Grabowski and Michal Rosikiewicz). The session covered the original mission and architecture of the integrations platform (built on Profile Cloud/Audience), the **Justin** middleware system that standardizes communication with external systems, detailed flows for both inbound (profile synchronization) and outbound (event processing) data movement, the transition from supplier-owned connectors to a partner-based generic connector model, and an emerging unified data capability for complex multi-table CRM queries. The presentation emphasized conceptual understanding over technical depth, with architectural diagrams illustrating the processing queues, workers, and service interactions that ensure uptime and consistency across integrations.

---

## Historical Context and Original Mission

### Why Audience/Profile Cloud Exists

The integrations team was formed to solve a fundamental data synchronization problem that existed across enterprise IT in the early 2010s. At that time, companies ran many disparate systems (CRM, ecommerce, marketing platforms, etc.) without a unified view of customer data.

> The idea originally is that if you go to any company, at least in 2013, 2014 when this stuff started being full, you would see a lot of different systems and the IT departments were integrating all of these systems with each other in order to be able to provide some kind of holistic customer view and things were always out of sync, things were always lagging. And no data was over there when you needed it.

**Apsis One (formerly Profile Cloud, then Audience)** was positioned as a hub in the middle of these systems to maintain a unified, up-to-date customer view.

### The Integration Team's Mission

The integrations team's core responsibility is to:

1. **Create bidirectional arrows** (data flows) between Apsis One and as many external systems as possible
2. **Provide the same SLA (Service Level Agreement)** that Apsis One provides for the rest of its platform—ensuring uptime through 24/7 pager duty and contractual guarantees
3. **Mitigate external risk** by standardizing use cases for system types (CRM, ecommerce, etc.) and controlling the scope of logic residing outside Apsis One

> Our mission as an integrations team in Apsis One was to ensure that these arrows exist for as many systems as possible and provide in these arrows same SOA as APSIS provides for the rest of APSIS one.

### Early Strategy: Standardization and Supplier Partnerships

The original architectural approach was to:

- **Standardize use cases** per system type (e.g., an "abandoned cart" infrastructure that ran hourly across all ecommerce systems)
- **Minimize external logic** through tight supplier relationships (e.g., with **Geran Kosnokana** for Microsoft Dynamics, who built a plugin with webhooks and API guidance)
- **Use connector libraries** to handle system-specific communication while keeping the core infrastructure generic

However, this strategy proved unwieldy in practice. Edge cases in external systems (such as Lime CRM's distinction between opting out of contact and removing contact-marketing relationships) required unexpected development effort and led to last-minute fixes just before launch.

---

## Justin: The Integration Middleware

### Core Concept

**Justin** is the integration middleware layer—named after Justin Timberlake because "it's job is to keep everything in sync." Justin provides:

1. **Generic infrastructure** for managing integrations
2. **Connector libraries** specific to each external system, enabling easy communication and field translation

The core modularity principle: the connector library should only need to handle a simple HTTP request that fetches fields from the external system, translates them to Justin's expected format, and returns them to the front end.

### Architecture Overview

Justin sits between Apsis One and external CRM/ecommerce systems, handling:
- **Field mappings** (external system fields ↔ Apsis profile attributes)
- **Subscription/consent mappings** (external consent data ↔ Apsis topics)
- **Real-time webhooks** (delta sync) for incremental updates
- **Full syncs** for bulk data imports
- **Outbound event delivery** (emails, SMS, MA flows, activities) with batching and retry logic

---

## Inbound Data Flow (CRM → Apsis One)

### Integration Installation and Setup

When a user installs an integration (e.g., FC Enterprise 12.1), they:

1. Provide connection credentials (URL, API key)
2. Set up **field mappings** on the field mapping page, mapping external system fields to Apsis profile attributes
3. Configure **subscription mappings** to define how consent/topic data flows from the external system
4. Run a **full sync** to import existing data
5. Set up **profile lists** (queries) if needed to tag profiles based on external list membership
6. Define **sync conditions** to filter which records should be synchronized

### Field Mapping Page and Schema Fetching

When rendering the field mapping dropdown, the system:

1. **Integration Manager** (an ever-running ECS task, formerly a Lambda) is called
2. Integration Manager uses the appropriate **connector library** (Dynamics, FC Enterprise, or generic connector) to fetch the schema from the external system
3. The schema list is returned to the front end, allowing the user to select which fields to map

> So if the URL that you call it with says dynamics, then it's going to try to use the Microsoft Dynamics connection library. If it says FC enterprise, it's going to try to use the FC enterprise connection library and if it's something running on the generic connector I mentioned, there's going to be an internal mapping.

### Mappings Manager and Justin's Laws

When the user saves field mappings:

1. **Mappings Manager** stores the mappings in its own database (separate user with access to mappings table only)
2. **Justin's Laws** (detailed elsewhere by Eric) ensure mappings don't create loops or other inconsistencies

---

## Real-Time Synchronization: Delta Sync

### Webhook Subscription

During installation and whenever mappings change, the system:

1. Subscribes to webhooks on the external system for events like contact creation or update
2. Some systems send data to an **official Delta Ingestion Manager endpoint** provided by the integrations team
3. Other systems send data to a **custom webhook endpoint** managed by integrations without customer involvement

### Delta Sync Processing

Webhook payloads flow through:

1. **Delta Ingestion Manager** (receives incoming webhooks)
2. **SQS FIFO Queue** (message group = CRM ID) ensuring order per customer
3. **Delta Sync Worker** (ever-running ECS task) consumes messages:
   - Asks **Mappings Manager** which mappings exist (with caching that invalidates on updates)
   - Applies field mappings to translate external data to Apsis format
   - Checks **sync conditions** (if defined) to determine whether to sync the record
   - Sends the processed profile data to **Audience**

### Sync Conditions

Users can define conditions that filter which records are synced. For example:
- Only sync contacts where `active == true`
- If an inbound message has `active == false`, also **delete the profile** in Apsis (if the feature is enabled)

---

## Full Synchronization

### Purpose and Architecture

A **full sync** imports all contacts from the external system according to the current field mappings and sync conditions. This is typically run after initial setup to populate Apsis with historical data.

When a user clicks "Start New Sync":

1. **Full Sync Manager** (ECS task) receives the sync request
2. **On-demand infrastructure** is spun up for this specific job (including producer and consumer tasks)
3. **Full Sync Producer** fetches paginated data from the external system using the connector library
4. Data is placed on a **full sync queue** in Justin's expected format
5. **Full Sync Consumer** processes each message:
   - Requests mappings from **Mappings Manager**
   - Applies field mappings and sync conditions
   - Applies subscription/consent mappings
   - Sends profiles to **Audience**

### Eventual Consistency and Buffering

A critical design challenge: what happens when new real-time updates (delta sync) arrive while a full sync is running?

**Problem:** If delta sync and full sync process messages out of order, eventual consistency is broken (e.g., a contact updated mid-full-sync might be overwritten by an older snapshot).

**Solution:** During a full sync:

1. Real-time delta sync messages are **buffered** in a separate queue for that integration
2. The system **sets the message visibility timeout** to delay processing until the full sync completes
3. After the full sync finishes, buffered messages are processed in the order they arrived

> We have something consuming from that queue and if a full sync is ongoing for the account that a message is is from or for the integration that a message is for here, then we will say then we will delay it in this queue by setting the visibility timeout on the message.

### Finalization States

The full sync UI shows several states:

- **Finalizing:** Profiles have been downloaded and are being processed; ingestion delay (lag in Audience) is being monitored
- **Successful:** All profiles have been downloaded, processed, and Audience ingestion has caught up

The Full Sync Manager waits until:
1. All producer threads report no more data in paginated requests
2. All consumer threads report the queue is empty
3. Audience's ingestion delay has caught up to the point when the last full sync message was written

---

## Profile Lists and Queries

### Terminology Challenge

External systems use different terminology:
- **Microsoft Dynamics:** "Market" = list of contacts (dynamic via query, or static via manual maintenance)
- **FC Enterprise:** "Query" = dynamic list, "Profile" = static list

Apsis One abstracts both into:
- **Dynamic Profile List:** Automatically maintained by a query (similar to Apsis segments)
- **Static Profile List:** Manually maintained

> In FSC enterprise, a dynamic list is called a query and a static list is called a profile. So that's why it says profiles here. It has nothing to do with the profiles and APSIS and it's also a very unfortunate thing that we've never had time to look at from a conceptual point of view or terminological point of view.

### How Profile List Sync Works

When a user imports a profile list (e.g., "Newsletter"):

1. The system **does not create profiles** in Apsis
2. Instead, it **adds a tag** to all profiles on that list
3. If the list is re-synced and a profile is no longer on it, the tag is **removed**

The system tracks which operation assigned each tag so it can safely remove it later.

### Recurring Profile List Sync

Users can set profile lists to sync on a schedule (currently 6:00 AM every morning):

1. A **CloudWatch event** triggers the **Profile List Sync Lambda**
2. The Lambda places jobs on the **profile list sync queue**
3. A **consumer** spins up one worker per integration (sequentially, to avoid DDOSing the customer)
4. Each worker **fetches every contact on the list** (held in memory) and sends a **tag request** to Audience

> This one actually fetches every contact on the list and does not work with a queue or anything, just holds it in memory and sends a tag request all the way to audience. So this is where we've had some problems in the past because before we addressed various edge cases, these failed a lot and there was no queue, so so there was a little bit of a problem there, but another quite stable.

**Historical Issue:** Before edge cases were handled, profile list syncs would fail frequently with no queue to retry, causing data inconsistencies. This has since been addressed.

---

## Outbound Data Flow (Apsis One → External Systems)

### Overview

Whenever a user creates an activity in Apsis One (email, SMS, form submission, MA flow, or event registration), the system can synchronize that activity and its subsequent events (opens, clicks, deliveries, responses) to the external CRM system.

> Whenever I create any kind of activity, so e-mail, SMS form, even an MA flow, we can sync this to the external system. I think the only thing we can't even sync events today actually, just not surveys.

### Activity Creation and CRM Sync UI

When creating an email:

1. The front end **asks the integrations backend:** "Are there any integrations that can sync emails?"
2. If yes, a **CRM Sync tab** appears showing available integrations
3. The user can **check "Sync to [CRM]"** to enable outbound sync
4. When the email is sent, the **Outbound Manager** bridges between the front end and the external system, instructing it to create a corresponding record

---

## Outbound Event Processing: Batching and Delivery

### Event-Driven Pipeline

As emails are opened, clicked, delivered, etc., Apsis One generates event records. Rather than sending each event one-by-one (which could generate massive traffic), a sophisticated batching system groups events:

1. **All Sub Queue (Audience Subscription Queue):** SQS FIFO queue receiving all events from Audience that need outbound sync
2. **All Sub Worker** (Audience Subscription Worker, ECS task):
   - Verifies that an integration is installed for the account
   - Verifies that this activity should be synced to the external system
   - Places the event on a **Kafka partition**
3. **Batch Production Worker** (ECS task):
   - Continuously polls the Kafka partition
   - Groups messages by integration (installation)
   - Holds batches until either:
     - The batch reaches **200 KB** in size, or
     - **1 second** has passed since polling started
   - Places ready batches on **SQS FIFO queue**

> We have 4 messages intended for installation A. That's if A1A2A3A43 messages intended for installation BB1B2 and B3 and two messages intended for installation CC1C2. And these are all kind of juggled up mixed with each other and the batch production worker's job, which is again an ever running ECS has to keep pulling this Kafka partition pulls and pulls and pulls and pulls.

**Size Limit Rationale:** The 200 KB batch size is chosen to stay well below the **256 KB SQS message size limit**, leaving headroom.

### Delivery and Retry

1. **Outbound Worker** (ECS task):
   - Takes ready batches from SQS
   - Uses the connector library to deliver batches to the external system
   - If delivery fails after retries, sends to **Dead Letter Queue**

2. **Retry Driver Lambda**:
   - Runs daily at 6:00 AM
   - Redrives messages from the dead letter queue back to the SQS queue
   - **Current behavior:** Redrives infinitely until successful (potential area for improvement)

> And then we have a retry driver Lambda which runs at 6:00 every morning and then just puts everything back to the queue.

### Scale Considerations

For a million events, this system could generate over 1,000 batches. While this has not caused problems historically, it's on the improvement roadmap to potentially use a different queue technology.

---

## Consent Synchronization (Bidirectional)

### Inbound Consent

During field mapping setup, users configure **subscription mappings** that define:
- Which external system field or resource represents consent
- How it maps to Apsis topics

Consent data flows inbound through the normal delta sync and full sync pipelines.

### Outbound Consent (Opt-Out)

When a profile opts out of a topic in Apsis One:

1. A **consent change event** is generated
2. It flows through the **All Sub Queue** and **All Sub Worker** (bypassing batch production, since consent messages are infrequent)
3. The **Outbound Worker** sends the opt-out to the external system
4. If it fails, it enters the **dead letter queue** and **retry carousel**

**Critical Difference from Attributes:** While consent is **bidirectional** (inbound and outbound), attribute changes flow **only inbound** (external system → Apsis). Attribute changes from Apsis back to the external system are **not supported** by design.

> As as then just said, like we are syncing the consent changes back to the CRM, but we do not sync attribute changes back to the CRM like the consent is bidirectional but the attributes only come to Apsis, but not from Apsis. And that's that's because historically customers have not wanted that. They feel like the CRM should be the data master.

**Rationale:** Customers prefer their CRM to be the authoritative data master, with Apsis primarily enriching and acting upon that data.

---

## Batch Idempotency and Eventual Consistency

### Challenge: Redriven Batches Lose Strict Ordering

When batches are redriven from the dead letter queue:
- They may be reprocessed out of order
- Strict **eventual consistency** is no longer guaranteed

### Solution: Partner Responsibility

The integrations team defines a contract with external systems:

1. **Each batch is assigned a unique ID**
2. The external system **must track which batch IDs it has already processed**
3. If a redriven batch arrives, the external system should **idempotently process it** (i.e., not apply it twice)

> And an example of how that works for at least FSC enterprise and it's sort of expected to work the same way for anything in the generic connector is that we we generate an ID for each batch and then they should keep track of whether they have already processed a batch.

The agreement is documented, but external systems are free to implement it however they choose. If they don't follow the contract and data inconsistencies result, that's beyond Apsis's control.

### Consent Redrive Special Case

For **opt-out messages**, the system plays it safe:

- Only **opt-out (opt-in == false)** messages are redriven
- **Opt-in messages are NOT redriven** once a redrive is in progress
- Reason: Redriving opt-in messages could re-opt users without their consent

---

## Dead Letter Queue Monitoring

### Observability and Operational Meetings

The integrations team has experienced numerous dead letter queue issues across different external systems, each with different failure modes:

- External system bugs or timeouts
- Network problems
- Incorrectly configured customer consent data that cannot be processed
- Vendor-specific quirks and edge cases

> There are a lot of different reasons why it could end up. The customer can also have incorrectly imported consent in apps that we cannot process.

### Operational Best Practice

Benjamin recommends establishing a **Monday operational meeting** to:
- Review items stuck in the dead letter queue
- Identify patterns (e.g., always failing for one customer or system)
- Proactively discuss workarounds and fixes
- Track long-standing issues with external vendors

> We've had a at a time when we had a lot of operational problems, we started having an operational meeting once per Monday where we looked over various customer integration problems and also look through what's in the dead level queue, what are we always redriving so that we get some kind of excited for this and talk about it and see how to avoid it.

---

## Marketing Automation: Integration Nodes and CRM Activities

### MA Flow Integration Nodes

Users can add **integration nodes** (CRM-specific nodes) directly in Marketing Automation flows. Example flow:

1. Listen for event: "Invited to event"
2. Wait 2 days
3. Check if profile has "confirmed" the event
4. If not confirmed: Create a task in FC Enterprise CRM to call the contact and follow up

### Implementation Details

When a profile reaches an integration node in a flow:

1. A **message is placed on the All Sub Queue** with details of the node interaction
2. The message flows through the **batch production dance** (grouping by integration)
3. The **Outbound Worker** sends the task creation to the external system

### Configuration and Schema

Integration nodes display **configuration fields specific to the external system schema**. For example, when creating a task in FC Enterprise:

- **Config values** are based on the CRM schema (e.g., urgency level 1-10, assignment rules, phone call type)
- If schema cannot be fetched, an **integration error** is displayed

> These are all config. Um. Config values based on the external systems schema for for whatever we're creating. In this case we're creating a task and they support setting urgency appliance 10.

### Scale Considerations

MA flows can trigger at massive scale (e.g., 500,000 people entering a flow at the same scheduled time). The batching system ensures that even with one-by-one event triggers, the outbound delivery is aggregated into efficient batches rather than overloading the external system.

---

## Form Submissions and Outbound Mapping

### Form Field Mapping Challenge

When a user creates a form and maps form fields to profile attributes, they might want form submissions to also sync to the external CRM—and ideally, different forms could map to different CRM fields.

**Current Limitation (as of 2022):**
- Form submissions cannot directly map to CRM fields on a per-form basis
- Instead, there is a separate **outbound mappings** tab (the reverse of field mappings)
- Outbound mappings are **global to the integration**, not per-form

### How It Works Today

When a form submission event arrives:

1. The **All Sub Worker** processes the message
2. Rather than mapping form fields directly, it retrieves the **mapped attributes from the profile** using outbound mappings
3. These attributes are included in the outbound payload to the CRM
4. The form fields themselves are not used

> Instead it takes those specifically mapped attributes from the profile and puts that in in an object in the outgoing payload.

**Customer Complaint:** CRM teams wanted per-form control, but the current design forces a one-to-one mapping globally.

---

## Multi-Integration and CRM ID Field Limitation

### The "Very Unfortunate" Design Constraint

A critical architectural limitation exists: **only one CRM integration per section.**

**Root Cause:** Both field mappings and profile identification depend on a hardcoded **CRM ID field**. When multiple CRM integrations exist on the same account:
- Both would try to map to the same `crm_id` field
- Profiles from Dynamics and FC Enterprise would be mixed up (different systems, different ID spaces)

> Field mappings, the map to CRM ID. This is very, very, very unfortunate. This is an old Jonas super urgent release something now situation.

### How IDs Are Actually Managed

The system does bootstrap a **unique key space per integration** (e.g., `dynamics_id`, `fc_enterprise_id`), but the UI always maps profiles to the generic `crm_id` field. This restriction prevents customers from using multiple CRM systems on the same section.

**Solution Workaround:** Customers needing multiple CRM integrations must use **separate sections**.

**Historical Context:** This limitation dates to an early urgent release decision (referenced as "Jonas super urgent") that was never revisited due to prioritization constraints.

### Recent Improvements (Post-2020)

Integrations built after 2020 properly **bootstrap a unique ID attribute per entity type** (e.g., `lead_id` vs. `contact_id`), allowing multiple entity mappings within a single integration without the CRM ID field collision.

> Anything done after 2020 has kept that in mind. For 2019, but we are dependent on the CRM ID for a lot of historical integrations and we would need to do a big migration in order to in order to deal with that. It could be a good idea to do during the fall if if if like to do as a joint migration project together with yourselves.

---

## Multiple Entities and Entity-Specific IDs

### Entity Types Beyond Contacts

Some CRM systems allow mapping multiple entity types (not just contacts):
- **Contacts** (primary, gets `contact_id`)
- **Leads** (separate, gets `lead_id`)
- Other entity types as supported

### UI and Schema Representation

When multiple entities are configured:

1. Each entity type appears as its **own table** in the field mapping page
2. Each has its **own schema** (different fields available)
3. Each gets a **dedicated ID attribute** (not the shared `crm_id`)

This design allows scalability without ID collisions, but requires migration work for older implementations.

---

## Integration Installation Options and Feature Flags

### Installer Options

Each integration has **installer options** that configure:
- Which features are enabled for this specific integration type
- What capabilities are available to customers

These options are defined **per integration type**, not per customer.

### Feature Availability Checking

The system performs a **two-level check** when rendering UI tabs:

**Level 1: Integration Specification**
- Does this integration type (e.g., Dynamics) support profile lists?
- Is outbound mapping enabled for this integration?
- Feature flags can be used to control availability

**Level 2: External System Instance**
- When connecting, does this specific CRM instance support the feature?
- For on-premise systems (e.g., Efficy/Edeal), versions may differ
- The schema is queried in real-time to detect what's actually supported

Example: Edeal customers running on-premise may have different versions:
- Customer A's Edeal instance supports outbound mapping
- Customer B's Edeal instance (older version) does not
- The system detects this per-instance and hides or shows tabs accordingly

> Because some specific integration might not, might not have any profile list for example, then we'll hide this tab and the same thing goes for upbound mappings. You might not support us sending you that and then we hide that tab if that is not present on the list.

---

## Evolution: From Supplier-Owned to Partner-Based Model

### Original Model: Supplier Partnerships

In the early years, integrations worked with **suppliers** (paid contractors):

**Example: Microsoft Dynamics with Geran Kosnokana**
- Geran built a plugin with webhooks and API guidance
- Apsis paid them by the hour
- Maintained SLA agreements with Geran
- Any feature addition required:
  1. Apsis ideation
  2. Discussion with Geran
  3. Geran implementation (paid)
  4. Apsis review

**Problems:**
- Slow feature velocity—feature requests had to go through Geran
- Difficult to scale—different systems (Dynamics, Salesforce, Lime CRM, Shopify, etc.) all required custom relationships
- Edge cases discovered late: e.g., Lime CRM's distinction between opt-out and contact removal wasn't covered by Geran's implementation
- Surprise work 1-2 days before launch

---

### Modern Model: Generic Connector and Partners

**New Approach:** Rather than owning implementations, Apsis provides a **generic connector specification** that external systems implement themselves.

**Process:**
1. Apsis defines a **contract** (API spec, required endpoints, field formats)
2. The external system (or a partner) **implements the contract** on their side
3. Apsis only needs a **config file** to activate it (no code changes)

> So instead of working in a setup where we control everything and we own everything, we try to switch strategies and work instead of suppliers but with partners.

### Supplier vs. Partner Distinction

**Suppliers** (traditional model):
- Apsis pays them for work
- SLAs and hourly rates
- Apsis controls the roadmap
- Example: Geran Kosnokana (no longer active)

**Partners** (modern model):
- External party builds to the generic connector spec
- Partner bills customers directly (Apsis doesn't pay)
- Partner owns support and maintenance
- Apsis gets revenue sharing or just ecosystem growth
- Example: **Sideshop** (Dynamics 365 CRM partner), **Intermail** (Loyalty and Relation Plus partners)

### Examples of Partners

**Sideshop:**
- Implements generic connector for Dynamics 365
- Customers pay Sideshop directly (~200 EUR/month)
- Sideshop handles all feature development
- Both sides benefit: Sideshop gets revenue, Apsis gets integration without owning it

> Side shop or a partner, meaning we don't pay them, but they get they get revenue. So the plan right now, which is not unfortunately being followed by the commercial organization, is that every single customer running this should be migrated to this. And Sideshop should be getting should be billing for the integration revenue.

**Intermail:**
- Provides Loyalty integration (custom, not generic connector)
- Provides Relation Plus integration (generic connector)
- Drives customers to Apsis, charges for integration
- Apsis gets integration growth without implementation cost

**Legacy:** Microsoft Dynamics (Geran Kosnokana)
- Still active but no longer preferred
- SLA agreements exist but are no longer paid for (customers still think they are)
- Should be migrated to Sideshop partnership

---

## Partner and Supplier Agreements

### Standardized Contracts and Processes

Benjamin and the legal team developed **standardized agreements** and **delivery processes** to ensure quality and consistency:

#### **Supplier Agreement**
- Defines terms with paid contractors
- Specifies SLAs, response times, issue severity definitions
- Includes **Supplier Delivery Process** checklist

#### **Supplier Delivery Process**
All deliverables must include:
- Walk-through of build process with Apsis
- **75% test code coverage** (enforced in build pipeline)
- **Detailed installation instructions**
- List of **user roles required**
- Overview of **files created/modified** with exact file paths
- Overview of **database tables and entities** created/altered
- Definition of done checklist

> We want them to walk us through the build process. Like before we say that this is done, they walk through the build process, show us that they have 75% test code coverage and that it's like if you don't have the test coverage, it prevents the bill from passing.

**Rationale:** These deliverables allow Apsis to answer customer questions ("Will this break my system?") and maintain operational clarity.

#### **Partner Agreement**
- Defines terms with parties implementing the generic connector
- Clarifies that the partner **owns support** and development
- Specifies that the partner **owns intellectual property** and billing
- Allows for rapid onboarding without legal complexity

### Documentation and Handoff

**Deliverables for Partners:**
- **Generic Connector API Specification** (versioned, stored in S3)
- **Generic Connector Implementation Guide** (lengthy technical document, last updated 2022, refresh planned for 2024)
- Links to all specifications in shared S3 folder with DNS alias

> All these files are hosted on a folder an S3 folder with some domain name assigned to it and that's where actual all these links go I believe. So we always have a latest generator connector API spec that we manually add to this folder.

---

## Historical Integrations: CMS and VMS Systems

### Legacy CMS Integrations

Apsis previously built integrations with **content management systems:**
- **Drupal**
- **FP Server** (?) [uncertain from transcript]
- **Sitecore**

These worked differently from modern CRM integrations:

**Instead of syncing profiles and events:**
1. User editing content gets a **segment dropdown**
2. User specifies: "Show this content only to segment X"
3. Content system sends a request to **Segment Evaluation API**
4. Apsis evaluates whether a viewer belongs to the segment
5. Content system handles **key space juggling** so nothing is whitelisted for the "web" key space

> Instead they essentially let the user get a drop down of segments when editing content and then say this content should only be shown to the segment and then they send a request to the segment evaluation API and then there's a little bit of key space juggling on the CMS side.

**Status:** These integrations have been **rendered open source**. Documentation and repositories are available for reference.

**Relevance:** Provides pattern for personalization use cases outside traditional CRM/ecommerce.

---

## Unified Data: Complex Multi-Table Queries from CRM

### The Problem It Solves

Traditional profile-based integration assumes:
- One record per person
- Flat attributes (e-mail, name, etc.)
- History in Apsis (for behavior and events)

However, customers often want to **access relational data from the CRM** that isn't stored as profile attributes in Apsis.

**Example:** Paul, CEO of an event company, wants to:
- Find all **CEOs of companies** whose **employees attended an event last year**
- Personalize an email: "Hello Alfonso, it seems like Imashek had a blast at Event in Poland last year. How do you feel about sending more reps from FSC this year?"

**Why This Is Hard:**
- The CRM stores structured data across multiple tables: Events → Attendees → People → Roles → Companies
- To find CEOs, you must traverse: Event → Event Attendees → Person → Person Roles (filter for CEO) → Company
- Apsis profile data is flat; it doesn't replicate all these relationships

---

### Solution Architecture

**Unified Data** leverages **Athena** (Apsis's columnar profile store) and allows customers to:

1. **Define pre-made queries in the CRM** that expose structured data with selected columns
2. **Stream that data to Apsis** in real-time for a specific send operation
3. **Join CRM data with Apsis profiles** to enable advanced personalization

#### **Technical Flow**

When creating a personalized email with CRM data:

1. **Audience (email tool)** triggers an export internally
2. Audience specifies: "Join with results from query 'CEOs from attending companies'"
3. Audience opens an **HTTP/2 connection** to integrations
4. **Integrations** uses a connector library to **fetch paginated data** from the CRM query
5. Data is **converted to Apache Columnar format** on-the-fly
6. Data is **streamed back** to Audience through the HTTP/2 connection
7. Audience **joins the streamed data** with profile attributes
8. Email is personalized with both profile data and CRM relation data

> We have audience being a meta profile store and Athena essentially right where profile store is react, where we have all these attributes with all these values and then we have meta telling us what each attribute, how each attribute actually looks.

**Why HTTP/2:** Allows streaming of large result sets (potentially millions of rows) without loading everything into memory on either side.

**Data Format:** Apache Columnar (efficient for analytics and joining)

#### **Key Design Principle**

This approach avoids **flattening all CRM data into Apsis profiles**, which would:
- Explode Apsis storage costs (Audience charges based on data volume)
- Create stale snapshots (role/company relationships change, but synced data wouldn't)
- Require continuous re-sync of complex relationships

Instead, Unified Data **pulls live data on-demand** when a send operation needs it.

---

### Current Status and Limitations

**Implementation Status:**
- Built for **FSC Maxo** (Product Suite project, Fall/Winter 2024)
- Delivered on schedule
- Successfully tested with emails and personalization

**Production Maturity:**
- One paying customer has used it
- Not extensively battle-tested with 100s of customers
- Successful sends have occurred, but not widespread usage

> Yeah. So we, I mean, we had the product suite project in the fall or winter of 2024 and we finished everything on time. But of course no customer has used it because Maxo didn't bring in customers. We have one that, yeah, we have one Berto customer, but. So. Yeah, I I would not say that it is really tried in tried in actual usage.

**Future Applicability:**
- The pattern is **system-agnostic** (built in integrations connector library, nothing Maxo-specific)
- Can be reused for any CRM where customers want multi-table queries
- Expect requests as customers want richer personalization

**Documentation:**
- Comprehensive technical documentation exists
- Details user experience flows
- Technical contract specifying API calls and payloads between Justin, Email Tool, Audience, and CRM

---

## Data Flattening Attempts and Why They Failed

### Historical Challenge: Replicating Relational Data

The team previously attempted to **flatten relational CRM data** (e.g., a person's title at a company) into Apsis profile attributes.

**Why It Didn't Work:**
1. **Audience stores full history** of every attribute on every profile
2. Replicating title/company for every person would **multiply Apsis storage** and costs
3. **Stale data:** Syncing title periodically means data gets out of sync when changes happen in CRM
4. **Expensive:** Scaling to thousands of customers with complex relationships becomes prohibitively costly

> That's what I wanted to cover with unified data. There's also very good documentation of exactly how that works in case you need to get back into it where like everything is described like how it works. User experience steps, all the various ways a user can pass through the current implementation.

**Why Unified Data Is Better:**
- Pulls data **on-demand** only when needed
- **Live data**, not snapshots
- Doesn't bloat Apsis storage
- Scales with query complexity, not total data volume

---

## Integration Installation Workflow Summary

### Inbound Flow (CRM → Apsis One)

**Components Involved:**
- Integration Manager (schema fetching)
- Mappings Manager (storing mappings)
- Delta Sync: Webhook → Delta Ingestion Manager → SQS → Delta Sync Worker → Audience
- Full Sync: Full Sync Manager → Full Sync Producer → Queue → Full Sync Consumer → Audience
- Buffering during full sync: Delta messages held until full sync completes
- Profile Lists: List Import → Worker → Tag Audience Profiles
- Sync Conditions: Conditional filtering during message processing

**Flows:**
1. **Field Mappings:** External field → Apsis attribute
2. **Subscription Mappings:** External consent → Apsis topic
3. **Real-time Sync:** Webhook → Delta Sync Worker → Audience
4. **Bulk Sync:** Full Sync → Import all data at once
5. **Profile Lists:** External list → Tag profiles
6. **Conditions:** Filter records before syncing

---

### Outbound Flow (Apsis One → CRM)

**Components Involved:**
- Outbound Manager (bridges UI to CRM)
- All Sub Queue (Audience Subscription Queue)
- All Sub Worker (event subscription worker)
- Batch Production Worker (groups by integration)
- Kafka partition (event juggling)
- Outbound Worker (delivery)
- Dead Letter Queue + Retry Driver (failure handling)

**Flows:**
1. **Activity Sync:** Email/SMS/Form → Outbound Manager → CRM
2. **Event Processing:** Activity events (opens, clicks, etc.) → All Sub → Batching → Outbound Worker → CRM
3. **MA Flow Nodes:** Profile reaches integration node → Event to All Sub → Batching → Outbound Worker → CRM
4. **Consent Sync:** Profile opts out → Consent event → All Sub → Outbound Worker → CRM (bypasses batching)
5. **Failure Recovery:** Dead Letter Queue → Retry Driver (daily at 6 AM) → Redrives

---

## Current Architecture Challenges and Open Questions

### Known Limitations

**CRM ID Field Collision:**
- Only one CRM integration per section (by design, due to hardcoded `crm_id` field)
- Requires separate sections for multi-CRM setups
- Should be migrated post-2020 integrations to use entity-specific IDs
- **Action Item:** Large migration project possible during fall

**Form Field Mapping:**
- Form submissions don't support per-form CRM field mapping
- Uses global outbound mappings instead
- CRM teams have complained about lack of per-form control
- **Workaround:** Use global outbound mappings (not ideal)

**Infinite Redrive:**
- Consent and event batch redrives continue indefinitely
- No limit on retry attempts
- Could theoretically retry forever
- **Concern:** Resource usage, customer confusion about stale failures

**Profile List Memory Usage:**
- Profile list sync fetches all contacts into memory before tagging
- No queue-based approach (unlike full sync)
- Caused failures in the past when edge cases weren't handled
- Now stable but less robust than full sync design

---

## Key Takeaways

1. **Apsis One Integrations operates a hub-and-spoke architecture** where Justin middleware standardizes communication with external CRM and ecommerce systems while maintaining contractual uptime guarantees.

2. **Inbound data flow** (CRM → Apsis) uses both real-time webhooks (delta sync) and scheduled bulk imports (full sync), with sophisticated buffering to maintain eventual consistency during simultaneous syncs.

3. **Outbound event flow** (Apsis → CRM) uses Kafka-based batching to group events by integration, avoiding one-by-one delivery that would overwhelm external systems, with daily retry cycles for failed batches.

4. **Consent is bidirectional** (in and out), but **attributes are inbound-only** by customer design preference—CRM remains the data master.

5. **The shift from supplier-owned integrations to partner-based generic connectors** (e.g., Sideshop for Dynamics) reduces Apsis's operational load and accelerates feature delivery by letting partners own implementation and support.

6. **The CRM ID field design limitation** (only one CRM per section) is a known technical debt dating to early urgent releases; post-2020 integrations properly use entity-specific IDs, but migration is needed for legacy systems.

7. **Unified Data** enables complex multi-table CRM queries to be streamed on-demand during email personalization, avoiding the cost and staleness of flattening relational data into profiles.

8. **Operational excellence requires proactive monitoring**—establish Monday operational meetings to review dead letter queue items, identify patterns across vendors, and coordinate fixes.

9. **Partner agreements and supplier contracts** (including delivery process checklists and SLA templates) are standardized documents essential for scaling partnerships; they should be in the GitHub repository and actively maintained.

10. **Legacy CMS integrations** (Drupal, Sitecore, FP Server) used a different pattern (segment evaluation API) and are now open source; they provide reference implementations for non-CRM personalization use cases.

---

## Unresolved Questions and Action Items

### Action Items Assigned

1. **Benjamin to merge PR for partner/supplier documentation**
   - Pull request submitted in 2022, received feedback spring 2024
   - Needs review/merge approval (Lukasz/Michal to coordinate with Felix)
   - **Owner:** Benjamin (needs help from Lukasz/Michal to get merged)

2. **Benjamin to upload documentation to shared repository**
   - Update generic connector implementation guide (last updated 2022, changes from 2024)
   - Move S3-hosted partner/supplier docs to GitHub repository
   - Provide list of all links and documentation to team
   - **Owner:** Benjamin

3. **Benjamin to provide PR link and schedule review**
   - Send PR details to Lukasz after meeting
   - Schedule deeper technical review session with Michal if needed
   - **Owner:** Benjamin

4. **Document handover of supplier/partner inventory**
   - Current list: Zero active suppliers, Two active partners (Intermail, Sideshop)
   - Historical context: Geran Kosnokana (Dynamics—legacy, no longer paid)
   - Spreadsheet provided to Felix before parental leave
   - **Owner:** Michal (create or locate comprehensive inventory)

5. **CRM ID field migration project planning**
   - Evaluate feasibility of migrating legacy (pre-2020) integrations to entity-specific IDs
   - Could be joint project with team during fall
   - Would fix one-CRM-per-section limitation
   - **Owner:** TBD (discuss with Eric for prioritization)

### Unresolved Questions

1. **Infinite Redrive Behavior**
   - How many times should failed batches/consent messages be redriven?
   - Should there be a time limit or attempt limit?
   - Current: Infinite redrives (noted as room for improvement)
   - **Owner:** TBD (technical design decision needed)

2. **CRM ID Field Collision Fix Timeline**
   - Is fall the right time to attempt migration?
   - Should it be a joint project with integrations team?
   - What's the scope and effort estimate?
   - **Owner:** TBD (prioritization needed from leadership)

3. **Unified Data Production Usage**
   - When will first large-scale customer use Unified Data?
   - What monitoring/SLA commitments are needed?
   - Should there be an explicit "beta" phase?
   - **Owner:** Benjamin/Eric (product/customer success planning)

4. **Form Field Mapping Per-Form Support**
   - Should per-form CRM field mapping be prioritized?
   - Is current global outbound mapping sufficient for customers?
   - **Owner:** TBD (product roadmap decision)

5. **Legacy CMS Integration Maintenance**
   - Are open-source Drupal/Sitecore/FP Server integrations still requested by customers?
   - Who owns documentation and issue response?
   - **Owner:** TBD (determine support model)

---

## Related Documentation Provided

Benjamin indicated he will provide:
- All architecture diagrams (inbound/outbound flows)
- Collection of links and references in a central document
- GitHub-hosted implementation guides and partner agreements
- Supplier/partner inventory and contact information
- Links to generic connector API specifications
- Unified Data technical documentation
- Historical CMS integration repositories (open source)