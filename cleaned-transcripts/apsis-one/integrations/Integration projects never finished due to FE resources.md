---
source_file: Integration projects never finished due to FE resources.txt
domain: Apsis One Integrations
topics: [Full Sync Report Improvements, Sync Conditions Enhancement, Large Customer Handling, Tribe Pagination Issues, Entity Configuration Management]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Full Sync Reports, Sync Conditions, Generic Connector, Profile Updates, Consent Messages, Microsoft Dynamics, Tribe CRM, Enterprise CRM, Audience Module, SQS]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.1, Tribe]
---

## Session Overview

This knowledge transfer session documents three major integration features that were started but not completed due to front-end resource constraints, plus two critical ongoing issues discovered during operations. Erik Andersson (departing engineer) reviews unfinished work on the full sync report UI, extended sync conditions, and outstanding bugs in the Tribe connector. The session covers the technical rationale for each feature, their importance to customers, and prioritization guidance for completing this work.

---

## Improved Full Sync Report — UI Implementation Pending

### Problem with Current Reporting

The current full sync report groups **profile update messages** and **consent messages** together under a single "successful" count, creating confusion for customers and account managers.

**Example**: A report showing "39,000 successful" leads customers to believe 39,000 contacts were synced, when actually this number includes both profile updates AND consent messages combined. If a customer only has 8,000–10,000 actual contacts, they wonder why 39,000 are being processed.

[Erik Andersson]: > "both the consent and profile update messages they are grouped up. So when when it says here successful 39,000 customers tend to think that this is 39,000 contacts or profile updates, then they wonder like why? Why are you syncing so many profiles?"

### Data Model Changes Already Completed (Backend)

The backend has already been refactored to **separate statistics**:

- **Amount of profile messages** (created, successful, failed, skipped)
- **Amount of consent messages** (created, successful, failed, skipped)
- Still maintain total and successful counts for backwards compatibility

This allows granular reporting — customers can see exactly how many consents were produced, how many profiles were created, etc.

### Why It's Not in the UI Yet

[Erik Andersson]: The work is done in backend "but the integration stores were always down prioritised in favour of e-mail and audience." The engineer who would have completed the frontend work (Josh) left the company, so the UI never got updated to reflect the new data model.

### Testing Status & Recommendations

[Erik Andersson]: > "this needs to be more tested because we've, I mean we've not had the real ability to fully test it, but it is working as far as as far as we saw when we tried it for the back end release."

**Priority Assessment** [Lukasz Grabowski]: Marked as low priority but low effort. This is a good candidate for frontend developers with spare capacity in Q1.

**Value**: Will significantly reduce customer support questions about full sync statistics.

---

## Extended Sync Conditions — Complex Feature Partially Implemented

### Current Limitations of Sync Conditions

Sync conditions allow customers to filter which contacts/profiles get synced into Apsis. Currently has two critical limitations:

**Limitation 1: Implicit AND only**
- Example: "email = X AND mobile = 123456 AND date_of_birth = yesterday"
- No ability to use OR logic

**Limitation 2: Equals operator only**
- Can only do `field = value`
- Customers request: `email is not empty`, `field contains X`, `field starts with`, `field ends with`

### Backend Implementation Complete

Erik has extended the backend to support **segment builder format operations** (equals, not-equals, contains, starts-with, ends-with, etc.) plus **AND/OR logic**.

[Erik Andersson]: > "what we wanted to do with this was essentially like anything you could do with in the segment builder you should be able to do in the sync conditions"

The backend work is done and has been tested by Prem, but the **frontend requires radical changes** to support the new condition builder UI.

### Critical Complication: CRM System Integration

#### Microsoft Dynamics (via Site Shop)

Site Shop has **two different sync condition endpoints**:

1. **Old endpoint** (current): Uses `section_discriminator` — only supports implicit AND structure
   ```
   entity: "lead"
   conditions: 
     - first_name equals "Elsa"
     - sync_to_apsis equals true
   ```

2. **New V2 endpoint** (for extended conditions): Supports the complex AND/OR structure but is more complex
   ```
   entity: "lead"
   conditions: [complex nested structure with AND/OR]
   ```

**Issue**: Before V2 conditions can be enabled for any account, Site Shop must implement support for the new structure. [Erik Andersson]: "I don't think they have actually implemented support for the new structure because it never took off inside of inside APSIS."

#### Other CRM Systems

- **Tribe**: Does not yet support sync conditions
- **Enterprise (FSC)**: Has a stateless API; sync conditions would need to be passed as query parameters in requests to their system

[Erik Andersson]: "me and Walker had discussions with enterprise that they will also need to support sync conditions now that we have like this massive 8 million contacts CRM"

### Recommendation

[Lukasz Grabowski]: Don't rush this feature. Complete other items first. Schedule for Q2 start, not Q1. It has consequences across multiple CRM integrations that need coordination.

[Erik Andersson]: Agrees — sync conditions have worked since integration release; no urgent need to change in the next 6–12 months.

---

## Large Customer Handling — Performance Crisis & Solution

### The Problem: Sync Conditions Must Be Evaluated in Apsis

When a customer has **millions of contacts** in their CRM but only wants to sync a subset, Apsis currently:

1. Downloads **all contacts** from the CRM system
2. Evaluates sync conditions **in memory** in Apsis
3. Filters to the desired subset

**Example**: Customer has 8 million contacts but only wants 100,000. Apsis downloads all 8 million, then filters. This is catastrophic for bandwidth, time, and cost.

[Erik Andersson]: > "if they have 8 million but we only want 100,000 of them. We still need to download 8 million of them, which is an insane data waste and time waste for the customer."

### Solution Architecture: Push Filtering to CRM Systems

**Approach 1 (Site Shop / Microsoft Dynamics)**
- Store sync conditions as configuration
- CRM system evaluates them and returns only matching records
- Already implemented by Site Shop

**Approach 2 (FSC Enterprise)**
- CRM has stateless API
- Apsis includes sync conditions in the query parameters or request body
- Example: `GET /contacts?filter=email_not_empty&filter=name_starts_with=John`
- CRM filters server-side and returns only matching records

**Apsis-side work needed**:
- Implement feature to take sync conditions and add them to query parameters or request body
- Grooming meeting scheduled to discuss implementation approach

**CRM-side work needed**: Each CRM system must be updated to handle sync conditions in API requests.

### Memory Optimization: Remove In-Memory Consent Caching

A secondary optimization issue must be fixed:

**Current process** in full sync:
1. Download all contacts, create profile update messages
2. Trigger consent exports from Audience module **for each subscription mapping**
3. **Store all results in memory**

With massive datasets, this in-memory storage crashes the full sync.

**Proposed fix**:
- Remove the in-memory caching optimization
- Stream each message directly to **SQS** queue instead
- Consumer pulls from SQS as capacity allows

[Erik Andersson]: > "there is no issue with a lot of profile updates being in SQS, there's a lot of problem having them in memory in the producer."

**Additional benefit**: Removes confusing behavior where subsequent full syncs show fewer processed messages without any CRM changes (because old cached consents aren't re-processed).

### Priority Assessment

[Lukasz Grabowski]: > "From those 3 epics, my vote is that that one is the most important and it is high prior to to plan development"

[Lukasz Grabowski]: Plans to include this in **Q2 sprint planning**, not Q1. Scope is closed for Q1.

**Why it's critical**: APSIS is at risk of losing large deals if it cannot handle customers with millions of contacts. This has already been discussed historically as a blocker.

---

## Tribe Connector: Pagination Duplication Bug

### Issue Discovery

A customer reported: "APSIS says we synced 272 contacts from Tribe, but only 204 are tagged in Apsis."

Erik investigated by:
1. Downloading the list directly from Tribe CRM — confirmed 272 in downloaded data
2. Checking the last profile sync tracking — showed 204 (matching Apsis count)
3. Simulating Apsis's actual pagination logic with page size 50:
   - Downloaded pages: 50, then 50, then 50, then 50, then 50
   - Compared CRM IDs across pages
   - **Found duplicate CRM IDs returned across pages**

**Root cause**: Tribe API has pagination bugs. Out of 272 records, **68 were duplicates** across pages (68/272 = ~25% duplication).

[Erik Andersson]: > "Tribe has given us duplicates on the pagination... this would appear that they have some quite big issues with their pages and this might also affect their full sync, which is not a very pleasant thought."

### Why Apsis Didn't Catch This Initially

Apsis's sync process doesn't deduplicate CRM IDs, so duplicates simply flow through the system. The discrepancy only became visible when comparing counts to actual tags created.

### Likelihood of Affecting Full Sync

[Erik Andersson]: The pagination duplication might also affect Tribe's full sync performance, but this is speculative.

### Resolution Path

**The issue is Tribe's responsibility** — they must fix their pagination API. Apsis cannot easily work around this because:
- Increasing page size doesn't solve the fundamental problem
- Unknown how many records exist in each CRM list, so pagination is necessary
- No standard way to deduplicate without additional API calls

[Erik Andersson]: > "this is something to more be aware of that this is an issue... the ball is in tribes ballpark like they need to check and fix this. There's nothing Apsis can do about this."

**Action**: Ball is currently with Tribe to investigate and fix. This is a known issue to monitor.

---

## Tribe Connector: Incorrect Lead Entity Configuration

### The Confusion

In Tribe, the generic connector is configured with a virtual "**lead entity**" for outbound mappings (form submissions creating new records in Tribe).

However, Tribe's actual behavior doesn't match this setup:

**What we thought**:
- Tribe has different entity types (contact, lead, etc.)
- Dropdown in Tribe allows selecting which entity type to create from form submissions
- Apsis needs to know which entity to expect to configure mappings

**What's actually happening**:
- **Regardless of what is selected in the Tribe dropdown, Tribe always responds with entity type "contact"**
- Tribe handles the routing internally — they don't return different entity types

[Erik Andersson]: > "no matter what you select in that drop down list inside of tribe, tribe is always responding saying this is a contact. And then they are handling this inside of tribe."

### Configuration Complexity This Causes

Because the lead entity is configured, it appears in multiple places:

- Tribe integration configuration page (lead entity configuration section)
- Consent mappings (additional entities shows lead)
- Entity dropdown in form submission mappings

**Result**: Customer confusion. [Erik Andersson]: > "this whole lead entity set up inside of Apsis is worthless and just creates confusion... this is the single most question I've got them regarding tribe like what is the lead in abscess because you don't have lead in tribe."

### Proposed Solution

**Remove all lead entity references for Tribe**:
1. Delete the lead entity configuration from Tribe settings
2. Remove lead from additional entities list
3. Only handle contacts in Tribe integration

**Effort estimate**: Should be completable in **one day**

**Benefits**:
- Eliminates customer confusion
- Reduces support questions about the "lead entity"
- Simplifies configuration

### Why It Hasn't Been Done Yet

[Erik Andersson]: > "I have after like 3 weeks still not gotten any response from anyone. If this is the case, if I can go ahead with this, that's why it is not been done."

**Blocker**: Needs confirmation from Tribe that they genuinely always return contacts and never return any other entity type. If Tribe does return other entity types in some scenarios, removing this code would cause form submissions to be lost or create duplicates.

**Risk scenario** if removed without confirmation:
- Form submission arrives and creates a contact in Apsis
- Apsis tries to send it to Tribe
- Tribe returns an unexpected entity type that Apsis doesn't recognize
- Apsis can't add CRM ID to the created profile (because it doesn't know how to process the unknown entity)
- Apsis does a full sync
- Contact from form submission appears as a **duplicate** in Apsis — one unconnected from CRM integration, one from full sync

[Erik Andersson]: > "if they do, we will start losing their form submissions... Or rather we will create duplicates... the form submission which is completely disconnected from the integration because we have no CRM ID and then you do a full sync and then the contact that they created from the form submission will be added to our key space, but that is completely separate from the submitted one from the form and now you have two profiles in UPS that are technically the same ones."

### Database Cleanup

Beyond configuration changes, there is also **database cleanup** needed:
- Remove keyspace entries for leads in Tribe
- Potentially hide Tribe lead attributes from customer accounts (though this would be an Audience module operation, not Integrations)

If needed, could use internal Audience APIs via delegation tokens to hide attributes, or flip a database switch.

[Erik Andersson]: > "the most important one is just like doing what I did here, like remove the lead entity configuration for Tribe and then I think 90% of the work is done."

### Priority Assessment

[Lukasz Grabowski]: Asked whether this should be prioritized.

[Erik Andersson]: > "If if we there we get the go ahead, I think you should just like you should just do it because it should be. You should be able to do it in a day and it will remove so much complexity for the customers"

If Tribe confirms, this should be done immediately — high return on one-day investment. **Blocked on Tribe confirmation** at this time.

---

## Tribe Connector: General Architecture Issues

### Tribe's Organizational Independence

Tribe operates as a relatively isolated system within FSC:

[Erik Andersson]: > "Tribe... likes go in their own way with things. They don't want to integrate with other systems"

**Examples**:
- Refused to integrate with Maxo's central customer database — Tribe has its own separate customer set in Maxo
- Maintains separate customer service system (all Tribe cases must be registered in Tribe's system, not Maxo)
- Developing their own marketing module to reduce dependency on Apsis

### Connector Stability

[Erik Andersson]: "tribe... it is by far the most not stable connector of them all, not so much maybe on app C side, but the virtual entity setup certainly doesn't help for it."

However, he notes: "it is generic connector, so it's the same way in the same way as all of them" — the instability is partly from Tribe's API behavior, not from Apsis architecture alone.

### Working with Tribe Development

[Michal Rosikiewicz] asked: "Did we ever worked together with developers from Tripe on a code?"

[Erik Andersson]: Working relationship exists for integration discussions, but Tribe has its own development roadmap and communication is slow. Currently cannot get timely responses needed to confirm lead entity behavior.

### Future Considerations

The generic connector for Tribe "kind of needs a revamp on both sides" given increased customer adoption, but this is lower priority than the three major issues documented above.

---

## Debugging & Investigation Tools: Internal Broker Service

### What Is The Internal Broker Service?

A utility service that allows engineers to construct and test requests to customer CRM systems without needing direct access (which is rare) or decrypted credentials.

**How it works**:
1. Build a request endpoint (GET, POST, etc.) in Postman or similar
2. Take the customer's CRM system URL from the database
3. Send request to the **internal broker service** instead of directly to CRM
4. Broker service adds correct authorization headers and routes the request
5. Returns the CRM's response

### Use Case Example

To debug the Tribe pagination issue:

1. Manually constructed requests: `GET /contacts?page=0`, `?page=1`, `?page=2`, etc.
2. Sent through internal broker service
3. Combined and compared results to find duplicates
4. This would have been nearly impossible without broker service, as direct CRM access is unavailable

[Erik Andersson]: > "this what I call internal broker service is a very important tool to remember that we have because it's quite often that we need to make like debug requests to the CRM system."

### Importance for Future Debugging

[Erik Andersson]: "things like that is it's very hard to find fast. But I mean the more, the more you work with this, like the more things you will, the more you will remember and find fast."

This tool will be essential for investigating issues with large CRM systems and complex multi-page pagination scenarios.

---

## Work Planning & Prioritization Summary

### Priority Ranking Agreed Upon

1. **Large Customer Handling (Size + Memory Optimization)** — HIGHEST PRIORITY
   - **Why**: Blocks ability to handle existing 8M+ contact accounts; business risk if we lose deals
   - **When**: Q2 sprint, begin at start of quarter
   - **Effort**: Medium to large, requires CRM system coordination
   - **Status**: Epic documented, needs story breakdown and estimation

2. **Extended Sync Conditions** — MEDIUM PRIORITY
   - **Why**: Nice-to-have feature; current system works since integration release
   - **When**: Q2 or later (not Q1)
   - **Effort**: Large due to UI rebuild and CRM endpoint coordination
   - **Status**: Backend done, frontend design needed

3. **Improved Full Sync Report UI** — LOW PRIORITY, LOW EFFORT
   - **Why**: Improves UX and reduces support load; backend already done
   - **When**: Q1 if frontend capacity available; otherwise later
   - **Effort**: Small — likely 1–2 days of work
   - **Status**: Ready to implement

4. **Tribe Lead Entity Removal** — BLOCKED, WAITING ON CONFIRMATION
   - **Why**: Removes customer confusion; significant return on one-day investment
   - **When**: As soon as Tribe confirms behavior
   - **Effort**: 1 day
   - **Status**: Story to be written; waiting for Tribe response (3 weeks pending)

5. **Tribe Pagination Issue** — AWARENESS & MONITORING
   - **Why**: Known issue with Tribe's API; in their ballpark to fix
   - **Effort**: None on Apsis side
   - **Status**: Ball with Tribe; monitor for escalation

### Next Steps

[Lukasz Grabowski]: "let's look at the the condition thing sync tomorrow. So yeah, let's try to write down the solution. I mean stories because in the the epic is described so... Just look at the code and see together and estimate it."

- **Tomorrow's meeting**: Write stories for extended sync conditions epic, review proposed approach, estimate work
- **Tribe lead removal**: Erik will document in a story for immediate execution once Tribe confirms

---

## Key Takeaways

1. **Three major features were started but not finished due to frontend resource constraints** (Josh's departure):
   - Full sync report improvements (backend done)
   - Extended sync conditions with AND/OR logic (backend done)
   - These are good candidates for new frontend developers

2. **Large customer handling is the most critical issue** — prevents Apsis from supporting 8M+ contact deployments. Requires both Apsis architectural changes (remove in-memory consent caching, stream to SQS) and CRM-side sync condition filtering. Schedule for Q2.

3. **Tribe connector has multiple issues**:
   - Pagination API returns duplicates (Tribe's problem to fix)
   - Incorrect lead entity configuration causes confusion (can fix in 1 day once confirmed)
   - Tribe operates as isolated system with slow communication

4. **Internal broker service is essential debugging tool** for investigating CRM integration issues without direct access.

5. **Sync conditions cannot be extended until CRM systems are ready**:
   - Site Shop needs V2 endpoint support for complex AND/OR
   - Enterprise (FSC) needs to handle conditions in query parameters
   - Tribe doesn't currently support conditions at all

6. **Frontend should prioritize** (in order): large customer handling performance → full sync report UI → extended sync conditions. Tribe lead removal depends on external confirmation.

---

## Unresolved Questions & Action Items

### Blocking Items

- **Tribe Lead Entity**: Awaiting confirmation from Tribe team (3+ weeks pending) that they always return contact entity type regardless of dropdown selection. [Erik Andersson] to write a story documenting this; can be executed immediately upon confirmation.

- **Extended Sync Conditions Implementation Approach**: Erik has a proposed solution but unclear if it's best approach. **Tomorrow's meeting**: Review code together, discuss design, and estimate effort.

### Items for Next Meeting

- **Q2 Sprint Planning**: Schedule detailed work on large customer handling epic (data push to CRM systems, memory optimization, SQS streaming)
- **Sync Conditions Story Writing**: Create detailed stories for frontend condition builder UI
- **Tribe Escalation**: If no response within X days, escalate to Tribe management for lead entity confirmation

### Information Requests

- Recording or follow-up session on **internal broker service** technique for debugging — referenced as valuable for future CRM investigations