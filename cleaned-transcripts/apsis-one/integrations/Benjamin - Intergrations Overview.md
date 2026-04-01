---
source_file: Benjamin - Intergrations Overview.txt
domain: Apsis One Integrations
topics: [integrations architecture overview, Justin middleware, inbound sync, outbound sync, field mappings, subscription mappings, full sync, delta sync, profile lists, sync conditions, outbound event batching, dead letter queue retry, unified data / data provider, supplier vs partner strategy, generic connector, CRM key space issue, MA integration nodes]
speakers: [Benjamin Nouhaag (outgoing integrations lead), Lukasz Grabowski, Michal Rosikiewicz, Premanand Thangamani (Prem), Speaker 1 (unidentified, appears to be a team lead)]
key_components: [Justin (integration middleware), Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager, Full Sync Producer, Full Sync Consumer, All-Sub Queue (Audience Subscription Queue), All-Sub Worker, Batch Production Worker, Outbound Worker, Dead Letter Queue, Retry Driver Lambda, Profile List Sync Lambda, Generic Connector, Unified Data / Data Provider, Athena, React (profile store), SQS FIFO, Kafka, ECS tasks, CloudWatch Events]
session_type: knowledge-transfer
---

# Apsis One Integrations – Conceptual Overview (KT Session)

## Session Overview

Benjamin Nouhaag, the outgoing integrations lead, walks through the full conceptual architecture of the Apsis One (formerly Profile Cloud) integrations domain for incoming team members Lukasz, Michal, Prem, and one other colleague. The session covers the original mission and design philosophy of the integrations team, the middleware system called **Justin**, and every major component of both inbound and outbound sync pipelines as seen through a live demo environment. It also covers the strategic shift from a supplier model to a partner/generic-connector model, the current legacy technical debt around CRM ID fields, and a preview of the **Unified Data** feature built for the Maxo product suite.

---

## Origin and Mission: Why the Integrations Layer Exists

The problem Apsis set out to solve (originally as "Profile Cloud," around 2013–2014): companies had many disparate systems that IT departments were manually integrating, resulting in data always being out of sync and unavailable when needed.

The solution was to place something in the middle — Profile Cloud, which became Audience in Apsis One. The integrations team's mission is to ensure that the arrows (data flows) between external systems and Audience exist for as many systems as possible, and to provide the same SoA (Service-Level Agreements) as Apsis provides for the rest of Apsis One — including uptime guarantees and 24/7 PagerDuty coverage.

> "This can be very challenging for integrations since some logic will always reside outside of Apsis."

### Original Design Strategy

The original strategy had two pillars:
1. **Standardize use cases** for different kinds of systems (CRM, ecommerce) into one place.
2. **Work with suppliers** who know each external system deeply, while keeping the scope of logic inside external systems minimal so that Apsis could monitor and guarantee uptime for as much of the chain as possible.

**Example of a standardized use case:** An abandoned cart infrastructure that ran once per hour, called standardized functionality to fetch carts from all connected systems, and threw abandoned cart events into Audience for MA to react to. Each external system only needed a small connector piece.

### Problems with the Original Strategy

When building connectors without supplier involvement (e.g., Lime CRM, Shopify), edge cases in the external system were discovered very late — often one or two days before launch — and required multi-week fixes.

**Example gotcha with Lime CRM:** You can opt out of marketing contact, but you can also *remove the relationship* between contact and marketing — which should functionally be an opt-out but was not covered by webhooks.

---

## Architecture Overview: Justin (The Integration Middleware)

**Justin** is the integration middleware layer. The name is intentional — its job is to keep everything **In Sync** (named after Justin Timberlake, as an internal joke).

Design principles:
- **Generic infrastructure** with small **connector libraries** for easy communication with each specific external system.
- The connector library for a given system only needs to handle things like: making an HTTP request to fetch all fields from that system and translating them into the format Justin expects.
- Modularity is the core design principle and remains so today.

---

## Shift to the Generic Connector Model

The current preferred way of working, moving away from the supplier/custom-connector model:

- Apsis provides an **API spec (contract)** to each external system wanting to integrate.
- The external system implements that contract themselves.
- Apsis only needs to create a **config file** for each new integration.
- This removes the need for Apsis to deeply understand each external system.

> "We just gave an API spec to each external system that wanted to integrate and we said here's a contract that must be fulfilled. You guys just implement it and all we need to do is a config file for you."

**Legacy systems still running on the old infrastructure:** Microsoft Dynamics 365 (original connector), Lime CRM, and others.

---

## Inbound Sync Architecture

### Integration Manager

The first Justin service encountered in a typical integration flow. Previously a Lambda, now an **ever-running ECS task**.

Responsibilities:
- When field mappings are rendered in the UI, Integration Manager fetches the schema from the external system using the appropriate connector library.
- Routing logic: if the URL indicates Dynamics, it uses the Microsoft Dynamics connector library; if FSC Enterprise, the FSC Enterprise connector library; if generic connector, there is an internal mapping.
- Returns schema data directly to the front end caller.

### Mappings Manager

Stores field mappings. Has its own database user with access to the mappings table.

Responsibilities:
- Ensures mappings don't create loops ("Justin's Laws" — Eric will cover in more detail).
- Stores both field mappings and subscription (consent) mappings.
- Caches mappings; cache is invalidated on update.

### Webhooks and Delta Sync

On installation, Apsis subscribes to webhooks for all relevant events in the external system. As mappings change, those webhooks are updated.

Two webhook paths exist:
1. Some systems (very few) send to an official endpoint exposed by the Apsis Delta Sync Manager via the One API.
2. Most systems send to a **custom webhook endpoint** that Apsis provides directly (without the platform team being in the loop).

**Delta Sync processing pipeline:**
1. Incoming webhook → **SQS FIFO queue** (message group = CRM ID).
2. Consumed by the **Delta Sync Worker** (ever-running ECS task), 1 or 10 messages at a time.
3. Delta Sync Worker asks Mappings Manager for which mappings exist on this account (cached).
4. Sends a message to Audience.

### Full Sync

Used after initial setup to sync everything per current mappings.

**Full Sync pipeline:**
1. User clicks "Start New Sync" → POST to the **Full Sync Manager** (ever-running ECS task, previously Lambda).
2. Full Sync Manager spins up a **dedicated stack on demand** for this specific sync job (acknowledged as overkill, but done to avoid queuing issues with FIFO SQS).
3. **Full Sync Producer** uses a connector library to fetch paginated contacts from the external system.
4. Data is placed on the **Full Sync Queue** for this specific operation.
5. **Full Sync Consumer** (same application as Delta Sync processing, different queue) consumes the queue, asks Mappings Manager for field and consent mappings, sends to Audience.

**Full sync status states:**
- **Finalizing:** All data has been written to Audience, but waiting for Audience ingestion delay to catch up to the point-in-time when the last message was written.
- **Successful:** Ingestion delay has caught up.

Full Sync Manager knows the sync is complete when:
- No producing threads see anything in their paginated requests (producer done).
- No consuming threads see anything in the queue (consumer done).
- Then it waits for ingestion delay before switching to "successful."

### Buffering During Full Sync (Delta Sync Pause)

**Problem:** During a full sync, real-time delta sync messages arriving for the same integration could break eventual consistency.

**Solution:** A buffer queue. When a delta sync message arrives and a full sync is ongoing for that integration, the message's **visibility timeout (backoff time) is set** to delay it in the queue. All messages for that integration are buffered and processed in order after the full sync completes.

> "This is a solution we often wish we didn't have, but we didn't have time to address it properly."

The underlying reason it exists: Apsis could not simply drop or reject real-time messages saying "a full sync is ongoing," so buffering was the pragmatic solution.

### Subscription (Consent) Mappings

Works the same way as field mappings but for consent/subscription data.

- Integration Manager fetches what consent resources exist in the external system.
- Audience is queried for what topics exist.
- A mapping is saved in the Mappings Manager.
- During full sync and delta sync processing, this mapping is used to determine how to handle consent.

In legacy custom connectors:
- A boolean field on a contact card → **virtual consent**
- A dedicated consent resource → **native consent**

In the generic connector, this distinction is not enforced — Apsis just processes what the external system provides.

### Profile Lists (Queries and Static Lists)

**Terminology note:** Naming is system-dependent and unfortunately confusing.
- In Microsoft Dynamics: a list = "Marketing List" (dynamic or static).
- In FSC Enterprise: dynamic list = "Query," static list = "Profile" (has nothing to do with Apsis profiles — acknowledged as a terminology problem never addressed).

The abstraction Apsis uses: **dynamic profile list** and **static profile list**. Every integrated system encountered has had this concept.

**How profile list sync works:**
- Importing a list does NOT create profiles — that is done by full sync or real-time delta sync.
- Importing a list **applies a tag** to every profile already in Audience who is on that list in the CRM.
- If the list is re-synced and a contact is no longer on it, the tag is removed. Apsis tracks who received the tag for each operation.

**Profile List Sync pipeline:**
1. User clicks "Start New Sync" for a profile list → posted to Full Sync Manager → placed on the **Profile List Sync Queue**.
2. At 6:00 AM every morning, a **CloudWatch Event triggers a Profile List Sync Lambda** which takes all recurring profile list syncs and also places them on the Profile List Sync Queue.
3. A consumer takes each entry and **spins up one worker per job**, one at a time per integration (to avoid DDoSing customer CRM infrastructure — most do not have scalable systems).
4. Worker fetches every contact on the list, holds in memory (no queue), and sends a tag request to Audience.

**Historical issue:** Before edge cases were addressed, this pipeline failed frequently with no queue to catch failures. Now considered stable.

### Sync Conditions

A filter applied at the integration level. Example: only sync contacts where `active = true`.

- Condition mapping saved in Mappings Manager.
- Evaluated during every full sync and delta sync message processing.
- **Additional behavior:** If an inbound message is received and the condition is no longer met (e.g., `active` is now `false`), Apsis will **delete the profile**. This is a feature-flag-activated behavior.

---

## Outbound Sync Architecture

Outbound syncing sends activity data (emails, SMS, forms, MA flows, events) from Apsis to the external CRM.

**Supported activity types for outbound sync:** Email, SMS, Form, MA flows, Event tool activities. Surveys are not supported.

For each activity: when the activity is created, a corresponding record is created in the CRM. Then as events occur (sent, delivered, opened, clicked), they are transferred in batches to the external system.

### Outbound Manager

A service acting as a bridge between the front end and the CRM system. Tells the CRM to create the corresponding record when an activity is synced.

The front end first asks the integrations backend whether an integration capable of syncing that activity type is installed; if yes, a UI toggle is shown to the user.

### All-Sub Queue and All-Sub Worker

All events from Audience (for all outbound purposes) flow through:

1. **All-Sub Queue (Audience Subscription Queue):** SQS FIFO queue.
2. **All-Sub Worker (Audience Subscription Worker):** Ever-running ECS task.
   - Asks Integration Manager whether an integration is installed for this event.
   - Verifies whether the specific activity should be synced (via Outbound Manager).
   - Places messages onto a **Kafka partition**.

### Batch Production Worker

Ever-running ECS task. Continuously polls the Kafka partition.

Batching logic:
- Groups messages by destination integration.
- Keeps polling until either:
  - The largest batch has surpassed **200 KB**, OR
  - It has been polling for more than **1 second**.
- Finished batches are placed on an **SQS FIFO queue** (256 KB limit; 200 KB threshold leaves comfortable headroom).
- For 1 million events, this could result in 1,000+ batches — not yet a problem in practice but noted as an area for future improvement (e.g., choosing a different queue technology).

### Outbound Worker

Ever-running ECS task. Takes batches from the outbound SQS FIFO queue, uses a connector library, and sends the batch to the external system.

**Batch ID generation:** A unique ID is generated for each batch. External systems are expected (per contract) to track whether they have already processed a given batch ID to handle redelivery idempotently. They are free to ignore this, but if things go wrong on their side due to non-compliance, that is not Apsis's responsibility.

### Dead Letter Queue and Retry

If the outbound worker fails after retries, messages go to a **Dead Letter Queue (DLQ)**.

A **Retry Driver Lambda** runs at **6:00 AM every morning** and redrives everything in the DLQ back to the outbound queue.

**Retry limit:** Currently **infinite** (no limit on how many times a message is redriven). Acknowledged as a room for improvement.

> [Premanand Thangamani]: The SQS message retention limit is 14 days. The retry driver re-drives daily, so messages can persist up to 14 days in the queue.

**Important safety rule on redriving consent messages:**
- On a redrive, **eventual consistency is no longer guaranteed**.
- Therefore: only **opt-out** (opt-in = false) consent messages are redriven.
- Opt-in (opt-in = true) consent messages are **not redriven** to avoid incorrectly re-opting someone in.
- For batches, consistency is also lost on redrive, but the contract with external systems explicitly states they must handle this.

**Common causes of DLQ accumulation:**
- CRM system bugs, timeouts, or network issues.
- Customer has incorrectly imported consent in Apsis that cannot be processed.

**Recommended operational practice:** Run a weekly operational meeting (e.g., Monday) to review customer integration problems and audit what is repeatedly being redriven. Benjamin's team did this during a period of high operational load and found it valuable. It also created a feedback loop for escalating unresolved issues to external system vendors.

### Consent as Bidirectional Sync

> "Consent is bidirectional but attributes are not."

- Attribute changes flow **only inbound** (CRM → Apsis). Apsis does not write attribute changes back to CRM.
- Consent/subscription changes are **bidirectional**: CRM → Apsis (via webhooks/full sync) AND Apsis → CRM (via the outbound pipeline).
- Historical reason: customers have consistently preferred that the CRM remains the data master for attributes.

Consent changes from Audience bypass the batch production worker (that component did not exist when consent outbound was implemented, and consent messages arrive one at a time rather than in high volume). They go: All-Sub Queue → All-Sub Worker → directly onto Outbound Queue → Outbound Worker → external system → DLQ carousel if failure.

### Marketing Automation Integration Nodes

Within an MA flow, users can place a **CRM-specific node** to create records (e.g., tasks) in the external system.

**Example use case demonstrated:**
- Listen for "contact invited to event."
- Wait 2 days.
- Check: has the contact confirmed attendance?
- If yes: do nothing.
- If no: create a task in the CRM (e.g., FSC Enterprise) for a salesperson to call the contact and get them to confirm.

Node config values (urgency, type, assignee, etc.) are driven by the external system's schema for whatever entity type is being created.

**Processing:** Whenever a contact reaches this MA node, a message is placed on the All-Sub Queue. It then goes through the full batch production pipeline (grouping, batching, Outbound Queue, Outbound Worker). This batching is necessary because MA flows can be triggered by a segment + schedule, potentially placing **500,000 contacts** into the flow simultaneously.

### Outbound Field Mappings (Form Submit Special Case)

For form submissions, a **reverse field mapping** exists (Apsis attributes → CRM fields), called **outbound mappings**.

**Important caveat:** These mappings are NOT per-form. They are pulled from the **profile** at the time of the form submit event. So when an All-Sub Worker sees a form submit event from Audience, it does not use the field values submitted in the form — it fetches the mapped attributes from the profile and uses those as the outbound payload.

> "That is very unfortunate and the CRM teams have also complained, but that's how it is."

The desired behavior (per-form field mapping directly to CRM) was not implemented due to what Benjamin described as "political reasons" in 2022.

### Feature Flag Gating for Outbound Features

Whether a feature (e.g., outbound mappings, profile lists) appears in the UI is controlled by a **two-layer check**:

1. **Apsis-side config:** Does this CRM system/connector type support this feature at all? (Configured in Justin's connector specification.)
2. **Instance-side check:** Does this specific deployed instance/version of the external system support it?

This two-layer check is particularly important for on-premise systems like **eDeal**, where different customers run different versions of the software with different versions of the Apsis connector installed. Customer A may support a feature that Customer B does not.

---

## The CRM ID Field Problem (Critical Technical Debt)

**This is a known, significant piece of technical debt.**

The field mappings UI shows a field called "CRM ID" and every integration maps to a field called `CRM ID` (called "CLE" in the French-language demo). This is a **single shared key space** — not unique per integration.

**Why this is a problem:** If two CRM integrations are installed on the same section, their CRM IDs could collide, causing profiles to be mixed up.

**Why it happened:** An early Jonas-era "super urgent release" established the CRM ID field and key space. It was spec'd for use by integrations but was never properly namespaced.

**Actual workaround in place:** Apsis bootstraps a **unique key space per integration** (e.g., FSC Enterprise key space, Dynamics key space, and even one per section). This prevents accidental mixing of profiles in practice.

**But:** Because the UI always maps to the single "CRM ID" field label, and because of this legacy constraint, Apsis enforces a **limit of one CRM-type integration per section**. The integrations page shows "only one at a time" for CRM-type integrations (Dynamics, Super Office, FSC, etc.).

> "We always map to the ******* CRM ID field and we hate this. We hate this so much, but we've never had time to fix it."

**Things done correctly post-2020:** Any integration work done after 2020 properly bootstraps entity-specific key spaces (e.g., a dedicated Lead ID attribute for a leads entity, separate from Contact ID). The legacy problem applies to pre-2020 integrations.

**Potential remediation:** A migration project during the fall (timing relative to session date, October 2025) to properly fix this — suggested as a potential joint project between the incoming team and Eric/Prem.

**Third-party integrations (e.g., Playable) are not affected** by this limitation because they don't use the CRM ID field at all. They work by: Apsis bootstraps a key space and attributes per a specification file the partner provides, generates an API key, and the user copy-pastes credentials into the third-party tool. No Apsis-side dependency on CRM ID.

---

## Multiple Entity Support in Integrations

For some integrations, Apsis can create and manage profiles based on **multiple entities or tables** in the external system (e.g., both Contacts and Leads in Dynamics).

Each entity type appears as its own mapping table in the UI.

> "Anything done after 2020 has kept that in mind" — meaning post-2020 work properly bootstraps entity-specific ID attributes rather than relying on the single CRM ID field.

---

## Supplier vs. Partner Strategy

### Supplier Model (Legacy/Undesired)

**Example:** Microsoft Dynamics 365 (original connector).
- Apsis contracted **CRM Consultana** (a Stockholm-based CRM contractor) to build a plugin with webhooks that Apsis could subscribe to.
- Apsis paid by the hour. SLAs existed (and may still be billed to customers even though Apsis no longer pays for them — ⚠️ this situation was flagged as unresolved).
- Whenever Apsis wanted to add a feature, it had to go to CRM Consultana, pay them, brainstorm, and go through a slow process. This was not scalable across multiple integrations.

### Partner Model (Current Direction)

**Example:** Dynamics 365 CRM by **Sideshop**.
- Sideshop is a **partner** — Apsis does not pay them; they bill customers directly (target: ~200 EUR/month for the integration).
- Sideshop implements the **generic connector interface** on their side and gets all Apsis integration features out of the box.
- Sideshop owns their own build, support, and feature development outside the generic connector scope.
- Sideshop can bring feature requests to Apsis, and Apsis only needs to implement at the generic level.

**Other current active partners:**
- **Sideshop** — Dynamics 365 CRM integration + Super Office CRM integration.
- **Intermail** — Two integrations:
  - **Intermail Loyalty** — custom-built (not on the generic connector).
  - **Relation Plus** — running on the generic connector (appears as "coming soon" in the demo UI).

**Current state of supplier contracts:** Zero active supplier contracts (Apsis does not want to pay for them). Two active partners: Intermail and Sideshop.

**Commercial misalignment noted:** The plan to migrate all Dynamics customers from the old connector to the Sideshop integration is not being followed by the commercial organization.

### Supplier Agreement Documentation

A standardized contract and process exists (developed with Apsis legal) for future supplier engagements:

- **Standardized contract template.**
- **SLA template** with:
  - Issue severity definitions aligned to Apsis's internal P1/P2 (critical) and P3+ (non-critical) classification.
  - Required response times.
  - Support obligations.
- **Supplier delivery process document** (must be signed), requiring:
  - Walk-through of the build process.
  - **75% test code coverage** (enforced — must prevent build from passing if not met).
  - Detailed installation instructions.
  - List of required user roles.
  - Overview of all files, items, and database tables created/altered, with **exact file paths**.
  - (Effective definition of done for supplier deliverables.)
- **Partner agreement** for generic connector partners: covers rights, termination clauses, support ownership, and links to:
  - Generic connector API spec.
  - Generic connector implementation guide (last updated 2022; a 2024 update exists but is not yet merged — action item below).

All these documents are hosted in an **S3 folder with a domain name assigned** and linked from a GitHub repository (process and policies repo). Files are currently under Benjamin's personal GitHub, pending handover.

### Legacy CMS Integrations (Historical Context)

Three legacy CMS integrations existed: **Drupal**, **FP Server**, and **Sitecore**.

These worked differently from CRM integrations: they allowed users to select a segment from a dropdown when editing content, and the CMS would call Apsis's **segment evaluation API** to determine whether to show that content to a given visitor. There was key space handling on the CMS side to avoid whitelisting in the web key space. All three have been rendered **open source** — repositories will be shared.

---

## Unified Data / Data Provider Feature

### Problem This Solves

Standard Apsis integrations import data as **flat profile documents**. Many CRM systems have **relational data** (multiple tables with relationships) that cannot be trivially flattened.

**Illustrative use case:** An event company wants to email CEOs of companies whose employees attended last year's trade show. This requires:
- Event → Attendee → Person → Role → Company → CEO

This cannot be constructed from flat profile data alone. The CRM has the advantage of structured relational tables for exactly this kind of query.

Additionally, Audience's profile store (React/Athena architecture):
- React (profile store): single-key lookup only; exporting everyone matching a segment would take approximately **one day**.
- Athena: used for bulk exports via a congestion pipeline from React.
- Problem: if flat attribute replication (e.g., replicating job title at a company per profile) were used to solve this, it would **significantly increase the Athena bill** due to full attribute history being kept per profile. This approach was deemed too expensive.

### How Unified Data Works

- An external system provides data to **Athena as an external table** in **Apache columnar format** (Apache Parquet / columnar). This is called a **Data Provider**.
- When Audience triggers an export (e.g., for an email send), it can be told to join its own data with results of a **pre-made query in the CRM**.
- Audience opens an **HTTP/2 streaming connection** to Integrations (HTTP/2 is used specifically because it supports streaming — not a standard request-response).
- Integrations fetches data from the CRM **paginated**, converts each page to Apache columnar format, and **streams it back** through the open connection.
- Audience/Athena joins the streamed CRM data with its own export.
- This avoids having to retrieve millions of rows in a single request or make multiple independent requests.

**Example:** Get all CEOs of companies whose staff attended Event X last year — returned from a pre-made CRM query — then join with Audience profile data to personalize email content with both the CEO's name and the name of their employee who attended.

### Integration-Agnostic Design

As with all Justin components, there is nothing system-specific in the implementation except in the connector library. The feature can be reused for any CRM.

### Current State

Built for the **Maxo product suite** project (completed fall/winter 2024). One Berto customer has used it. Not widely tested in production — not proven stable at scale with many customers.

> "I would not say that it is really tried in actual usage. One customer has sent with it, but it hasn't been used by 100 customers and proven to be stable."

Comprehensive documentation exists describing:
- The full user experience flow.
- All UX paths through the current implementation.
- Technical contract between Justin, the email tool, Audience, and the CRM.
- Every API call and example payload for each step of the user journey.

---

## Key Takeaways

1. **Justin is the integration middleware.** It is made up of multiple always-running ECS tasks (Integration Manager, Mappings Manager, Delta Sync Worker, Full Sync Manager/Producer/Consumer, All-Sub Worker, Batch Production Worker, Outbound Worker) plus Lambdas (Retry Driver, Profile List Sync) and SQS/Kafka queues connecting them.

2. **Inbound and outbound are asymmetric by design.** Attributes flow only CRM → Apsis. Consent is bidirectional. Events/activity data flow only Apsis → CRM.

3. **The CRM ID field is the most significant piece of technical debt.** It prevents multiple CRM integrations per section and is the direct cause of several architectural limitations. A migration project is desirable.

4. **The generic connector model is the strategic direction.** All new integrations should use it. Partners own their implementation and support. Apsis only maintains the generic-level contract.

5. **Eventual consistency during full sync is maintained via buffering**, not by blocking or dropping delta sync messages — a pragmatic solution with acknowledged downsides.

6. **The retry loop for the DLQ is currently infinite.** Only opt-out consent messages are redriven (opt-in messages are not, to avoid incorrect re-opt-ins).

7. **Unified Data is a built but lightly-used capability** that solves the flat-data-model limitation for complex relational CRM queries. It should be known to exist when customers request this kind of cross-entity personalization.

8. **Feature availability per integration is two-layer checked:** Apsis-side connector spec AND instance-side capability check. Particularly important for on-premise systems with varying versions.

9. **Currently zero active supplier contracts; two active partners** (Sideshop for Dynamics/Super Office, Intermail for Loyalty/Relation Plus). Standard contract and SLA templates exist for future supplier engagements.

10. **Operational recommendation:** During periods of high integration load, run a weekly (e.g., Monday) operational meeting to review DLQ contents and customer integration issues systematically.

---

## Unresolved Questions and Action Items

| Item | Owner | Status |
|---|---|---|
| Merge the process/policies PR (submitted 2022, feedback received spring 2024, not yet merged) | Michal / Speaker 1 to review; Benjamin to send PR link | Pending — Benjamin to send PR after meeting |
| Update the generic connector implementation guide with 2024 changes and merge into the readme | Benjamin | Target: by next day |
| Upload supplier/partner/process DOCX files to GitHub repo | Benjamin | Pending |
| Upload all documentation to the shared OneDrive folder (team's standard location) | Benjamin to provide docs; Speaker 1 to move to folder | Pending |
| Provide list of suppliers and partners (active and historical, including contacts) | Benjamin | Can provide from memory if spreadsheet not found; gave to Felix before parental leave |
| Share CMS integration (Drupal, FP Server, Sitecore) open-source repositories | Benjamin | Pending |
| Share unified data documentation links | Benjamin | Pending |
| Investigate whether SLA with CRM Consultana is still being billed to customers despite Apsis no longer paying | Not assigned | ⚠️ Unresolved — flagged as potentially problematic |
| Consider migration project to fix CRM ID field / key space issue | Incoming team (Eric, Prem, Lukasz, Michal) | No timeline set — suggested for fall |
| Retry limit: add a cap on how many times DLQ messages are redriven (currently infinite) | Not assigned | ⚠️ Known gap, no owner or timeline |
| Evaluate moving batching infrastructure to a different queue technology (SQS 256 KB limit is a constraint) | Not assigned | Noted as future improvement |