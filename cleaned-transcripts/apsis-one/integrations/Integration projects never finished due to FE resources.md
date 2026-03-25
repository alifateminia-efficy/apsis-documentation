```yaml
source_file: Integration projects never finished due to FE resources.txt
domain: Apsis One Integrations
topics: 
  - Full sync reporting and statistics
  - Sync conditions and filtering
  - Large customer handling and performance optimization
  - Tribe connector issues and pagination
  - Microsoft Dynamics integration
  - Lead entity handling in Tribe
session_type: knowledge-transfer
speakers: 
  - Erik Andersson (Integration domain expert, departing)
  - Lukasz Grabowski (Team lead)
  - Tomasz Kowalski (Developer)
  - Michal Rosikiewicz (Developer)
key_components:
  - Full sync reports
  - Sync conditions engine
  - Tribe connector
  - Microsoft Dynamics via Siteshop
  - FSC Enterprise
  - Audience consent exports
  - Internal broker service
  - SQS queue system
subdomains:
  - Architecture
  - Microsoft Dynamics Integration
  - Tribe Integration
  - Duplicate profiles in Apsis
---

## Session Overview

This knowledge transfer session covers three major unfinished integration projects and several ongoing issues discovered during maintenance work. Erik Andersson, a departing integration domain expert, walks the team through incomplete frontend work, performance bottlenecks affecting large customers, and critical connector issues with the Tribe CRM system. The session establishes priorities for Q1/Q2 development and provides context on architectural decisions that now require revision.

---

## Improved Full Sync Report — Unfinished Frontend Work

### The Problem with Current Reporting

The current full sync report conflates two different types of messages in its statistics, creating customer confusion:

> When the report says "successful 39,000 customers," clients interpret this as 39,000 contacts or profile updates. However, this number actually combines both profile update messages AND consent messages.

For example, a customer with 8,000–10,000 actual contacts sees a report claiming 39,000 successful syncs and wonders why the count is so inflated. This leads to frequent support escalations and NPS impact. [Erik Andersson]

### Backend Implementation Already Complete

The team has already separated the statistics in the backend data model. The sync report backend now tracks:
- Amount of profile messages (created, failed, skipped)
- Amount of consent messages (created, failed, skipped)

This allows granular reporting per-message type, but **this work has not been reflected in the frontend UI**.

**Why it wasn't finished:** Josh (previous frontend developer) left the company, and integration work was consistently deprioritized in favor of email and audience features. [Erik Andersson]

### Recommended Action

> This should be a very low hanging fruit if you have someone swift at frontend work, and it will add a lot of value for the customer. If nothing else, it will reduce the support questions you get for full syncs. [Erik Andersson]

**Caveat:** The backend implementation needs further testing beyond the initial validation done during backend release. [Erik Andersson]

**Priority:** Low effort, low priority — can be picked up by frontend developers with spare capacity in Q1. [Lukasz Grabowski]

---

## Extended Sync Conditions — Unfinished Frontend & CRM Alignment

### Current Limitations

The existing sync conditions feature has two critical weaknesses:

1. **Implicit AND logic only:** All conditions are combined with AND. Customers cannot use OR logic.
   - Example: "email equals X AND mobile equals 123456 AND date_of_birth equals yesterday"

2. **Equals operator only:** The system only supports equality checks. No support for:
   - `not_equals`
   - `contains`
   - `starts_with`
   - `ends_with`
   - Empty/non-empty checks

### Backend Work — Segment Builder Format Support

The backend now supports **segment builder format** in sync conditions with extended operators. [Erik Andersson]

The team added:
- OR operator support (in addition to AND)
- All segment builder operators: equals, not_equals, contains, start_with, ends_with, etc.

**Status:** Implemented in backend, tested by Prem, but **not exposed in the frontend UI** (requires significant UI redesign). [Erik Andersson]

### CRM System Complications

This feature has consequences across multiple CRM connectors because sync conditions can be evaluated either:

**Approach 1 — CRM-side evaluation (Siteshop model):**
- Conditions stored as configuration in the connector
- CRM system evaluates conditions and generates paginated results
- Current endpoint used: `/general/connector` with implicit AND structure only
- New endpoint tailored for V2: supports the AND/OR structure but **only Siteshop has implemented support so far**

**Approach 2 — Apsis-side evaluation (FSC Enterprise model):**
- Stateless API; Apsis must include conditions as query parameters in sync requests
- Requires CRM to filter results based on parameters sent in requests

### Important Caveat on Version 2 Rollout

> Before enabling the V2 version of sync conditions for any account, this needs to be verified with Siteshop because I don't think they have actually implemented support for the new structure because it never took off inside Apsis. [Erik Andersson]

FSC Enterprise will also need to be involved because they now have an 8 million contact CRM and require proper sync condition handling.

**Recommendation:** Don't rush this feature. Finish all other items on the list first due to dependencies on other CRM systems. [Erik Andersson]

**Priority:** Nice-to-have / lower priority compared to performance optimization. Lukasz suggests postponing beyond Q1. [Lukasz Grabowski]

---

## Large Customer Performance Optimization — Critical Issue

### The Problem: Evaluating Conditions In-Memory

Today, sync conditions are evaluated **inside Apsis**, which means:

1. Download **all** contacts from the CRM system (even if customer has 8 million)
2. Apply sync conditions in memory to filter to the desired subset (e.g., 100,000 contacts)
3. Create profile update messages for filtered contacts

For a customer with 8 million contacts where only 100,000 match the sync conditions, this is massive data and time waste. [Erik Andersson]

### Required Solution: Push Conditions to CRM

To handle large customers, CRM systems must evaluate sync conditions before returning data:
- **Siteshop approach:** Store conditions as configuration; they generate filtered pages
- **FSC Enterprise approach:** Include conditions in API request parameters; they filter their own results

Apsis must implement the ability to:
- Transform sync conditions into query parameters or request body format
- Pass conditions to CRM systems during full syncs and consent retrieval
- Handle both AND/OR condition structures

### Memory Optimization: Remove Consent Comparison Cache

Within the full sync, there's an optimization that stores all consent export results in memory to avoid reprocessing unchanged consents:

**Current flow:**
1. Download all contacts → create profile update messages
2. Trigger consent exports from audience for each subscription mapping
3. **Compare incoming consent status with Apsis status**
4. Store results in memory
5. Send to SQS

**Problem:** With massive contact loads, this in-memory cache causes memory exhaustion and crashes. [Erik Andersson]

**Proposed solution:** Remove this optimization and stream everything:
- Take profile update messages → convert to correct format → put in SQS immediately
- Handle each consent message as it arrives
- No in-memory comparison of previous state

> There is no issue with a lot of profile updates being in SQS. There is a lot of problem having them in memory in the producer. [Erik Andersson]

**Side benefit:** Clearer behavior for customers. If a consent status hasn't changed since last full sync, it won't be reprocessed, but the customer sees the same number of messages processed each run (rather than seeing inconsistent counts). [Erik Andersson]

### Priority Assessment

> My vote is that this one is the most important and it is high priority to plan development. [Lukasz Grabowski]

**Erik's concurrence:** The other items (sync report, sync conditions) have been working since initial release with no urgent problems. This performance issue is a blocker for growing into the large customer segment. [Erik Andersson]

**Planned timeline:** Include in Q2 planning at the beginning of the quarter. [Lukasz Grabowski]

---

## Tribe Connector Issues

### Pagination Duplicate Bug

**Discovery:** A customer reported downloading 272 contacts from Tribe but only seeing 204 in Apsis.

**Investigation process:**
1. Downloaded data from Tribe API directly → confirmed 272 contacts in response
2. Compared against internal tracking of last profile sync → also showed 204 contacts
3. Tested pagination manually: requested 50, then 50, then 50, then 50...
4. Found that Tribe was returning duplicate CRM IDs across page boundaries

> When I downloaded 50 contacts and then 50 and then 50 and then 50 and then 50, I noticed that some of the example CRM IDs were not included in the total set of the data. Out of those 272 there were like 68 duplicates. This was like quite consistent. [Erik Andersson]

**Root cause:** Tribe's pagination implementation has bugs returning overlapping results between pages.

**Impact:** 
- Unknown if this affects full syncs (likely does)
- Apsis happily accepts duplicate IDs with no deduplication in the sink
- Cannot simply increase page size as a workaround because we don't know list sizes in advance

**Resolution:** Ball is in Tribe's court. They must investigate and fix their pagination. [Erik Andersson]

**Awareness item:** This is not something Apsis can directly fix; teams should be aware of the issue for customer escalations.

### Lead Entity Configuration — Worthless Setup

**Background:** In Tribe, the outbound mappings allow selection of which entity to create from form submissions:
- Dropdown in Tribe UI lets you select an entity
- Apsis generic connector cannot handle dynamic entities, so it needs a fixed entity list
- Apsis was configured with a "lead" entity to handle this

**Discovery:** Regardless of what is selected in the dropdown, Tribe always responds with entity type = "contact" and handles it internally. The "lead" entity selection in Apsis has zero effect.

> The generic connector can't really handle dynamic entities like we need to know which entity should I download the attributes for here so we can set up the mapping. But we did discover quite recently that no matter what you select in that drop down list inside of tribe, tribe is always responding saying this is a contact. [Erik Andersson]

**Current state:** The Tribe configuration in Apsis includes:
- Lead entity configuration (useless)
- Lead entity loaded in additional entities
- Lead entity appears on consent mapping configuration page

This creates customer confusion about what "lead" means.

**Proposed solution:** Remove all lead-related configuration from Apsis for Tribe:
1. Remove lead entity configuration from Tribe settings
2. Remove lead entity from additional entities loading
3. Remove lead entity from consent mapping page
4. Database cleanup of lead-related entries for Tribe

**Blockers:** Erik has awaited confirmation from Tribe for 3 weeks without response. Cannot proceed until confirmed that Tribe always returns "contact" type. [Erik Andersson]

**Why confirmation is critical:** If Tribe ever returns a different entity type (e.g., "deal", "account"), and Apsis doesn't know how to process it:
- Form submission creates a profile in Apsis
- Apsis cannot add a CRM ID (doesn't recognize the entity)
- Apsis doesn't merge with existing profiles
- Full sync later creates a duplicate profile for the same contact
- Two disconnected profiles exist in Apsis for the same entity

> We will create duplicates because we will get the form submissions from them. There will be a profile created from the form submission. Then we will try to send this to the CRM system and we will not do any merge. We will not add any CRM ID to it if it is an entity that we don't know how to process... now you have two profiles in Apsis that are technically the same ones. [Erik Andersson]

**Priority:** Once confirmation is received, should be completable in a single day with high customer impact (reduces confusion significantly). [Erik Andersson]

> This is the single most question I've got them regarding tribe — what is the lead in Apsis because you don't have lead in tribe. [Erik Andersson]

**Additional work:** May require audience operations to hide Tribe lead attributes from customer accounts (if Apsis has delegation tokens for audience APIs).

---

## Technical Debugging Tools & Methods

### Internal Broker Service for CRM Debugging

To debug integration issues without direct access to customer CRM systems:

**Tool:** Internal broker service endpoints that route requests through a proxy with correct authorization headers

**Use case:** Construct requests to customer CRM systems and see responses, useful for:
- Testing pagination behavior
- Checking API schema
- Comparing expected vs. actual data formats
- Validation during issue investigation

**Process:**
1. Build endpoint URL from customer database
2. Construct request (e.g., `page=0`, `page=1`, `page=2` for pagination testing)
3. Send to broker service
4. Broker adds authorization and forwards to customer's CRM
5. Response shows actual CRM behavior

[Erik Andersson] used this tool to discover the Tribe pagination duplicates. [Tomasz Kowalski] confirmed the tool was referenced in a previous KT session with recording available.

### Generic Connector Collection (Postman)

A Postman collection exists with request templates for each CRM connector, but requires direct database access to decrypt customer credentials (rarely available). [Erik Andersson]

---

## Tribe as a Special Case: Organizational Silos

**Context:** Tribe operates as a more isolated system within FSC compared to other connectors.

**Characteristics:**
- Refuses to integrate with central customer database (Maxo) — maintains separate customer set
- Uses separate customer service system (cases registered in Tribe, not Maxo)
- Developing a separate marketing module for Tribe to reduce Apsis dependency
- Takes an independent approach to systems integration

**Current state:** Despite these silos, Apsis still maintains a Tribe connector because:
1. Existing customers have integrated Apsis with Tribe
2. New customers are being sold both Tribe and Apsis together

**Communication challenges:** The Tribe team does not respond promptly to integration questions (e.g., 3 weeks without confirmation on lead entity). [Erik Andersson]

> Tribe kind of likes go in their own way with things. They don't want to integrate with other systems... the issue is that tribe kind of likes go in their own way with things. They don't want to integrate with other systems. [Erik Andersson]

**Connector stability assessment:** Tribe is "by far the most not stable connector of them all," though this may be partly due to the virtual entity setup complexity rather than pure Apsis-side issues. Tribe will likely need a revamp as customer numbers increase. [Erik Andersson]

---

## Prioritization Summary & Q1/Q2 Planning

### Agreed Priorities

**Q1 (Closed scope — no new commitments)**

**Q2 (Start of quarter):**
1. **Large customer performance optimization** — HIGH PRIORITY
   - Address memory crash with large contact loads
   - Push sync conditions to CRM systems
   - Stream consent messages instead of caching
   - Estimated as bigger scope due to testing requirements and cross-CRM coordination

2. **Extended sync conditions** — MEDIUM PRIORITY (lower than performance)
   - Nice-to-have feature
   - Can wait beyond Q2 if needed
   - Significant UI redesign required

**Backlog / As-available:**
1. **Improved full sync report** — LOW PRIORITY, LOW EFFORT
   - Frontend-only work
   - Pick up if frontend developers have capacity
   - No blocking dependencies

2. **Tribe lead entity removal** — DEPENDS ON EXTERNAL CONFIRMATION
   - High impact if approved (1 day of work)
   - Waiting for Tribe team confirmation
   - Erik to document as a story for handoff

---

## Unresolved Questions & Action Items

### Awaiting External Confirmation
- **Tribe lead entity:** Erik has been waiting 3 weeks for Tribe team to confirm that Tribe always returns "contact" entity type regardless of dropdown selection. Without this, cannot proceed with removing lead entity configuration. [Erik Andersson] to create story and notes for team.

### Grooming Meeting Scheduled
- **Sync conditions query parameters:** Tomorrow's grooming meeting will discuss implementation approach for passing sync conditions to CRM systems. Erik noted he has a suggestion but unsure if it's the best approach. [Erik Andersson]

### Frontend Estimation Needed
- **Full sync report UI:** Needs frontend estimation once sprint planning occurs

---

## Key Takeaways

1. **Three major projects were started but not completed due to frontend resource constraints:**
   - Improved full sync report (backend done, UI incomplete)
   - Extended sync conditions (backend done, UI incomplete)
   - These represent significant technical debt and customer confusion

2. **Large customer handling is a critical gap** that will lose business if not addressed. Current architecture downloads all contacts regardless of sync conditions, making the system unscalable for 2M+ contact CRMs.

3. **Tribe connector has multiple issues:**
   - Pagination returns duplicates (customer-side fix needed)
   - Lead entity configuration is a vestigial workaround creating confusion (requires external confirmation to remove)
   - More unstable than other connectors and likely to need revamp as adoption grows

4. **The internal broker service is a valuable debugging tool** for investigating integration issues without direct customer access. More developers should know about and use it.

5. **Apsis cannot unilaterally fix all integration issues** — some (like Tribe pagination) require fixes from the CRM system side. Clear communication of responsibility and escalation is important.

6. **Erik Andersson is available as a contractor for tomorrow's grooming meeting**, providing continuity for detailed technical discussions on the sync conditions implementation.