---
source_file: Integration projects never finished due to FE resources.txt
domain: Apsis One Integrations
topics: [full sync report improvements, extended sync conditions, large customer scalability, Tribe pagination bug, Tribe lead entity cleanup, internal broker service debugging tool]
speakers: ["Erik Andersson (outgoing developer/domain expert)", "Lukasz Grabowski (team lead)", "Tomasz Kowalski (developer)", "Michal Rosikiewicz (developer)"]
key_components: [Apsis One, generic connector, Tribe CRM, SQS, audience, sync conditions, profile sync, consent sync, internal broker service, full sync, profile list sync]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Duplicate profiles"]
---

## Session Overview

This is a knowledge transfer session led by Erik Andersson, covering several integration features that were built in the back end but never completed due to front-end resource constraints. The session covers: (1) an improved full sync report with separated profile/consent statistics; (2) extended sync conditions supporting AND/OR logic; (3) a critical scalability issue for large customers (8M+ contacts); (4) a Tribe pagination bug producing duplicate contacts; and (5) a Tribe lead entity cleanup that is blocked pending confirmation from the Tribe team. Erik also demonstrates the internal broker service as a key debugging tool. Prioritization is discussed between Erik and Lukasz, with the large-customer scalability issue identified as the most urgent.

---

## Improved Full Sync Report (Back End Done, Front End Pending)

### Problem: Conflated Statistics in Sync Report

The current full sync report groups **profile update messages** and **consent messages** together under a single "successful" count. Customers see, for example, 39,000 "successful" records and are confused because they only have 8,000–10,000 contacts. The 39,000 figure actually represents profile updates *plus* consent messages combined.

This also confuses NPS and account managers.

### What Was Built

A change was made to the **data model** to separate these statistics. The sync report now tracks:

- Number of profile messages (created, successful, failed, skipped)
- Number of consent messages (created, successful, failed, skipped)

Backwards compatibility is maintained: the old `total` and `successful` fields are still present.

### Current Status

> "The work is done in back end. I would highly suggest that you look at this because this should be a very low hanging fruit if you have someone that is swift at front end work and it will add a lot of value for the customer. If nothing else, it will reduce the support questions you get for the full syncs."

The front-end change was never made because Josh left the company and integration work was consistently deprioritized in favor of email and audience modules.

**Caveat:** Needs more thorough testing since it hasn't been fully validated — only confirmed working during the back-end release.

### Priority Assessment

[Lukasz]: Low priority, low effort. Can be added as a backlog item if front-end bandwidth exists. Not a Q1 item. The report has looked this way since release with no critical impact.

---

## Extended Sync Conditions with AND/OR Support (Back End Done, Front End Pending)

### Current Weaknesses

The current sync conditions UI has two major limitations:

1. **Implicit AND only** — all conditions are AND-ed together with no way to use OR logic.
2. **Only `equals` operator** — no support for `not empty`, `contains`, `starts with`, `ends with`, etc.

### What Was Built

The back end was extended to support the **segment builder format** for sync conditions, adding:

- AND/OR logical grouping
- Operators matching the segment builder: equals, not equals, contains, starts with, ends with, etc.

Prem tested this back-end work. The front end was not updated because it requires significant UI changes.

### Generic Connector API: Two Endpoint Versions

When sync conditions are changed, they are sent to the CRM system via an endpoint in the **generic connector**. There are now two versions:

**V1 (current structure — AND only):**
- Single `section` discriminator
- Sends entity name and flat list of conditions (e.g., `first_name = Elsa AND sync_to_apsis = true`)
- Used by all current live CRM integrations

**V2 (new structure — AND/OR support):**
- Significantly more complex payload structure
- Per-entity condition sets
- Currently only **Siteshop** has this endpoint defined on their side

### Important Caveat Before Enabling V2

> "Before this is actually enabled for any account — the V2 version of the sync conditions — this needs to be verified with Siteshop because I don't think they have actually implemented support for the new structure, because it never took off inside of Apsis."

Additionally, Erik and Walker have had discussions with **FSC Enterprise** about needing sync condition support for their large-customer deployment (see next section). Enterprise will also need to be involved.

### Priority Assessment

[Lukasz]: Nice-to-have, lower priority given current team situation. Not Q1. The feature has been absent since release with no critical issue.

[Erik]: Agrees — no urgency to rush this. Finish other items on the list first, as it has consequences for other CRM systems.

---

## Critical Scalability Issue: Large Customers with 8M+ Contacts

### The Core Problem

Sync conditions are currently **evaluated inside Apsis**, which means the full sync must download *all* contacts from the CRM system before filtering. For a customer with 8 million contacts where only 100,000 are wanted, Apsis still downloads all 8 million — a severe data and time waste.

### Two Approaches for CRM-Side Condition Evaluation

**Approach 1 (Siteshop model):** Apsis stores conditions as configuration and sends them to the CRM. The CRM evaluates conditions and generates paginated results accordingly.

**Approach 2 (FSC Enterprise model):** The CRM has a stateless API. Apsis includes sync conditions in the request body/query parameters when fetching contacts during full syncs and consent fetches.

Most of the work is on the CRM side, but Apsis also needs to implement the feature of attaching sync conditions to outbound requests.

> "There is some content brewing behind the scenes that we are not able to handle these type of customers, but we have historically said like we can't handle customers of this side — but it will be a big loss for Apsis if we can't."

A grooming meeting was scheduled for the following day to discuss the solution approach.

### Secondary Problem: In-Memory Storage Causing Full Sync Crashes

During the full sync, the process:
1. Downloads all contacts from CRM → creates profile update messages
2. Triggers consent exports from **audience** for each mapped subscription
3. **Stores all results in memory** and compares incoming consent status against what's stored in Apsis (as an optimization to skip unchanged consents)

For massive contact loads, this in-memory storage causes the full sync to crash with memory errors.

### Proposed Fix: Remove the In-Memory Optimization, Stream Everything

The optimization being removed is: comparing incoming consent status against the current Apsis state to skip unchanged consents.

**Why the optimization exists:** To avoid re-processing consents that haven't changed.

**Why to remove it:** 
- The optimization causes confusion for customers anyway — if 10 consent messages were handled in one full sync, and nothing changed before the next full sync, those 10 messages won't be processed again, making it appear that fewer messages were processed. Customers can't explain this discrepancy.
- Removing it makes behavior more predictable and transparent.

**The new approach:** Stream everything — take each profile update message, convert it to the correct format, put it into **SQS**, and let the consumer process them as they arrive.

> "There is no issue with a lot of profile updates being in SQS. There is a lot of problem having them in memory in the producer."

### Priority Assessment

[Lukasz]: **Highest priority** of all items discussed. Should be planned for Q2 at the start of the quarter. Not in Q1 scope (scope closed), but should be the first thing picked up in Q2.

[Erik]: Fully agrees. This is the most important issue because it blocks handling existing big customers.

---

## Tribe CRM: Pagination Bug Producing Duplicate Contacts

### How It Was Discovered

A customer reported that their sync report showed 272 contacts downloaded from the CRM system, but only 204 tags appeared in Apsis. Erik investigated:

1. Downloaded the list from the CRM system directly → confirmed 272 records.
2. Checked the internal record of which contacts were processed in the last profile sync (used for tag/untag tracking) → also showed 204.
3. Downloaded contacts from the CRM using the same pagination method Apsis uses (pages of 50) → discovered that **some CRM IDs appeared in multiple pages**.

**Finding:** Out of 272 contacts, approximately 68 were **duplicates produced by Tribe's pagination**. The pagination result sets overlap.

### Impact

- The profile list sync is affected (confirmed).
- The **full sync may also be affected** — this has not been confirmed but is a concern.
- Apsis happily accepts duplicate IDs and doesn't deduplicate them in the sync process, so duplicates pass through silently.

### Responsibility

This is a **Tribe-side bug**. Apsis cannot fundamentally fix this — increasing page size is not a real solution since list sizes are unpredictable and pagination will always be required.

> "The ball is in Tribe's ballpark — they need to check and fix this."

Teams should be aware this issue exists and may generate customer support questions.

---

## Tribe CRM: Lead Entity Cleanup (Blocked Pending Tribe Confirmation)

### Background: The Virtual Lead Entity

In Tribe, for **outbound mappings** (form submissions from Apsis), there is a dropdown in the Tribe UI where the customer selects which CRM entity to create from form submissions. Because the generic connector cannot handle dynamically changing entity types, a **"lead" entity** was introduced in Apsis approximately 1.5 years ago.

### The Discovery

It was recently discovered that **regardless of what is selected in Tribe's dropdown**, Tribe's API always responds identifying the entity as a `contact`. Tribe handles the entity distinction internally. This means:

- The lead entity configuration inside Apsis is **completely unused and meaningless** for Tribe.
- It creates confusion: the most common support question for Tribe is "what is the lead entity in Apsis? We don't have leads in Tribe."

### What Needs to Be Done

1. Remove the lead entity configuration from the Tribe connector in Apsis.
2. Remove the lead entity from the additional entities loaded on the consent mappings configuration page.
3. Database cleanup: remove existing lead entity entries (e.g., key spaces).
4. *(Nice to have, lower priority)* Hide Tribe-specific lead attributes on customer accounts in Audience — achievable via delegation tokens and internal Audience APIs, or a database script.

[Erik]: "If we get the go-ahead, you should just do it. You should be able to do it in a day and it will remove so much complexity for customers."

### Why It Hasn't Been Done Yet

**Blocked:** Erik requires confirmation from the Tribe team that they *always* respond with contacts and never respond with other entity types in any situation. If Tribe does return other entity types and those are removed from Apsis's handling:

> "We will not do any merge. We will not add any CRM ID to it if it is an entity that we don't know how to process. Then you will have the form submission which is completely disconnected from the integration because we have no CRM ID. Then you do a full sync and the contact that they created from the form submission will be added to our key space, but that is completely separate from the submitted one from the form — and now you have two profiles in Apsis that are technically the same."

Erik has been waiting 3 weeks for a response from the Tribe team. He will write a story/ticket about this before leaving.

### Tribe's Organizational Context

- Tribe operates as a **more isolated/siloed system** within FSC.
- They refused to integrate with the central customer database in Maxo — Tribe and Maxo maintain separate customer sets.
- Tribe has its own separate customer service system (not Maxo).
- Tribe is currently building their own marketing module to reduce dependency on Apsis, but existing Apsis-integrated customers still need to be maintained.
- New customers are still being sold Tribe + Apsis together.
- Communication channel with Tribe exists but is currently unresponsive.

---

## Debugging Tool: Internal Broker Service for CRM Requests

### Problem It Solves

The generic connector Postman collection requires direct access to a customer's CRM system, which is rarely available. Credentials are encrypted in the database.

### How It Works

A **broker service** (referred to as the "internal broker service") allows developers to construct requests to a customer's CRM system and route them through Apsis infrastructure. The broker service automatically adds the correct authorization headers.

**Workflow:**
1. Get the customer's CRM system URL from the database.
2. Build the desired request (e.g., a paginated contact fetch).
3. Send the request to the internal broker service endpoint — it routes it to the CRM with correct auth.

This was used to diagnose the Tribe pagination bug: Erik manually requested pages 0, 1, 2, 3, 4, 5 separately and combined the results to identify duplicate CRM IDs.

> "This internal broker service is a very important tool to remember that we have because it's quite often that we need to make debug requests to the CRM system — how does it look when I make it manually and how does it look like inside of Apsis?"

A recording of a previous KT session covering this tool exists. Erik offered to demonstrate it again if needed.

---

## Key Takeaways

1. **Large customer scalability (8M+ contacts)** is the highest priority issue. The in-memory optimization in the full sync must be removed and replaced with SQS streaming. CRM-side sync condition evaluation must be implemented. Target: Q2.

2. **Improved full sync report** is back-end complete. The front-end change is low effort and low risk — suitable for a sprint with available front-end capacity. Not urgent.

3. **Extended sync conditions (AND/OR)** are back-end complete and tested by Prem. Front-end requires significant UI work. Before enabling for any account, Siteshop must confirm their V2 endpoint is implemented. FSC Enterprise also needs to be looped in. Priority: lower, after scalability work.

4. **Tribe pagination duplicates** are a Tribe-side bug. Apsis cannot fix it. Teams should be aware it exists and will likely generate support questions. May also affect full syncs.

5. **Tribe lead entity cleanup** is straightforward (estimated ~1 day) and will eliminate the most common Tribe support question. Currently **blocked** on Tribe team confirmation that they always return `contact` entity type. Without confirmation, removing this handling risks creating duplicate profiles in Apsis (form submission profile + full sync profile with no shared CRM ID).

6. **The internal broker service** is a critical debugging tool for making authenticated requests to customer CRM systems without direct access. All developers should be familiar with it.

---

## Unresolved Questions / Action Items

- [ ] **Tribe lead entity confirmation:** Get written confirmation from Tribe that their API always returns `contact` entity type regardless of dropdown selection. Erik to write a ticket/story. *(Blocked 3+ weeks at time of session)*
- [ ] **Sync conditions grooming:** Meeting scheduled for the day after this session to review Erik's proposed solution approach and write stories/estimates. Erik noted his suggestion may not be the optimal approach.
- [ ] **Tribe pagination:** Tribe needs to investigate and fix their pagination deduplication bug. Someone should follow up with the Tribe team.
- [ ] **FSC Enterprise sync conditions:** Enterprise team needs to be looped into the sync condition evaluation work for large customers.
- [ ] **Full sync scalability:** Architecture/approach to be confirmed in the grooming meeting; Apsis-side implementation of attaching sync conditions to outbound requests needs to be scoped.
- [ ] **⚠️ AMBIGUOUS:** The exact endpoint paths and payload schemas for the V1 vs. V2 sync condition endpoints in the generic connector were shown on screen but not verbalized in detail in the transcript. Refer to the codebase or a separate recording for specifics.