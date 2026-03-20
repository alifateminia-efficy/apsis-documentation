---
source_file: Benjamin - Integrations Overview.txt
domain: Apsis One - Integrations
topics: [Architecture Overview, Inbound Sync (Contacts/Profiles), Outbound Sync (Events/Activities), Integration Manager, Field Mappings, Subscription Mappings, Full Sync, Delta Sync, Profile Lists, Webhook Processing, Queue Management, Batching, Dead Letter Queue, Partner and Supplier Strategy, Unified Data, Generic Connector, Legacy vs. Modern Integrations]
speakers: [Benjamin Nouhaag, Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani, Speaker 1/Eric]
key_components: [Justin (Integration Middleware), Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager, Full Sync Producer/Consumer, Profile List Sync, Outbound Manager, All Sub Queue (Audience Subscription Queue), Batch Production Worker, Outbound Worker, Connector Libraries, Kafka, SQS, ECS Tasks, Generic Connector, Unified Data/Apache Columnar Format]
session_type: knowledge-transfer
---

## Session Overview

Benjamin Nouhaag delivered a comprehensive conceptual and architectural overview of the Apsis One Integrations domain to incoming team members. The session covered the original mission of the integrations team (ensuring real-time data synchronization across customer CRM and marketing systems), the core architecture including the **Justin** middleware layer, detailed workflows for inbound profile synchronization (contacts, field mappings, consent), outbound activity syncing (emails, SMS, forms, MA flows), queue-based processing patterns, and a shift in vendor strategy from supplier-controlled connectors to partner-based generic connector implementations. The session also introduced the Unified Data capability (developed for Maxo) as a reusable solution for complex cross-system data queries. The KT emphasized both technical architecture and organizational/contractual approaches to managing external integrations.

---

## Original Mission and Strategic Context

### Why Integrations Exist

[Benjamin Nouhaag]: The Integrations team's original mission emerged from a real problem observed in enterprise environments around 2013-2014: companies had many disparate systems, IT departments were manually integrating them, and there was no holistic customer view. Data was always out of sync and unavailable when needed. The concept of **Profile Cloud** (later renamed **Audience** in Apsis One) was built to solve this by placing a central hub between all these systems.

> "Our mission as an integrations team in Apsis One was to ensure that these arrows exist for as many systems as possible and provide in these arrows the same SLA as APSIS provides for the rest of Apsis One."

[Benjamin Nouhaag]: This means ensuring uptime commitments, 24/7 pager duty, and maintaining SLAs even though some logic will always reside outside Apsis. The original strategy was to **standardize use cases for different kinds of systems** (CRM, ecom) into one place and work with suppliers who understand each system deeply while keeping external system scope minimal for observability.

### Classic Example: Abandoned Cart Infrastructure

The abandoned cart standardized use case demonstrates the original approach:
- A standardized job runs once per hour
- Calls standardized functionality to fetch carts from all connected ecom systems
- Throws abandoned cart events into Audience for MA to react
- System-specific pieces only handle that particular system's communication

---

## Core Architecture: Justin and Connector Libraries

### Justin: The Integration Middleware

[Benjamin Nouhaag]: Justin (named after Justin Timberlake because "his job is to keep everything in sync") is the generic infrastructure layer that enables modular integration with external systems.

**Design Principle:** Modularity. Justin provides a generic infrastructure and connector libraries ensure easy communication with each external system.

**Example Flow:** If a UI allows mapping profile fields to contact fields in Microsoft Dynamics:
1. Only the connector library needs an HTTP request to fetch fields from Dynamics
2. Translate to Justin's expected format
3. Return fields to front end

### Connector Library Pattern

The integration pattern evolved through two strategies:

**Legacy Supplier-Driven Approach (Still Running):**
- Worked with suppliers who deeply understood the external system
- Example: **Geran Kosnokana** built a Dynamics 365 plugin with webhooks and API guidance for Apsis
- Problem: Inflexible and increasingly problematic because even with good API understanding, edge cases emerged frequently (e.g., Lime CRM allowing contact opt-out OR removing contact-marketing relationship as separate operations, neither cleanly mapping to marketing consent)

**Modern Generic Connector Approach (Current Direction):**
- Provide an API spec/contract that external systems must fulfill
- External system implements the contract; Apsis only needs a config file
- More scalable across multiple systems but requires partner buy-in

---

## Inbound Synchronization: Profiles and Contacts

### Integration Installation and Field Mapping

When a user installs an integration (e.g., FC Enterprise 12.1):
1. Standard wizard collects URL and API key for the external system
2. User lands on the **field mapping page**

**Integration Manager Service** (currently an ECS task, formerly Lambda):
- Handles basic operations when interacting with external systems
- Uses connector libraries keyed by system type (Dynamics, FC Enterprise, or generic connector)
- Fetches schema from external system and returns to front end

[Benjamin Nouhaag]: "If the URL says dynamics, it uses the Microsoft Dynamics connection library. If it says FC enterprise, it uses FC enterprise. If it's something running on the generic connector, there's an internal mapping."

### Mappings Manager and Justin's Laws

**Mappings Manager Service:**
- Stores field mappings in its own database with dedicated user access
- Enforces **Justin's Laws** (rules preventing loops and other invalid mapping configurations)
- Updates webhooks whenever mappings change

---

## Real-Time Delta Sync (Webhook Processing)

### Webhook Subscription and Processing Flow

**During Installation:**
1. Subscribe to webhooks for all needed events in external system
2. As mappings change, update webhook subscriptions

**When External System Updates:**
1. Contact created or updated in CRM
2. Webhook sent to official Apsis endpoint (Delta Single Manager) OR custom webhook endpoint (without Apsis visibility)
3. Data placed on **SQS FIFO queue** with message group = CRM ID
4. **Delta Sync Worker** (ECS task) consumes queue:
   - Asks Mappings Manager which mappings exist for this account (with caching and cache invalidation)
   - Applies field mappings
   - Sends message to Audience

[Benjamin Nouhaag]: "Whenever we install, we subscribe to Webhooks, we save mappings, then we're going to treat those Webhooks differently because we use those mappings to know how to deal with the profile."

---

## Subscription/Consent Mapping

**Consent as Bidirectional Data:**
- Map consent basis from external system (either a dedicated consent resource or fields on contact card)
- Legacy terminology: **virtual consent** (boolean fields) vs. **native consent** (dedicated resources)
- Mappings work identically to field mappings:
  1. Integration Manager fetches available consent basis from external system
  2. Fetch available topics from Audience
  3. Mappings Manager stores mapping
  4. During webhook processing, use mapping to interpret consent

**Important Caveat:** Consent changes flow bidirectionally (inbound from CRM and outbound from Apsis), but attribute changes are **unidirectional—only inbound**. [Benjamin Nouhaag]: "Historically customers have not wanted attribute changes going back to the CRM. They feel like the CRM should be the data master."

---

## Full Synchronization (Batch Import)

### Full Sync Operation and Infrastructure

Full Sync is typically run after initial setup to ensure all existing contacts are synced.

**Full Sync Manager Service** (ECS task):
- Provides list of existing, historical, and currently running full syncs
- When user clicks "Start new sync", spins up **entire stack on demand** for that specific job
- [Benjamin Nouhaag]: "It's a bit overkill, but we've had some queuing issues with FQSOA."

**Full Sync Producer:**
- Downloads everything paginated from external system
- Uses connector library to fetch contacts in system's format
- Translates to Justin's expected format
- Places on **full sync queue** for this operation

**Full Sync Consumer:**
- Consumes queue, processes messages same way as Delta Sync
- Asks Mappings Manager for field and consent mappings
- Sends profiles to Audience

### Buffering During Full Sync (Eventual Consistency)

**Critical Pattern:** While a full sync is running for an integration, real-time delta messages must be **buffered** to maintain eventual consistency.

**Why this matters:** If a contact update comes in mid-full-sync, we cannot determine if it has already been processed. Solution:
1. When Delta Sync receives message during full sync, set message visibility timeout
2. Hold buffered messages in queue
3. After full sync completes, process buffered messages in arrival order
4. Ensures we never break eventual consistency

[Benjamin Nouhaag]: "If multiple messages come for Benjamin, we can buffer them up here and then we take them all afterwards. This is a solution we often wish we didn't have, but we didn't [have the bandwidth to solve differently]."

### Full Sync Completion Lifecycle

Full Sync has three states:
- **Running**: Producer and consumer threads are actively processing
- **Finalizing**: Producer threads completed (no more data from CRM), but consumer is still processing queue; system waits for Audience ingestion delay to catch up to the last write timestamp
- **Successful**: All data downloaded, processed, and ingested by Audience

---

## Profile Lists (Static and Dynamic Lists)

### Abstraction of CRM List Concepts

Different CRM systems have different terminology but the same concept:
- **Dynamic lists** (e.g., Dynamics "Market", Salesforce lists with query criteria): Automatically maintained by a query evaluated at runtime
- **Static lists**: Manually maintained lists of contacts

Apsis abstracts these into **static profile list** and **dynamic profile list**.

[Benjamin Nouhaag]: "In FSC enterprise, a dynamic list is called a query and a static list is called a profile. So that's why it says profiles here. It has nothing to do with the profiles in APSIS—it's a very unfortunate [naming] thing that we've never had time to look at."

### Profile List Import Mechanism

Importing a profile list does **not** create profiles; it **tags** profiles that already exist (via full sync or delta sync).

**Example:** Import "Newsletter" list from CRM:
1. Fetch all contacts on that list
2. Add "Newsletter" tag to each profile
3. If contact removed from list on re-import, remove tag

**Recurring Profile List Syncs:**
- Can be scheduled (e.g., 6:00 AM every morning)
- Triggered by CloudWatch event → Lambda → Profile List Sync Manager → Profile List Sync Queue
- Consumer spins up one worker per job per integration (serial, not parallel, to avoid DDOS'ing customer infrastructure)
- Fetches all contacts on list, holds in memory, sends tag request to Audience

[Benjamin Nouhaag]: "These failed a lot before we addressed various edge cases because there was no queue. But now they're quite stable."

---

## Sync Conditions (Filtering)

### Conditional Synchronization

Allow customers to specify conditions under which a contact should be synced.

**Example:** Only sync active contacts by creating mapping where `active == true`

**During Processing:**
- Delta Sync or Full Sync checks condition on each message
- If condition not met, message ignored
- If feature enabled and condition becomes false, profile is deleted from Audience (deactivation by feature flag)

---

## Outbound Synchronization: Activities and Events

### High-Level Pattern

Any activity created in Apsis One (Email, SMS, Form, MA flow) can be synced back to external system.

[Benjamin Nouhaag]: "Whenever I create any kind of activity—email, SMS form, even an MA flow—we can sync this to the external system. The only thing we can't sync today are events and surveys."

**Sync Lifecycle for Activity:**
1. User creates activity and checks "sync to [CRM]" checkbox
2. **Outbound Manager** bridges front end and CRM to create corresponding record
3. As events arrive (sent, delivered, opened, clicked), transfer in batches to external system

### Email Sync Example (Applies Similarly to SMS, Forms, etc.)

**Frontend Integration:**
- After user creates email, CRM Sync tab appears
- Shows available integrations that can receive email events
- User checks box to enable sync

[Benjamin Nouhaag]: "The front end is actually asking integrations back end and said 'hey, is there integration installed that can sync emails?' If we say yes, here it is. Then they're gonna show this and then the user is allowed to click this."

---

## Event Batching and Processing Pipeline

### Audience Subscription Queue and Batch Production

**Problem:** One-by-one syncing of events could generate excessive traffic. If sending email to 1M people, that's 1M+ individual messages.

**Solution:** Batch production pipeline

**All Sub Queue (Audience Subscription Queue):**
- FIFO SQS queue consuming all events from Audience (sent, delivered, opened, clicked, consent changes)
- **All Sub Worker** (ECS task) reads queue and:
  1. Asks Integration Manager if integration is installed for activity
  2. Verifies activity should be synced
  3. Places messages on **Kafka partition**

[Benjamin Nouhaag]: "In this example, we have 4 messages intended for installation A (A1, A2, A3, A4), 3 messages for B (B1, B2, B3), and 2 for C (C1, C2). These are juggled up and mixed with each other."

**Batch Production Worker** (ECS task):
- Continuously polls Kafka partition
- Groups messages by integration
- Creates batches when either:
  - Largest batch exceeds 200KB, OR
  - Polling for more than 1 second
- Reason for 200KB: SQS 256K limit; batches placed on **SQS FIFO queue**

[Benjamin Nouhaag]: "For a million events it could be over 1000 batches, but we never had any problems. But this is something that's been on our list—something that could be good to improve by choosing a different queue."

### Outbound Worker and Dead Letter Queue

**Outbound Worker** (ECS task):
- Takes ready-to-go batches from SQS queue
- Uses connector library to send to external system
- If fails after retries, message goes to **Dead Letter Queue**

**Retry Driver Lambda:**
- Runs 6:00 AM every morning
- Re-drives all messages from dead letter queue back to processing queue
- [Benjamin Nouhaag]: "We keep redriving things this way infinitely. That's a bit of a room for improvement."

[Premanand Thangamani]: "14 days is the limit based on how long a message can stay in the queue to retake. We retry every day."

---

## Consent Outbound: Opt-In/Opt-Out Synchronization

**Consent Flow:** When profile opts out in Apsis One, write opt-out back to external system.

**Path:**
1. Opt-out message → All Sub Queue
2. All Sub Worker validates and passes through
3. **Bypasses Batch Production** (consent typically low volume, sent one-by-one)
4. Goes to Outbound Queue
5. Outbound Worker sends to external system
6. If fails, goes to Dead Letter Queue and retries daily

**Important Caveat:** [Benjamin Nouhaag]: "We do not sync attribute changes back to the CRM like the consent is bidirectional but the attributes only come to Apsis, not from Apsis."

### Handling Dead Letter Queue and Redrive Safety

[Lukasz Grabowski]: "Do you have alerts on the dead letter queue? Do you have situation that there were some not processed messages?"

[Benjamin Nouhaag]: "Yes, we have that all the time. The question is why it is not processed... Each CRM system can have bugs, timeouts, network problems. The customer can also have incorrectly imported consent."

**Safety During Redrive:**
- When redriving messages, **do not redrive consent opt-in messages** (would lose eventual consistency)
- **Only redrive opt-out messages**
- For batches: lose consistency anyway, but external system contracts explicitly state they must handle duplicate batch IDs

[Benjamin Nouhaag]: "We generate an ID for each batch and then they should keep track of whether they have already processed a batch... If things go weird on their side because they're not following it, then it's not something we can do anything about."

### Operational Monitoring

[Benjamin Nouhaag]: "We've had an operational meeting once per Monday where we looked over various customer integration problems and what's in the dead letter queue, what are we always redriving. So that's something I would recommend if you start having problems."

---

## Marketing Automation Flow Integration (MA Flows)

### CRM Sync Node in Flows

Users can embed integration nodes in MA flows to create records in external systems based on flow logic.

**Example Workflow:** [Benjamin Nouhaag demonstrates a flow that:
1. Listens for event "event invited"
2. Waits 2 days
3. Checks if profile has "event confirmed"
4. If not confirmed: creates task in CRM (e.g., "Call customer") with:
   - Type: phone call
   - Urgency: 9 out of 10
   - Assigned to: [specific user or role]

**Technical Flow:**
- When contact reaches integration node, message placed on All Sub Queue
- Goes through exact same batching pipeline as activities
- Outbound Worker sends to CRM system
- Reason for batching: MA flows can trigger for large segments on schedule (potentially 500K people at 12:00 every day)

[Benjamin Nouhaag]: "In MA you are able to trigger based on a segment and a schedule. As far as we know we could be getting 500,000 people into this flow at 12:00 every day, so we need batching."

---

## Multiple Integrations Per Section: The CRM ID Limitation

### Current Constraint

[Michal Rosikiewicz]: "When you have multiple integrations on account and configure email syncing outbound communication, you will have multiple choices or multiple steps. How does it look?"

[Benjamin Nouhaag]: "That's actually per section, and right now you can only have one such integration per section."

### Root Cause: Shared CRM ID Field

[Benjamin Nouhaag]: "The field mappings map to CRM ID. This is very, very, very unfortunate. This is an old Jonas super urgent release something situation... We always map to the CRM ID field and we hate this so much, but we've never had time to fix it."

**Why This Matters:**
- Each integration should have its own key space to avoid mixing profiles
- Bootstrap unique key space for each integration (e.g., FC Enterprise key space, Dynamics key space)
- But all map to shared CRM ID field in Audience
- **Workaround:** Restrict one CRM integration per section

**Scope of Impact:**
- Only affects "native" integrations (those with complex inbound syncing like Dynamics, FC Enterprise)
- Third-party integrations like Playable have no dependency on CRM ID (no inbound profile syncing)
- Anything built after 2020 kept unique key spaces in mind
- Pre-2020 systems still dependent on CRM ID

[Benjamin Nouhaag]: "Anything done after 2020 has kept that in mind. For 2019 and earlier, we are dependent on the CRM ID for a lot of historical integrations and we would need to do a big migration in order to deal with that. It could be a good idea to do during the fall if [you want to do] a joint migration project together."

### Multiple Entity Types in Single Integration

For some integrations, support mapping multiple entities (e.g., contacts and leads):
- Each entity shows as separate table in field mappings
- Bootstrap specific lead ID attribute for that system
- Properly namespaced per entity and system

---

## Form Submission Mapping: Outbound Field Mapping Gap

### Inbound Form Field Mapping
When user creates form:
- Can map form fields to profile attributes (e.g., email field → email attribute)
- Can map form fields to external system fields (desired but limited)

### Current Outbound Behavior (Limitation)

[Benjamin Nouhaag]: "In 2022 this was not possible [to map directly from form to CRM]. Instead... we have another tab called outbound mappings, which is essentially field mappings reversed... Those are not dependent on any specific form. Those are grabbed from the profile instead."

**How It Works Today:**
1. Form submitted, creates profile attributes
2. All Sub Worker receives form submit event
3. Does NOT map form fields directly to CRM
4. Instead: takes **profile attributes** mapped in outbound mappings and sends to external system

**Problem:** [Benjamin Nouhaag]: "That's very unfortunate and the CRM teams have also complained, but that's how it is."

### Feature Flag and Instance-Level Checking

Outbound mappings availability depends on:
1. **Integration spec check** (does this CRM type support it in general?)
2. **Instance check** (does this specific customer's instance support it?)

[Benjamin Nouhaag]: "We have a spec for each integration of what it's allowed to do... But then we are also checking in the external system, does this specific version of the system support it? Because it might be that they have one production version which does not have so much features, but then they have a development instance where they are creating this new feature."

**Example:** On-premise vs. cloud-hosted systems may have different capabilities:
- F.Distributed Edition versions vary significantly
- Some customers on version X, others on version X+10
- Check both Apsis-side spec and customer instance capabilities

---

## Partner and Supplier Strategy: Evolution and Current Model

### Original Problem with Supplier-Driven Integrations

[Benjamin Nouhaag]: "We originally built things per a certain strategy... to standardize use cases for different kinds of systems like CRM and ecom into one place and then work with suppliers that know each system... the strategy was very unwieldy. No matter how good we thought we understood something in the external system, things can always get a little bit weird... We ran into a lot of these crazy little situations very often like one or two days before launch and they required a three-week fix."

**Key Issue:** Whenever new features needed adding, Apsis had to pay suppliers (e.g., CRM Consultana for Dynamics) to update plugins, creating slow feedback loops and inflexible feature development.

### Shift to Partner Model

**New Model:** Partners implement the **Generic Connector** interface themselves on their side.

**Benefits:**
- Partners get revenue directly from customers (no Apsis payment required)
- Partners can innovate beyond the generic interface
- Apsis only adds features at the generic level, which all partners benefit from
- Faster iteration without external dependencies

**Examples:**
- **Sideshop** (Partner): Builds Dynamics 365 CRM connector on generic interface; customers pay Sideshop ~200 EUR/month
- **Super Office** (Partner): CRM integration via Sideshop
- **Intermail** (Partner): Builds custom loyalty integration; also has Relation Plus product on generic connector

[Benjamin Nouhaag]: "The plan right now... is that every single customer running this should be migrated to this. And Sideshop should be getting should be billing for the integration revenue. So anyone wanted to use Dynamics, they'll just get the Sideshop integration."

### Legacy Supplier Relationships (Still Running)

- **Geran Kosnokana** (Supplier): Built Dynamics plugin; Apsis pays hourly; SLA contract no longer active (though customers may believe it is)
- These remain operational but Apsis no longer actively maintains supplier relationships this way

---

## Supplier and Partner Agreements: Contractual Framework

[Benjamin Nouhaag]: "We actually have a standardized contract we can sign. Which I worked with legal back in the days to develop. So this is basically to ensure we get to what we want, but of course it requires a specific project plan... an SLA template... This basically says I want you to support us like this, blah blah blah."

### Key Contractual Documents

**Supplier Delivery Process Document:**
- Before delivery, supplier must walk through build process
- Require 75% test code coverage (build fails without it)
- Detailed instruction on how to install
- List of required user roles
- Overview of files/items created, with exact file paths
- Overview of database tables/entities created or altered
- Definition of done for suppliers

**Supplier Service Level Agreement Template:**
- Support model expectations
- Issue severity definitions (P1/P2 = critical, P3+ = non-critical)
- Response time requirements
- Standardized expectations for operational behavior

[Benjamin Nouhaag]: "Currently we don't have any active suppliers with this agreement, but if you need to enter a collaboration with a supplier in the future, I would advise you to either use this one or at least read through it to find important things that you may be missing, because we've done this a couple times and we've missed things."

**Partner Agreement:**
- For partners implementing generic connector
- Specifies what partners can do
- Termination rights for Apsis
- Partners own everything, including support

### Documentation and Handover

[Benjamin Nouhaag]: "There's a document describing exactly how we work with partners and how we work with suppliers in the future. At the end of it, it will tell you what to do if you want to work with a supplier... All of these files are hosted on an S3 folder with some domain name assigned to it, and that's where actual all these links go. So we always have a latest generator connector API spec that we manually add to this folder."

**Generic Connector Resources:**
- Generic Connector API Spec (latest version in S3)
- Generic Connector Implementation Guide (lengthy, updated with 2024 changes; needs merge)

### Current Active Partners and Suppliers

[Michal Rosikiewicz]: "Do you have inventory for the list of active agreements with partners and suppliers?"

[Benjamin Nouhaag]: "There is, I gave in my handover to Felix before my parental leave. I put together a spreadsheet. But it's very short now... The list of suppliers and partners. We essentially have zero active supplier contracts... and we have two active partners: Intermail and Sideshop."

**Active Partners:**
- **Sideshop**: Dynamics 365 CRM (generic connector implementation)
- **Intermail**: Loyalty platform and Relation Plus (custom + generic connector)

---

## Historical CMS and VMS Integrations (Discontinued)

[Benjamin Nouhaag]: "We did have a couple of CMS integrations back in the day. We had Drupal, FP Server, and Sitecore, which did not function in the way that these work today... They essentially let the user get a drop down of segments when editing content and then say 'this content should only be shown to the segment' and then they send a request to the segment evaluation API... there's a little bit of key space juggling on the CMS side, since nothing should ever be whitelisted for the web key space."

**State:** All rendered open source; documentation and repositories to be provided.

---

## Unified Data: Cross-System Complex Queries

### Business Problem Unified Data Solves

[Benjamin Nouhaag]: "Paul is the head of sales... He wants to reach out to CEOs of Europe-based companies whose employees attended [an expo] last year. Now a problem here, right? We have CEOs of companies, that means we need role and company and we need to know their attendance, but they're usually structured in a very... flat document... not like our profiles."

**Example:** Querying event attendance requires following relationships:
- Event → Attendee → Person → Role → Company → CEO
- Not available in flat profile model

**Current Profile Limitation:**
[Benjamin Nouhaag]: "We have the event... we need to go to to get to an attendee. And then we have a person. And then through the person we can through the role find a company and then we can find the CEO... This is not something we can trivially do with the data that we're importing through Justin today."

### Unified Data Solution Architecture

**Use Case:** Paul wants to send personalized email:
> "Hello Alfonso. It seems like Imashek had a blast at [Event] Poland last year. How do you feel about sending more reps from [Company] this year? We look forward to boosting your business in China."

**Technical Implementation:**
- Audience maintains profile store (with full history of attributes) and metadata
- React provides real-time page views but cannot export efficiently
- Congestion pipeline ingests from React to Athena (warehouse optimized for exports)
- **Data Provider Pattern:** Any external system can provide data to Athena as external table (Apache columnar format)

[Benjamin Nouhaag]: "What we can do for Maxo is that when we create the kind of email sending... Audience can open up an HTTP/2 connection to integrations and fetch paginated data from a specific query in the CRM... get all CEOs and companies who had attendees at the event last year and we can get paginated data."

**HTTP/2 Streaming (Not Traditional Request/Response):**

[Speaker 1/Michal]: "Why is mentioning that this is HTTP/2 connection important?"

[Benjamin Nouhaag]: "You can stream over HTTP/2. So basically audience triggers an export internally and is told to join that with the results of the query in an external system. When that happens, they open this connection through which we can stream a response... we might get 2,000,000 entries back from the CRM system and you do not want to get all of that in one request."

**Streaming Process:**
1. Audience triggers export and initiates HTTP/2 stream
2. Integrations begins fetching paginated data from CRM system
3. Each page translated to Apache columnar format and streamed back
4. Audience joins streamed CRM data with its own exports
5. Continue until CRM provides all data

### Unified Data Status and Limitations

[Speaker 1]: "Is this feature like production tested? What's the state of this feature?"

[Benjamin Nouhaag]: "We had the product suite project in the fall or winter of 2024 and we finished everything on time. But of course no customer has used it because Maxo didn't bring in customers... One customer has used it, but I would not say it's really tried in actual usage... One customer has said it, but it hasn't been used by 100 customers and proven to be stable."

**Why Unified Data Matters for Integration Team:**
[Benjamin Nouhaag]: "I'm not trying to give you a complete understanding of how this works. I want to make sure that you know this exists because you will have so many upset customers wanting this kind of data, and here's the first step taken in that direction."

### Historical Approach to Complex Data (Deprecated)

**Previous attempts:** Flatten external system data and send as events to Audience. Rejected as too expensive because:
- Audience keeps full history of every attribute on every profile
- Replicating role/title/company data on every profile increases Athena bill significantly
- Unified Data approach is superior because it joins at query time, not storage time

---

## Key Takeaways

1. **Architecture Philosophy:** Justin provides generic infrastructure; connector libraries provide system-specific translation. Modularity enables scaling across many systems while maintaining consistency.

2. **Inbound Flow:** Contacts flow from external systems through webhooks (delta sync) or batch import (full sync) → Integration Manager → Mappings Manager → Audience. Eventual consistency is maintained through clever buffering during full syncs.

3. **Outbound Flow:** Activities (emails, SMS, forms, MA flows) trigger messages placed on All Sub Queue → processed by All Sub Worker → batched for efficiency by Batch Production Worker → sent by Outbound Worker. Failed messages go to Dead Letter Queue and retry daily.

4. **Consent is Bidirectional:** Inbound consent mapping works like field mapping. Outbound consent changes are written back to CRM. Attribute changes are **not** written back (intentional design—CRM is data master).

5. **Profile Lists:** Both dynamic and static lists from external systems are abstracted and used to tag existing profiles (don't create new ones). Can be run on-demand or on recurring schedule.

6. **One CRM Per Section (Unfortunate Limitation):** Caused by shared CRM ID field architecture from early design. Workaround until migration. Does not affect third-party integrations without inbound syncing.

7. **Partner > Supplier:** Apsis is shifting from paying suppliers to update integrations toward partners implementing the Generic Connector interface themselves. Partners monetize directly; Apsis invests in generic-level features.

8. **Contractual Framework:** Supplier and partner agreements exist as templates. Include delivery process (testing, documentation, file paths, database schema), SLA templates, and partner agreement terms.

9. **Unified Data:** A query-time joining capability (over HTTP/2 streaming) allows complex cross-system data (e.g., "CEOs of companies whose employees attended event X") without flattening data into profiles. Production-ready but not yet heavily used.

10. **Operational Monitoring:** Regular operational reviews (e.g., Monday meetings) of dead letter queue and customer issues are essential for long-term health. Redrives have idempotency requirements on external system side.

---

## Unresolved Questions and Action Items

### Action Items (from Benjamin)

1. **Documentation Merge:** Benjamin has submitted a PR (since 2022, feedback received spring 2024) with supplier/partner process documentation. Needs review and merge. Requires GitHub reviewer to click merge.
   - **Assigned to:** Speaker 1 (to coordinate with Felix for feedback)
   - **Deliverable:** Updated documentation in shared folder/GitHub

2. **Upload Documentation to GitHub:** Move supplier/partner agreements, delivery process, and SLA templates from personal S3 folder to shared repository.
   - **Assigned to:** Benjamin to prepare; Speaker 1 to move to shared folder

3. **Generic Connector API Spec Update:** 2024 updates need to be merged into main implementation guide (currently separate; not yet in readme).
   - **Target:** Complete by next session
   - **Assigned to:** Benjamin

4. **Supplier/Partner Inventory:** Maintain active list of partners and suppliers with contact information.
   - **Status:** Spreadsheet exists in Benjamin's handover to Felix
   - **Recommendation:** Formalize and keep updated

### Open Questions / Points for Technical Deep Dive

1. **CRM ID Field Migration:** How to approach migrating pre-2020 systems away from shared CRM ID field? Benjamin suggests potential fall project.

2. **Form Outbound Mapping:** Current design maps form submissions to outbound profile attributes, not direct form→CRM mappings. Confirmed as limitation but no committed fix timeline.

3. **Infinite Redrive of Dead Letter Messages:** Currently no limit on how many times a message is redriven. Flagged as "room for improvement."

4. **Kafka Partitioning and Scaling:** All event batching uses single Kafka partition per integration. Potential bottleneck not yet explored.

5. **Unified Data Production Readiness:** With only one real customer using it, more real-world testing needed before confidently recommending for new use cases.

### Next Steps

- [Benjamin]: Provide comprehensive documentation list to team
- [Benjamin]: Coordinate with technical team (Eric) for detailed architecture walkthroughs
- [Team]: Review supplier/partner strategy and decide on operational approach going forward
- [Team]: Consider CRM ID migration project scope if multiple integrations per section becomes critical requirement

---

## Technical References and Documentation (To Be Provided)

- Presentation slides (inbound and outbound diagrams)
- Generic Connector API Spec (latest)
- Generic Connector Implementation Guide (2024 updates pending merge)
- Supplier Delivery Process Document
- Supplier SLA Template
- Partner Agreement Template
- Process & Policies Document (PR pending merge)
- Unified Data Architecture and Implementation Documentation
- List of Historical Integrations (Drupal, FP Server, Sitecore—repositories available)
- Supplier/Partner Inventory Spreadsheet