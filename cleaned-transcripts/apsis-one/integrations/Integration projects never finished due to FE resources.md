---
source_file: Integration projects never finished due to FE resources.txt
domain: Apsis One Integrations
topics: [Sync Reports, Sync Conditions, Large Customer Handling, Tribe Connector Issues, Pagination Problems, Performance Optimization]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Sync Reports, Sync Conditions, Full Sync, CRM Systems, Microsoft Dynamics, Tribe CRM, Generic Connector, SQS, Audience Module, Consent Messages, Profile Updates, Pagination]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Tribe]
---

## Session Overview

This knowledge transfer session covers unfinished integration projects and ongoing technical issues in the Apsis One Integrations domain. The discussion focuses on three major unfinished feature initiatives (improved full sync reports, extended sync conditions, and large customer performance optimization), two critical discovered issues (Tribe pagination duplicates and the virtual lead entity confusion), and their prioritization for future development. The root cause for incomplete work is identified as frontend resource constraints and competing priorities between integrations and core platform features like email and audience management.

---

## Incomplete Frontend Features

### Improved Full Sync Report — Separated Consent and Profile Statistics

**Context:** The current full sync report groups consent messages and profile update messages together, creating confusion for customers and support teams.

[Erik Andersson]: The sync report currently displays statistics like "successful 39,000 customers" but customers interpret this as 39,000 contacts when it actually represents a combination of 39,000 profile updates plus consent messages. This leads to confusion when a customer with only 8,000–10,000 contacts sees 39,000 successful syncs and wonders why so many profiles are being synced.

**Backend work completed:** Data model changes have been implemented to separate these statistics into:
- Amount of profile messages (created, successful, failed, skipped)
- Amount of consent messages (created, successful, failed, skipped)

This allows customers to generate granular reports showing exactly how many consents and profile updates were processed.

**Why it's not live on frontend:** [Erik Andersson]: Josh left the company, and integration stores were deprioritized in favor of email and audience management features. The backend implementation is complete and has been tested during backend releases, though more thorough testing is recommended.

**Recommendation:** [Erik Andersson]: "This should be a very low hanging fruit if you have someone that is swift at front end work and it will add a lot of value for the customer. If nothing else, it will reduce the support questions you get for the full syncs."

**Priority assessment:** [Lukasz Grabowski]: Low priority but low effort. Can be added if frontend developers have spare capacity.

---

### Extended Sync Conditions — Advanced Filtering Logic

**Current limitations:**

1. **Implicit AND logic only:** All conditions are combined with AND operators. Customers cannot express OR logic. For example, they can only say "email AND mobile AND birth date" but not "email OR mobile".

2. **Equals operator only:** Only exact matching is supported. Customers cannot filter for empty fields, non-empty fields, contains, starts with, ends with, or other comparison operators.

**Example current constraint:** To sync only contacts with a specific email, birth date, and mobile number, the system requires all three conditions to match (AND).

**Backend implementation completed:** 

[Erik Andersson]: The backend now supports segment builder format in sync conditions with added support for AND/OR operators. This enables support for:
- Equals
- Not equals
- Contains
- Starts with
- Ends with
- Additional operators

The goal is to allow customers to do "anything you could do in the segment builder, you should be able to do in the sync conditions."

[Erik Andersson]: Prem tested this work in the backend, but it is not yet implemented on the frontend, which requires "quite radical" changes to the UI.

**Critical CRM system consideration — Microsoft Dynamics:**

The sync conditions structure evaluation has two different approaches:

1. **Version 1 (Current):** A general endpoint sends conditions as:
   ```
   entity: [entity name]
   conditions: [implicit AND structure]
   ```
   Example: "First name = Elsa AND sync to Apsis = true" OR "entity = lead"

2. **Version 2 (New with AND/OR support):** A new endpoint tailored for the complex structure with AND/OR operators. This is more complex but significantly more powerful.

**Status by CRM system:**
- **Site Shop:** Already supports Version 2 and evaluates sync conditions on their side (generates pages based on condition configuration)
- **Microsoft Dynamics:** Does NOT yet support Version 2. [Erik Andersson]: "I don't think they have actually implemented support for the new structure because it never took off inside of inside APSIS."
- **Efficy Enterprise:** Will need to add support. [Erik Andersson]: "We have like this massive 8 million contacts CRM, so they will need to be involved in this as well."

**Important caveat:** [Erik Andersson]: Before enabling Version 2 for any account, verification with Site Shop and coordination with other CRM systems is required.

**Priority assessment:** [Lukasz Grabowski]: This is higher priority than the report improvement but should not rush. Should be scoped for Q2 development and reviewed alongside other backlog items.

---

## Critical Ongoing Issues Discovered After Initial Feature Work

### Large Customer Handling — Sync Conditions Evaluation Performance

**Problem statement:** The system cannot efficiently handle customers with 2+ million contacts in their CRM system.

**Root cause:** Sync conditions are currently evaluated inside Apsis, which requires downloading all contacts from the CRM system. For a customer with 8 million contacts where only 100,000 match the sync conditions, the system must download all 8 million, creating "insane data waste and time waste."

[Erik Andersson]: "We still need to download 8 million of them, which is an insane data waste and time waste for the customer."

**Solution approaches requiring CRM system changes:**

1. **Server-side filtering (Site Shop approach):** Store conditions as configuration in the CRM system and have it evaluate them or generate pages based on those conditions.

2. **Stateless API approach (Efficy Enterprise approach):** Include sync conditions as query parameters or request body when asking for contacts during full syncs and consent retrieval.

**Work required:**

- **CRM system side:** Most of the work. Each CRM system needs to implement condition filtering in their API.
- **Apsis side:** Implement functionality to extract sync conditions and add them as query parameters or in request body.

[Erik Andersson]: "Within this same project there is also one optimization in the full sync that needs to be removed because... if you have a massive load of profile stored then this will just utterly crash the full sync due to memory issues."

**Current optimization problem:** The system stores consent status comparisons in memory during full syncs. When consent exports are triggered from Audience for each subscription mapping, all results are stored in memory. With massive profile loads, this causes out-of-memory crashes.

**Proposed fix:** Remove the optimization that compares incoming consent status with existing Apsis status. Instead, stream all messages directly:
- Take profile update messages
- Convert to correct format
- Put directly in SQS
- Let consumer process them as they arrive

[Erik Andersson]: "There is no issue with a lot of profile updates being in SQS, there's a lot of problem having them in memory in the producer."

**Customer visibility problem:** Currently, if a consent message status hasn't changed since the last full sync, it's not processed. This makes customers see fewer messages processed on subsequent full syncs without understanding why, even though they made no changes to the CRM.

**Priority assessment:** [Lukasz Grabowski]: "My vote is that that one is the most important and it is high prior to plan development." This should be included in Q2 planning. [Erik Andersson]: Agrees absolutely. "The same condition, I mean like this is how it has been working since we released integration. There is no problem like keeping it for that for like another half year or year... whereas these the size of the customer is a kind of big problem."

---

## Tribe CRM Connector Issues

### Issue 1: Pagination Returning Duplicate Records

**Discovery:** A customer reported seeing 272 downloaded contacts from Tribe but only 204 tags in Apsis.

**Investigation process:**
1. Downloaded list from CRM system manually: confirmed 272 contacts
2. Checked database entries from last profile sync: only 204 (matches Apsis)
3. Downloaded contacts using the same pagination method used in profile syncs (50 records at a time): discovered duplicate CRM IDs across pages

**Finding:** Tribe has a pagination bug where the same contact appears in multiple pages. Of the 272 records returned, approximately 68 were duplicates.

[Erik Andersson]: "When I downloaded 50 contacts and then 50 and then 50 and then 50 and then 50, then I noticed that the some of the example CRM IDs were not included in the total set of the data. And what had happened is that Tribe has given us duplicates on the pagination."

**Impact:** This is consistent across queries and likely affects full syncs as well, though Apsis has not detected it because it "happily accept IDs" without deduplication.

**Why Apsis doesn't deduplicate:** The system design accepts whatever IDs the CRM provides without checking for duplicates.

**Why increasing page size doesn't help:** The issue is inherent to Tribe's pagination implementation. There's no way to know how many profiles exist in a list, so pagination must be used. Increasing page size doesn't solve the underlying duplicate problem.

**Current status:** "The ball is in tribes ballpark like they need to check and fix this. There's nothing Apsis can do about this."

**Action required:** Tribe must fix their pagination implementation. Apsis team should monitor for customer escalations about missing or duplicate contacts.

### Issue 2: Virtual Lead Entity — Confusion and Wasted Configuration

**Context:** In Tribe, the outbound flow (form submission to CRM) allows users to select which entity to create via a dropdown list. The generic connector in Apsis needs to know which entity to handle when setting up mappings.

**The problem:** 

[Erik Andersson]: "The lead entity as I described before, this is not like something which really exists inside of tribe, because in tribe for the outbound mappings you have a drop down list which selects like which entity should I create from form submissions inside of Apsis, but the generic connector can't really handle like dynamic entities like we we need to know like which entity should I which entity should I download the attributes for here so we can set up the the mapping."

The Apsis solution was to create a "lead" entity configuration to represent this dynamic selection. However, recent discovery suggests this is unnecessary.

**Recent finding:** Tribe always responds with "contact" entity regardless of what is selected in their dropdown. They handle the distinction internally.

[Erik Andersson]: "We did discover quite recently also that no matter what you select in that drop down list inside of tribe, tribe is always responding saying this is a contact. And then they are handling this inside of tribe. And that means that this whole lead entity set up inside of Apsis is worthless and just creates confusion."

**Consequences of the current setup:**

1. Configuration confusion: Customers don't understand why there's a "lead" entity in Apsis when Tribe doesn't have leads
2. Redundant configuration: Lead entity is loaded as an additional entity in configuration
3. Unnecessary complexity: Lead entity appears in consent mappings configuration page, adding visual clutter

[Erik Andersson]: "This is the single most question I've got them regarding tribe like what is the lead in abscess because you don't have lead in tribe... trying to explain to someone like what a virtual entity is, is not a big or that's not a very fun thing to do."

**Proposed solution:** Remove all lead entity handling for Tribe and only support contact entity.

**Implementation scope:**
1. Remove the lead entity configuration for Tribe in Apsis
2. Hide the tribe lead attributes on customer accounts (audience operation, may require coordination)
3. Clean up database entries for Tribe leads

**Why it hasn't been done:** [Erik Andersson]: "Because I want confirmation from tribe that I can actually go ahead and do this because I have previously explained to every consultant why we need to have this now... I just, I really need the confirmation from tribe that they don't do any other shenanigans and give us other entities than contacts in some situations because if they do, we will start losing their form submissions."

**Risk if implemented without confirmation:** If Tribe ever sends an entity type other than "contact" and Apsis doesn't have handling for it, form submissions will not map correctly to CRM records:

[Erik Andersson]: "We will get the form submissions from them. There will be a profile created from the form submission. Then we will try to send this to the CRM system and we will not do any merge. We will not add any CRM ID to it... then you do a full sync and then the contact that they created from the form submission will be added to our key space, but that is completely separate from the submitted one from the form and now you have two profiles in UPS that are technically the same ones."

**Current status:** Awaiting written confirmation from Tribe (no response after 3 weeks). [Erik Andersson] plans to create a story during the day so the work is documented.

**Effort and priority:** [Erik Andersson]: "If we there we get the go ahead, I think you should just like you should just do it because it should be. You should be able to do it in a day and it will remove so much complexity for the customers."

Priority remains pending Tribe confirmation.

---

## Tribe Connector Context — System Independence

[Erik Andersson]: Tribe operates as a relatively isolated system within FSC. They "go in their own way with things" and don't integrate with other FSC systems where possible:

- **Refused central customer database integration:** Tribe does not integrate with the Maxo central customer database. This results in duplicate customer records across systems.
- **Separate customer service system:** Tribe customer cases must be registered in Tribe's system, not Maxo
- **Marketing module independence:** Tribe is building its own marketing module to reduce reliance on Apsis

**Implication for integration maintenance:** Despite Tribe's preference for independence, they still have customers integrated with Apsis. The integration must be maintained, and many customers are sold Tribe+Apsis as a bundle.

[Erik Andersson]: "We still have customers that are integrated with Apsis, so we still need to maintain this one. And also there are a lot of customers that have are being sold tribe together with Apsis, so [we need to continue supporting it]."

**Collaboration challenges:** [Erik Andersson]: "We have the channel with them. It's just that I right now can't get any answers from them." Communication with Tribe developers is difficult and slow.

**General connector stability note:** [Erik Andersson]: "That whole connector kind of needs a revamp on both sides I think because now they are getting so many more customers and tribe and it is by far the most not stable connector of them all, not so much maybe on app C side, but the virtual entity setup certainly doesn't help for it."

---

## Debugging Methodology — Internal Broker Service

**Tool reference:** When investigating the Tribe pagination issue, [Erik Andersson] used a critical internal debugging tool called the **internal broker service**.

**What it does:** The broker service allows construction of debug requests to customer CRM systems without needing direct access credentials. It routes requests through the broker, which adds correct authorization headers.

**How it works:**

1. Retrieve the customer's CRM system URL from the database
2. Build the endpoint request (e.g., schema retrieval, paginated list queries)
3. Send the request to the internal broker service endpoint
4. The broker adds correct authentication and forwards to the CRM
5. Results are returned for analysis

**Why it's important:** [Erik Andersson]: "This is a very important tool to remember that we have because it's quite often that we need to make like debug requests to the CRM system... Like how does it look when I make it manually and how does it work?"

**Example usage:** To debug the Tribe pagination issue:
```
Request: page=0, page=1, page=2, page=3, page=4 (50 records each)
Result: Combined the pages and compared for duplicates
Finding: Same CRM IDs appeared across multiple pages
```

**Note:** A recording exists of the broker service functionality from a previous KT session. Direct demonstration available if needed.

**Learning curve:** [Erik Andersson]: "The more you work with this, like the more things you will, the more you will remember and find fast."

---

## Development Prioritization Summary

Based on the discussion, the team prioritized the backlog as follows:

| Feature/Issue | Priority | Effort | Timing | Notes |
|---|---|---|---|---|
| Full sync report improvement | Low | Low | After Q1 | Frontend-only. Can be picked up if developers have spare cycles. |
| Extended sync conditions | Medium-High | Medium | Q2 | Requires frontend implementation, CRM system coordination (especially Dynamics), and testing. Should not rush; needs grooming meeting to finalize approach. |
| Large customer handling | **Highest** | High | Q2 | Most critical for business. Requires both CRM system changes and Apsis optimization. Memory/performance issue with current full sync design. |
| Tribe lead entity removal | High (conditional) | Very Low (1 day) | Pending confirmation | Trivial to implement once Tribe confirms they only send "contact" entity. High value for customer experience. |
| Tribe pagination issue | Monitor | External | — | No action required from Apsis. Tribe must fix their pagination. Monitor for customer escalations. |

[Lukasz Grabowski]: "I think we we should include this in in in the next quarter because not for now, I think it's the scope is closed for Q1, but for Q2 I would like to have it and start doing this at the beginning."

---

## Future Collaboration Plans

[Lukasz Grabowski]: Erik will continue as a contractor during the next phase of work. A grooming meeting is scheduled for tomorrow to finalize the approach and user stories for the extended sync conditions feature.

[Erik Andersson]: Multiple colleagues will be available for follow-up discussions and detailed technical reviews of the proposed solutions.

---

## Key Takeaways

1. **Three major incomplete features exist due to frontend resource constraints:**
   - Full sync report improvements (backend complete, frontend pending)
   - Extended sync conditions (backend complete, requires CRM system coordination and frontend work)
   - These were not finished when Josh left and integrations were deprioritized

2. **Large customer handling is the most critical unresolved issue:** The current architecture cannot efficiently handle customers with 2+ million contacts. Solving this requires coordinated changes across CRM systems and Apsis, with memory optimization on the Apsis full sync path.

3. **Tribe connector has multiple stability concerns:**
   - Pagination duplicates are a Tribe-side bug that needs their attention
   - Virtual lead entity is a design mistake causing customer confusion and should be removed once Tribe confirms behavior
   - Tribe operates as an isolated system within FSC, making collaboration challenging

4. **The internal broker service is essential for CRM debugging:** It allows authenticated requests to customer systems without direct credential access. This tool was critical for discovering the Tribe pagination issue.

5. **CRM system coordination is non-trivial:** Changes to sync conditions affect multiple CRM systems differently (Site Shop handles server-side filtering; Efficy Enterprise will need similar support). This coordination should not be rushed.

6. **Customer impact is real:** The full sync report confusion generates support tickets. The lead entity confusion generates frequent customer questions. These are not purely technical issues.

---

## Unresolved Questions & Action Items

1. **Awaiting Tribe confirmation:** Can Tribe confirm they always respond with "contact" entity and never send other entity types in form submission outbound mappings? (Erik to create story documenting this requirement)

2. **Extended sync conditions approach:** Multiple solution approaches exist for handling large customers and advanced filtering. Tomorrow's grooming meeting will finalize the technical approach and break down into user stories with estimates.

3. **CRM system readiness:** Before rolling out extended sync conditions Version 2, need to verify Microsoft Dynamics and Efficy Enterprise support for the new AND/OR condition structure (currently only Site Shop supports it).

4. **Database cleanup for Tribe leads:** If confirmation is received, what's the process for cleaning up existing lead entity configuration and database entries in production customer accounts?