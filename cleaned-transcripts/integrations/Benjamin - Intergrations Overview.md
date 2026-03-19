---
source_file: Benjamin - Intergrations Overview.txt
domain: Integrations
topics: [Architecture Overview, Inbound Sync (Delta & Full), Outbound Sync, Field Mappings, Consent Management, Integration Connectors, Partner Strategy, Unified Data, CRM Integrations, Justin Integration Layer]
speakers: [Benjamin Nouhaag (Integration Architect), Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1 (Eric)]
key_components: [Justin (Integration Middleware), Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager/Producer/Consumer, Profile List Sync, Outbound Manager, All Sub Worker (Audience Subscription Worker), Batch Production Worker, Outbound Worker, Retry Driver Lambda]
session_type: knowledge-transfer
---

## Session Overview

This is a conceptual knowledge-transfer session introducing a newly assigned team to the **Integrations domain** of APSIS One. Benjamin Nouhaag provides a high-level architectural overview of how the integrations platform works, covering the original mission (providing a unified customer view across disparate systems), the core design principles (standardization and modularity), and the current operational model. The session walks through both **inbound sync** (profiles, attributes, and consent from external systems) and **outbound sync** (activities and consent changes back to CRMs), explains the role of **Justin** (the integration middleware layer), and introduces the strategic shift from supplier-owned connectors to **partner-driven generic connector implementations**. A brief discussion of **Unified Data** — a feature for querying complex relational data from CRMs — is included as a forward-looking capability.

---

## Historical Context and Original Mission

### Why Audience/Justin Exists

Before 2013-2014, companies had many disconnected systems managed by IT departments, all out of sync with each other. Data was always lagging or missing when needed. **Profile Cloud** (later renamed **Audience** in APSIS One) was built to put a middleware layer in the middle to provide a holistic customer view and solve data synchronization problems.

### Original Design Principles

The integrations team's mission in APSIS One is to:

1. Ensure arrows (data connections) exist for as many external systems as possible
2. Provide the same **SLA (Service Level Agreement)** for integrations as APSIS provides for the rest of the platform — including uptime guarantees, 24/7 pager duty, and operational reliability
3. Standardize use cases for different system types (CRM, ecommerce, etc.) into one place
4. Work with suppliers who understand specific systems and minimize logic in external systems so the team can monitor and ensure uptime

> "By that I mean ensuring uptime, whatever we have in our contracts, having 24/7 pager duty and this can be very challenging when integrations since some logic will always reside outside of APSIS and that's why we've originally built things per a certain strategy."

### Standardized Use Case Example

In the ecommerce integrations, the team built a standardized **abandoned cart infrastructure** that:
- Ran once per hour
- Called standardized functionality to fetch carts from all connected systems
- Threw abandoned cart events into Audience for marketing automation to react to
- Had minimal system-specific connector code for each platform

---

## The Justin Integration Middleware

### Core Purpose

**Justin** is named after Justin Timberlake (a joke: "It's job is to keep everything in sync"). It serves as a generic infrastructure layer with pluggable **connector libraries** for each external system.

### Modularity Principle

The original design principle — and still the current one — is modularity:

- Justin provides generic infrastructure for all integrations
- Each connector library handles system-specific communication (HTTP requests, data translation, schema mapping)
- If a UI maps a Profile attribute to a Dynamics Contact field, the connector library only needs to:
  1. Fetch fields from the external system via HTTP
  2. Translate them to Justin's format
  3. Return them to Justin
  4. Serve them to the frontend

### Evolution of Connector Strategy

**Early approach (Legacy - still in place):**
- Worked with **Gera Konsultana** (a contractor) to build plugins for Microsoft Dynamics 365 with webhooks
- Tried to understand external system APIs deeply
- Ran into edge cases constantly (e.g., in Lime CRM, you can remove a contact-marketing relationship, but this isn't semantically equivalent to opting out, causing unexpected behavior)
- Lead to unexpected fixes 1-2 days before launch that required 3+ weeks of work

> "So we ran into a lot of these crazy little situations very often like one or two days before launch and they required like a three-week fix."

**Current approach (Generic Connector):**
- Provide an API spec/contract that external systems must implement
- External system partners implement the contract on their side
- APSIS only maintains a config file for each partner
- Shifts responsibility for compliance and correctness to the external system provider

---

## Inbound Sync Flow (Profile, Attributes, and Consent from External Systems)

### Step 1: Field Mapping Configuration

When a user installs an integration (e.g., FC Enterprise 12.1), they get a standard wizard that provides a URL and API key.

**The Mappings Page:**
- User maps fields from the external system to profile attributes in Audience
- Dropdown shows available fields from the external system
- Field names are auto-mapped where possible (e.g., "Cle" → "Sierra Mighty")

**Service involved: Integration Manager**
- Fetches schema from the external system using the appropriate connector library
- Routes requests based on the integration type:
  - `dynamics` → Microsoft Dynamics connector library
  - `fc_enterprise` → FC Enterprise connector library
  - Generic connector systems use internal mapping
- Returns available fields to the frontend

### Step 2: Mappings Manager and Justin's Laws

When mappings are saved, the **Mappings Manager** validates them:
- Ensures mappings don't create loops
- Enforces **Justin's Laws** (rules for valid mapping configurations)
- Stores mappings in its own database with dedicated user access to the mappings table

### Step 3: Webhook Subscription

During installation, the system:
- Subscribes to webhooks on the external system for contacts created/updated
- Updates webhook subscriptions whenever mappings change
- Some systems send updates to an official endpoint in Delta Sync Manager; others use custom webhook endpoints provided by the integrations team

### Step 4: Real-Time Delta Sync (Change-Driven)

When the external system sends a contact update in real time:

1. **Delta Sync Manager** receives the webhook
2. Message goes to an **SQS FIFO queue** with **message group ID = CRM ID** (ensures per-contact ordering)
3. **Delta Sync Worker** (ever-running ECS task):
   - Retrieves mappings from Mappings Manager (with caching, invalidated on updates)
   - Processes messages one at a time or in small batches
   - Applies sync conditions (see below)
   - Sends profile update messages to Audience

### Step 5: Consent/Subscription Mapping

Similar flow to field mappings, but for consent/subscription data:

- User maps external system consent fields/resources to Audience topics
- External systems may have:
  - Dedicated consent resources (preferred)
  - Boolean fields on contacts (if no dedicated resource exists)
  - In legacy connectors, defined as **virtual consent** (boolean fields) or **native consent** (dedicated resources)
- Generic connector systems simply expose what the external system provides
- **Integration Manager** fetches available consent basis from external system; Mappings Manager stores the consent mapping

### Step 6: Full Sync (Periodic Bulk Synchronization)

Runs after initial setup or on demand to sync all historical data.

**Architecture:**

- **Full Sync Manager** (ever-running ECS task):
  - Receives sync operation requests via POST
  - Spins up ad-hoc infrastructure for each sync job (reason: past queuing issues with FIFO SLAs)
  
- **Full Sync Producer** (runs per sync job):
  - Uses connector library to fetch paginated contact list from external system
  - Formats data in Justin's expected format
  - Puts messages on a **full sync queue** for this specific operation
  
- **Full Sync Consumer** (runs per sync job):
  - Consumes from the full sync queue
  - Runs same message processing logic as Delta Sync Worker
  - Applies field mappings and sync conditions
  - Sends profile updates to Audience

**Eventual Consistency Protection:**

During a full sync, the system must prevent processing of real-time delta sync messages that might already be in the full sync dataset, which could break consistency.

- Real-time webhook messages for the integration are **buffered in a queue**
- If a full sync is active for the integration, messages are **delayed** by setting a visibility timeout on the SQS message
- After full sync completes and all messages are processed by Audience, buffered messages are released and processed in order
- This ensures: if Benjamin receives two updates in the external system during full sync, both are processed in order after the full sync completes

**Completion Signaling:**

The Full Sync Manager displays completion status:
- **"Downloaded and processed"** (Successful): All profiles downloaded and ingested into Audience
- **"Downloaded and being processed"** (Finalizing): Profiles downloaded but Audience ingestion is still catching up
- Only moves to "Successful" when ingestion delay catches up to the moment the full sync wrote its last message to Audience

### Step 7: Sync Conditions (Filtering)

Users can create conditions that determine whether a profile should be synced:

- Example: `active_field == true` — only sync active contacts
- Implemented as mappings stored in Mappings Manager
- Evaluated during:
  - Delta sync: If condition not met, message ignored
  - Full sync: If condition not met, message ignored
- **Inbound deletion**: If a profile previously synced but now fails the condition, it can be deleted from Audience (togglable via feature plan)

Example [Benjamin]: *"Usually you would have some kind of bool field active equal to true. So then maybe maybe I only want to sync my active contacts."*

### Step 8: Profile Lists (Static and Dynamic)

Users can import segment-like lists from external systems and apply tags to matching profiles.

**Terminology Note:**
- The term "profile" in this context is unfortunate — it refers to **CRM lists**, not APSIS profiles
- CRM systems have this concept under different names:
  - Dynamics: "Markets" (dynamic or static lists)
  - FC Enterprise: "Queries" (dynamic) and "Profiles" (static)
  - Dynamic: Auto-maintained by CRM queries
  - Static: Manually maintained

**How It Works:**

1. User imports a CRM list (e.g., "Newsletter" in FC Enterprise)
2. System adds a tag (e.g., "Newsletter") to all profiles on that list
3. If the user re-imports and a profile no longer exists on the list, the tag is removed
4. The system tracks which operation added which tags for each profile

**Recurring Imports:**

- Can be set to run on a schedule (currently 6:00 AM every day, extensible to other schedules)
- **CloudWatch Event** triggers a **Profile List Sync Lambda** which puts jobs on the **profile list sync queue**
- **Profile List Sync Consumer** spins up one worker per job (one at a time per integration to avoid DDoS)
- **Profile List Sync Worker**:
  - Fetches every contact on the list
  - Holds data in memory (different from delta/full sync architecture)
  - Sends a single tag request to Audience
  - Has had stability issues in the past due to lack of queueing; now more stable

> "This one actually fetches every contact on the list and does not work with a queue or anything, just holds it in memory and sends a tag request all the way to audience. So this is where we've had some problems in the past..."

---

## Outbound Sync Flow (Activities and Consent Back to External Systems)

### Overview of Outbound Capabilities

When users create **activities** (emails, SMS, MA flows, events, forms) in APSIS One, the system can sync them to the external CRM system along with their engagement events (sent, delivered, opened, clicked, etc.). The system does **not** sync surveys or arbitrary events.

> "Whenever I create any kind of activity, so e-mail, SMS form, even an MA flow, we can sync this to the external system."

**Attribute directionality limitation:** Only **consent is bidirectional**; attribute changes sync only inbound (CRM → APSIS), not outbound.

> "As as then just said, like we are syncing the consent changes back to the CRM, but we do not sync attribute changes back to the CRM like the consent is bidirectional but the attributes only come to Apsis, but not from Apsis. And that's that's because historically customers have not wanted that. They feel like the CRM should be the data master."

### Step 1: Activity Creation and CRM Sync Option

When a user creates an email in APSIS One:

1. **Frontend** calls Integration Manager to check: "Are there any installed integrations that can sync emails?"
2. Integration Manager returns list of capable integrations
3. Frontend shows a **"CRM Sync"** tab with toggles for each integration
4. User selects which integrations to sync to (e.g., "FC Enterprise 12.1")
5. User also sees integration-specific settings (e.g., which CRM field receives certain data)

### Step 2: Outbound Manager Creates CRM Record

When the user clicks "Sync with [CRM]" and sends the activity:

- **Outbound Manager** acts as a bridge between the frontend and the CRM system
- CRM is instructed to create a record corresponding to the activity (e.g., an email activity)
- The activity now has a relationship to the CRM system

### Step 3: Event Processing Pipeline (Batching for Efficiency)

As the activity is sent and engagements occur (opened, clicked, delivered, etc.), Audience generates events:

1. **All Sub Queue (Audience Subscription Queue)**:
   - SQS FIFO queue
   - Receives all engagement events from Audience
   
2. **All Sub Worker (Audience Subscription Worker)** (ever-running ECS task):
   - Verifies that this activity should be synced for this integration
   - **Checks with Integration Manager** whether integration is installed for this account
   - **Verifies with Mappings Manager** whether outbound is enabled for this integration
   - Places messages on a **Kafka partition** (not SQS)
   - Reason for Kafka: Messages are interleaved by integration ID and need batching

3. **Batch Production Worker** (ever-running ECS task):
   - Continuously pulls from the Kafka partition
   - **Groups messages by integration destination**
   - **Batches until either:**
     - Largest batch reaches 200 KB (stays well below SQS 256 KB limit)
     - OR 1 second has elapsed
   - Example batching:
     ```
     A1 A2 A3 A4 → Batch A (for integration A)
     B1 B2 B3     → Batch B (for integration B)
     C1 C2         → Batch C (for integration C)
     ```
   - Puts ready batches on **SQS FIFO queue**

> "Just puts things in batches that are ready to go to the external system, so groups them by which integration it's supposed to go to, and they keep polling until either the largest batch has surpassed 200K or it has been polling for more than one second."

4. **Outbound Worker** (ever-running ECS task):
   - Consumes ready batches from SQS queue
   - Uses connector library to send batch to external system
   - Each integration's connector library decides what to do with the batch

### Step 4: Failure Handling and Retry

If outbound delivery fails:

1. Message goes to **Dead Letter Queue** after retries
2. **Retry Driver Lambda** runs every morning at 6:00 AM
3. Puts all DLQ messages back on the outbound queue for re-processing
4. **Currently: infinite retry** (identified as a gap for improvement)

> "Oh yeah, infinitely. Yeah, that that that's a bit of a room for improvement."

**Eventual consistency caveat:** When redriving failed messages, eventual consistency is no longer guaranteed. Each external system is responsible for idempotency:

- **Batch ID generation**: APSIS generates a unique ID for each batch
- **External system responsibility**: Must track which batch IDs have already been processed
- **Contract requirement**: Detailed in integration specification/delivery process document
- If external system ignores idempotency requirement, consistency issues are their responsibility

### Step 5: Consent Syncing (Bidirectional)

When a profile opts out inside APSIS One, the system must write the opt-out to the external CRM:

1. Opt-out message placed on **All Sub Queue**
2. **All Sub Worker** processes it (bypasses batch production — consent is low volume)
3. Consent message placed directly on **Outbound Queue**
4. **Outbound Worker** sends singular consent message to CRM
5. If it fails, goes through the same retry carousel as activity events

**Asymmetry in redrives:** When redriving failed consent messages:
- **Do NOT redrive opt-in messages** (once redriven, no eventual consistency guarantee)
- **DO redrive opt-out messages** (safer assumption for playing it safe)

> "We don't redrive any consent message saying opt in equals true. We only redrive consent messages saying opt in equals false because once we do a redrive, we no longer have eventual consistency. And so we we basically taking playing it safe and we sync only opt out to the redrive."

### Step 6: Form Submissions and Attribute Mapping Asymmetry

**Current limitation:** Form submissions cannot map form fields directly to CRM fields. Instead:

- When a form is submitted, engagement events are placed on All Sub Queue
- **All Sub Worker** does not use the form submission event data directly
- Instead, it fetches **outbound attribute mappings** from the profile
- Only mapped attributes (not form fields) are sent to the CRM

**Desired state (not implemented):** Users should be able to map form fields directly to CRM fields per form, not just use profile attributes.

**Reason for current limitation:** Political/organizational factors (pre-2022 decision).

> "To say it nicely in 2022 this was not possible. So instead if I go in here I would have another tab here called outbound mappings which is essentially my field mappings but reversed. So from APSIS to the CRM."

### Step 7: Feature Flags and Integration Capability Control

The system controls what each integration can do via:

1. **Backend configuration** (`installer_options`): Specifies which features are enabled for each integration type (e.g., Dynamics supports feature X, Salesforce doesn't)
2. **Instance capability detection**: When installing, the system checks the specific CRM instance to see what it supports (different versions may have different features)
3. **UI rendering**: Tabs (field mappings, consent mappings, outbound mappings, profile lists, etc.) only appear if enabled for that integration type AND that instance

Example: If a generic connector partner hasn't implemented outbound mappings, that tab won't appear.

---

## Multi-Entity Integrations and the CRM ID Problem

### The CRM ID Limitation (Legacy Issue)

**The Problem:**

The system has a critical architectural debt: all integrations map profile data to a single `CRM_ID` field in Audience, causing a blocking constraint.

> "The field mappings, the map to CRM ID. This is very, very, very unfortunate. This is an old Jonas super urgent release something now situation."

**Why this is bad:**

- Users can only install **one CRM integration per section**
- If you install Dynamics on Section A and another CRM on Section B, that's fine
- But if you want Dynamics on Section A AND Salesforce on Section A, you cannot because both try to map to the same `CRM_ID` field
- This was a shortcut taken years ago ("Jonas super urgent release") and never fixed

**The workaround:**

The team maintains a **unique key space per integration** (not per CRM, but per instance):
- `salesforce_key_space`
- `dynamics_key_space`
- `dynamics_sideshop_key_space` (different from the native Dynamics one)
- etc.

But all of these still map to the `CRM_ID` field in the outbound direction, preventing multi-CRM on same section.

### Multi-Entity Support (Proper Implementation Post-2020)

For integrations built after 2020, the system properly handles multiple entity types:

- Can have separate tables for "contacts" and "leads"
- Each entity type gets its own dedicated ID attribute (e.g., `lead_id`, `contact_id`)
- Render as separate mapping sections in the UI

**Legacy integrations (pre-2020):**
- Depend on single CRM ID field
- Would require migration to separate ID attributes
- Identified as a possible joint project between integrations team and new team "during the fall"

---

## Partner and Supplier Strategy

### Historical Approach: Supplier-Owned Connectors (Legacy)

**Example: Microsoft Dynamics 365 (Native Connector)**

- **Supplier:** Gera Konsultana (Stockholm-based contractor)
- **Model:** Pay supplier by the hour; maintain SLA; pay support costs
- **Architecture:**
  - Supplier builds plugin on customer's Dynamics instance with webhooks
  - APSIS maintains connector library in Justin
  - Any feature request requires coordination with supplier, time, and cost
- **Problem:** Inflexible, slow to innovate, expensive to maintain across multiple CRM systems

> "It's very hard to add something and then scale that across multiple systems when we have to own kind of ideation on what should happen on the other side."

### Current Approach: Partner-Driven Generic Connector

**The Shift:**

Instead of paying suppliers to build custom connectors, APSIS now defines a **generic connector specification** that external system vendors implement themselves.

**Supplier vs. Partner Model:**

- **Supplier:** You pay them; they build for you (e.g., old Gera Konsultana relationship)
- **Partner:** They implement your spec themselves; they charge their own customers

**Active Partners:**

1. **Sideshop** (Dynamics 365 CRM by Sideshop)
   - Implements generic connector specification
   - Customers pay Sideshop ~200 EUR/month for the integration
   - Sideshop gets integration revenue
   - APSIS maintains only the generic connector spec, not integration code per customer
   - Sideshop can build additional features outside the generic spec (their own innovation)
   - Can bring feature requests back to APSIS for generic-level improvements

2. **Intermail** (Loyalty and Relation Plus)
   - Relation Plus: Implements generic connector; own revenue model
   - Intermail Loyalty: Custom-built (pre-generic connector era)
   - Drives customers to APSIS; gets revenue from integration

> "Sideshop should be getting should be billing for the integration revenue. So anyone wanted to use Dynamics, they'll just get the Sideshop integration. They pay whatever 200 EUR a month to Sideshop for the integration and Sideshop then just implement our generic connector interface on their side."

### Standardized Partnership Documents

Benjamin created templated documents for working with partners/suppliers:

#### 1. **Partnership Agreement (for Partners)**
- Defines what partners can do
- Termination conditions
- Partners own support and all non-APSIS-controlled aspects

#### 2. **Supplier Delivery Process**
- Defines deliverables partners must provide to APSIS
- Requirements:
  - Walk-through of build process
  - 75% test code coverage (enforced by CI/CD)
  - Detailed installation instructions
  - List of user roles required
  - Complete file paths for all created/altered items
  - Database tables and entities created/altered
  - Clear definition of done

#### 3. **SLA Template**
- Response times for different severity levels
- Maps to APSIS internal P1/P2/P3 definitions
- Critical (P1) vs. non-critical (P3+)
- Standardized across supplier agreements

#### 4. **Generic Connector API Specification**
- Details the contract external systems must implement
- Specifies required API endpoints, data formats (Apache Columnar for unified data queries)
- Maintained in S3 with versioning (latest always available)

#### 5. **Generic Connector Implementation Guide**
- Lengthy document showing how to implement the generic connector spec
- From external system's perspective
- Last updated 2022, with 2024 updates pending merge

> "There is another one with updates from 2024, but it's pretty big change. So I'm going to try to get all of that merged in here by tomorrow and updated on the readme."

### Current Active Agreements

- **Zero active supplier contracts** (APSIS chooses not to pay for ongoing support)
- **Two active partners:**
  - Sideshop (Dynamics 365 CRM, SuperOffice)
  - Intermail (Loyalty, Relation Plus)

### Historical Partners/Suppliers (Discontinued)

- **CMS integrations** (Drupal, FP Server, Sitecore): Different model, not standard inbound/outbound
  - Worked via segment dropdown on content edit
  - User could segment which content to show to which APSIS segments
  - Used segment evaluation API
  - All now open-sourced and repositories available

### Document Locations

All partnership documents are maintained in:
- **S3 folder** with domain-assigned name (auto-distributed via links)
- **GitHub repository** (to be migrated by Benjamin)
- **Shared OneDrive folder** (for internal team documentation)

---

## Operational Monitoring and Problem Resolution

### Dead Letter Queue Monitoring

The system has alerts on the **Dead Letter Queue** for failed outbound messages:

- Messages consistently fail to process due to various issues:
  - External system bugs
  - Timeouts
  - Network problems
  - CRM-specific issues (some systems fail, others succeed)
  - Customer-provided bad data (e.g., incorrectly imported consent)

> "There are a lot of different reasons why it could end up. The customer can also have incorrectly imported consent in apps that we cannot process."

### Operational Meetings

During periods of high operational issues, the team held **Monday operational meetings** to:
- Review customer integration problems
- Analyze dead letter queue contents
- Identify patterns (e.g., "we've been reporting this issue to Vendor X for 2-4 weeks with no fix")
- Discuss prevention strategies

> "At a time when we had a lot of operational problems, we started having an operational meeting once per Monday where we looked over various customer integration problems and also look through what's in the dead level queue..."

**Benjamin's recommendation:** Resume these meetings if operational problems resurface.

---

## Marketing Automation Integration Node

### Use Case Example

Benjamin demonstrates creating an MA flow that:

1. **Listens** for event invitations
2. **Waits** two days
3. **Checks** if the invitee confirmed attendance
4. **If confirmed:** Do nothing
5. **If NOT confirmed:** Create a task in the CRM for a salesperson to call and confirm

### Implementation

MA flows can include **integration nodes** that create records in the CRM:

```
Listen: Event Invitation
  ↓
Wait: 2 days
  ↓
Branch: Profile.event_tool_status == "confirmed"?
  ├─ YES: End (do nothing)
  └─ NO: [CRM Node] Create Task
         - Type: Phone Call
         - Field mapping: Urgency=9, etc.
         - Assigned to: [CRM-defined value]
```

### Processing Path

Integration nodes in MA flow follow the same outbound processing pipeline as activity events:

1. Node execution → message to **All Sub Queue**
2. All Sub Worker → **Kafka partition**
3. Batch Production Worker → **SQS batch queue**
4. Outbound Worker → **CRM connector library** → External system
5. On failure → **DLQ** → **Retry Driver** (6 AM daily)

**Why batching is essential:** MA flows can trigger for thousands of profiles simultaneously (e.g., 500,000 people in a scheduled segment). One-by-one processing would cause too many requests and rate limiting.

> "As far as we know we could be getting 500,000 people into this flow at 12:00 every day, in which case we can split and send one by one kind of messages."

### Limitations

The system shows validation errors if schema fetch fails from the external system (e.g., broken integration authentication), preventing the user from configuring the node.

---

## Unified Data: Advanced Querying for Complex CRM Data

### The Business Problem

Customers often want to query complex relational data from CRMs and use it for personalization or segmentation.

**Example: Paul, head of sales for an event company (AMI Events)**

Paul wants to send personalized emails to **CEOs of European companies whose employees attended last year's trade show**, saying:

> "It seems like Imashek had a blast at NG Poland last year. How do you feel about sending more reps from FSC this year? We look forward to boosting your business in China."

**Why this is hard:** 

APSIS profiles are flat documents (one record = one person). CRM systems have relational tables:
- `Event` → has many `Attendees` → related to `Person`
- `Person` → has many `Roles` (different per company)
- `Role` → belongs to `Company`

To find CEOs of companies with attendees, you need to join across multiple tables. Traditional inbound sync flattens this data and loses the relationships.

### Current Solution: Unified Data (Implemented for Maxo, Reusable)

**Architecture:**

1. **External System (CRM)**: Defines pre-made queries that expose complex data
   - Example: "CEOs of companies who had attendees at Event X last year"
   - Query specifies selected fields/columns available to export

2. **Integration Layer (Justin)**: Acts as a bridge
   - Audience triggers a query/export and specifies a CRM query to join
   - Integrations opens an **HTTP/2 streaming connection** to Justin
   - Justin fetches paginated data from CRM (can be millions of rows)
   - Converts to **Apache Columnar format** on the fly
   - Streams back to Audience without buffering entire dataset in memory

3. **Audience (Analytics/Email tool)**: Joins Unified Data with internal exports
   - User creates an email campaign
   - Specifies: "Personalize with joined data from CRM query: CEOs of companies with attendees"
   - Audience fetches CRM data via streaming, joins locally, sends emails with personalized data

### Why HTTP/2 Streaming is Critical

- External system might return **millions of rows**
- Cannot fit in a single request/response
- Cannot make multiple sequential requests (loses context)
- HTTP/2 allows streaming: fetch N rows, stream, fetch next N rows, stream, repeat until done
- Integrations translates on the fly to Apache Columnar format

> "So we might get say 2,000,000 entries back from the CRM system and you do not want to get all of that in one request, nor would this feature actually work if you were to retrieve multiple requests."

### Current State and Future Applicability

**Status:**
- Built for **Maxo** (APSIS product suite) during fall/winter 2024
- Delivered on time
- One Berto customer tested it
- Has not seen wide adoption (Maxo didn't bring in many customers)
- Documented with full technical detail and API contracts

> "We have one Berto customer, but I don't know if you can say that it's really very tried and verified. One customer has said it, but it hasn't been used by 100 customers and proven to be stable."

**Reusability:**
- Implementation is generic — works with any CRM
- No system-specific code except in connector library
- Can be enabled for other CRM systems if customer requests this capability
- Worth maintaining because the capability will be requested again

### Documentation

Comprehensive documentation available:
- User experience flows
- Technical contract between Justin, email tool, Audience, and CRM
- Every API call with example payloads
- Should be operationally maintainable with good docs even with limited adoption

---

## Architecture Diagrams and Supporting Materials

Benjamin mentioned that diagrams for **inbound and outbound flows** will be shared with the new team. The inbound diagram covers:
- Field mappings, consent mappings, delta sync, full sync, profile lists, sync conditions

The outbound diagram covers:
- Activity creation, CRM sync, event processing, batching, retry flow

Abandoned cart (legacy, now torn down) would appear on both diagrams but is no longer in use.

---

## Key Takeaways

1. **Mission-Driven Architecture:** The integrations platform exists to provide unified customer views across disparate systems with APSIS-level SLAs. Early designs were too ambitious and led to operational challenges.

2. **Justin as Central Middleware:** Justin's modularity principle is core — generic infrastructure with pluggable connectors. This allows standardization while supporting diverse external systems.

3. **Inbound is Straightforward:** Field mappings, consent mappings, delta sync (real-time), and full sync (periodic) are well-established. Sync conditions filter which profiles to import.

4. **Outbound is Complex:** Events (not attributes) are sent outbound. Batching is essential for performance. Consent is bidirectional; attributes are not. Retry is infinite (room for improvement).

5. **CRM ID Debt:** The `CRM_ID` field limitation prevents multiple CRM integrations per section. Should be addressed via migration to per-integration ID fields.

6. **Partner Model is the Future:** Shift from supplier-owned connectors to partners implementing generic connector spec. Reduces operational burden and scales better. Sideshop (Dynamics) and Intermail (Loyalty, Relation Plus) are current active partners.

7. **Unified Data is Powerful:** For complex relational data queries from CRMs, the unified data feature (HTTP/2 streaming + Apache Columnar format) is reusable and proven (albeit not heavily adopted yet).

8. **Operational Vigilance Required:** Dead letter queues need monitoring. Monday operational meetings useful when issues are frequent. External system bugs, timeouts, and data quality issues are common causes of failures.

9. **Documentation Gaps Being Addressed:** Generic connector specs, implementation guides, supplier/partner agreements, and SLA templates are being migrated to GitHub for team continuity.

10. **Retention of Complexity:** The system handles many edge cases (eventual consistency, idempotency, multi-entity support, feature flag controls). New team should read through partnership documents and implementation guides to understand full scope before changes.

---

## Unresolved Questions and Action Items

### Benjamin's To-Do Items:

1. **Merge Generic Connector Implementation Guide PR**
   - Last reviewed: Spring 2024
   - Pending: Updates from 2024 release
   - Status: Benjamin will merge updated version by next day
   - Action: Needs review/approval (Speaker 1 / Eric will help)

2. **Upload Editable Documents to GitHub**
   - Partnership agreement, supplier delivery process, SLA template, generic connector API spec
   - Current location: S3 folder (personal documents)
   - Destination: GitHub repository (team repository)
   - Format: Editable (proposed: browser-based docs or .docx files)
   - Action: Team to decide on storage format and method

3. **Create Handover Document with All Resources**
   - Links to: presentations, process/policy documents, GitHub PR, partner agreement, SLA template, generic connector specs, unified data documentation, partner/supplier inventory
   - Format: Requested as GitHub Quick Document or shared OneDrive folder
   - Action: Benjamin to provide; Speaker 1 to move to appropriate location

4. **Create Inventory of Active/Historical Partners and Suppliers**
   - Benjamin had created a spreadsheet for Felix before parental leave
   - Current status: Brief (only 2 active partners listed)
   - Needed: Expanded to include historical relationships with contact info
   - Action: Benjamin to recreate from memory if spreadsheet not found

### Questions Not Fully Explored (Suitable for Technical Deep-Dive with Eric):

1. **How exactly do sync conditions get applied during full sync** (edge cases)?
2. **Why was SQS FIFO initially chosen for delta sync, then Kafka chosen for outbound event batching?** (Different trade-offs)
3. **What happens if a profile is synced to the CRM, then the CRM deletes it, then it tries to sync again?** (Idempotency and upsert logic)
4. **How are connector library versions managed and deployed?** (Breaking changes, backwards compatibility)
5. **What is the exact scaling limit of batch production before 200KB is hit?** (1000+ batches for 1M events — is this a problem?)
6. **Are there rate limits on external CRM systems that could cause outbound failures?** (Not discussed in detail)

---

## References for Follow-Up

- **Generic Connector API Specification**: Available in S3 folder; latest version always maintained
- **Generic Connector Implementation Guide**: 2022 version with 2024 updates pending merge
- **Partnership Agreement Template**: Included in shared documents
- **Supplier Delivery Process**: Included in shared documents
- **SLA Template**: Included in shared documents
- **Unified Data Technical Documentation**: Comprehensive; should be preserved and referenced
- **Historical CMS Integrations**: Drupal, FP Server, Sitecore repositories (now open-source)

---

## Session Metadata

**Duration:** 1h 31m 56s  
**Recording Started:** October 13, 2025, 7:50 AM  
**Break Taken:** Yes (5-6 minutes, ~1:09:48-1:15:48)  
**Attendees:** Benjamin Nouhaag, Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1 (Eric)  
**Recording Ended:** Before handover discussion items (organizational, not technical)