---
source_file: Integration projects never finished due to FE resources.txt
domain: Apsis One - Integrations
topics: [Full Sync Reporting, Sync Conditions, CRM Integration Performance, Tribe Connector Issues, Pagination Bugs, Large Customer Handling, Profile Updates, Consent Management]
speakers: ["Erik Andersson", "Lukasz Grabowski", "Tomasz Kowalski", "Michal Rosikiewicz"]
key_components: ["Full Sync Report", "Sync Conditions", "Generic Connector", "Tribe CRM", "Microsoft Dynamics", "FSC Enterprise", "SQS", "Profile Update Messages", "Consent Messages", "Internal Broker Service"]
session_type: knowledge-transfer
---

## Session Overview

This knowledge transfer session covers critical incomplete integration projects within Apsis One that were deprioritized due to frontend resource constraints. Erik Andersson walks the team through three major backend implementations that lack frontend completion, an urgent performance issue affecting large customers (2M+ contacts), pagination bugs discovered in the Tribe connector, and confusion around the Tribe "lead entity" that doesn't actually exist in the source system. The session emphasizes architectural decisions made and their downstream implications, particularly around how sync conditions must be evaluated differently across CRM systems.

---

## Full Sync Report Improvements

### Current Problem with Sync Statistics

[Erik Andersson] The existing full sync report groups consent and profile update messages together, creating confusion for customers. When a report shows "successful 39,000 customers," this number actually represents both profile updates AND consent messages combined—not 39,000 unique contacts. A customer with 8,000-10,000 profiles might see 39,000 in the report and have no idea why they're syncing so many profiles.

This confusion also affects support teams: NPS managers and account managers routinely misinterpret the metrics.

### Backend Implementation Complete, Frontend Missing

[Erik Andersson] The backend team has already separated these statistics in the data model. The sync report now tracks:
- Amount of profile messages (created, successful, failed, skipped)
- Amount of consent messages (created, successful, failed, skipped)
- Backward compatibility fields (total, successful) still present for existing integrations

The frontend still displays the old grouped format because **Josh left the company** and integration stores were deprioritized in favor of email and audience products. The change is "not in place for the front end, but the work is done in back end."

### Why This Matters

> This should be a very low hanging fruit if you have someone that is swift at front end work and it will add a lot of value for the customer. If nothing else, it will reduce the support questions you get for the full syncs.

[Erik Andersson] This is rated as low priority but low effort—an ideal task if frontend developers have capacity.

### Testing Status

The implementation has been tested during backend releases but lacks comprehensive real-world validation due to limited testing capacity.

---

## Extended Sync Conditions: Operators and Logic Support

### Current Limitations

[Erik Andersson] The sync conditions feature allows filtering which contacts/profiles to sync to Apsis based on field values. Currently, it has two critical weaknesses:

1. **Implicit AND logic only**: All conditions are combined with AND. You can say "email AND mobile number AND date of birth" but cannot express OR relationships.
2. **Equals operator only**: Only supports exact matches (`email equals this`). Customers frequently request NOT EQUALS, CONTAINS, STARTS WITH, ENDS WITH, and other operators available in the segment builder.

### Backend Solution Implemented

[Erik Andersson] The backend now supports:
- **Operator expansion**: equals, not equals, contains, starts with, ends with, and more
- **Logic expansion**: AND and OR operators (in addition to implicit AND)
- **Segment builder parity**: Anything possible in the segment builder should work in sync conditions

> What we wanted to do with this was essentially like anything you could do in the segment builder you should be able to do in the sync conditions.

The work is implemented and tested in the backend (by Prem), but the frontend has not been updated because it requires "quite radical" changes to the UI/UX layer.

### CRM System-Specific Implementation: Evaluation Location Matters

[Erik Andersson] A critical complication emerged during implementation: **where** sync conditions are evaluated changes the architecture significantly.

#### Current Structure (All CRM Systems)
The general connector sends sync conditions to the CRM system via an endpoint:
```
- entity_type (e.g., contact, lead)
- conditions: implicit AND structure
  Example: "first_name = 'Elsa' AND sync_to_apsis = true"
```

#### Version 2 Structure (Tribe/Site Shop Only)
A new endpoint supports the more complex structure with AND/OR logic:
```
- entity_type
- conditions: complex nested structure with AND/OR operators
```

**Critical caveat**: Site Shop currently supports V2 sync conditions evaluation. **Microsoft Dynamics and FSC Enterprise do not yet support V2.** Before enabling V2 for any account, Site Shop (and eventually Enterprise) must verify they can parse and evaluate this new condition structure.

#### The Two Evaluation Approaches

**Approach 1 (Site Shop)**: Store conditions as configuration, CRM system evaluates them internally or uses them to generate API paginated requests.

**Approach 2 (FSC Enterprise)**: Stateless API; Apsis must include sync conditions in the query parameter or request body when asking for contacts during full syncs and consent retrieval.

[Erik Andersson] Most of this work is on the CRM connector side, but Apsis must implement the ability to take sync conditions and add them to query parameters or request bodies.

### Architecture Decision: Defer This Feature

> I would probably suggest like don't rush with this feature, finish off all other on the list first because it does have some consequences in the other CRM systems as well.

[Lukasz Grabowski] Agreed—rated as "nice to have" but lower priority. The current implicit AND structure has worked since integration release and can wait.

---

## Critical Performance Issue: Large Customer Handling (8M+ Contacts)

### The Problem

[Erik Andersson] For customers with 2M+ contacts (observed: 8M contact CRM), the current architecture causes massive inefficiency:

**Current Behavior**: Sync conditions are evaluated inside Apsis, requiring all contacts to be downloaded from the CRM system first. If a customer has 8 million contacts but only wants 100,000 of them, Apsis must download all 8 million, then filter.

**Impact**: Extreme data waste, bandwidth waste, and time waste for customers. Scalability threat for Apsis (risk of losing large deals).

### Solution Requires CRM System Participation

[Erik Andersson] Apsis cannot solve this alone. CRM systems must handle sync condition evaluation at retrieval time:

1. **Push conditions to CRM**: Store conditions as config; CRM uses them to filter paginated results (like Site Shop).
2. **Include conditions in request**: Send sync conditions as parameters in API calls; CRM applies filtering before returning results (like FSC Enterprise's stateless API).

### Apsis-Side Work Required

Within the full sync process, Apsis must:
- Implement logic to include sync conditions when requesting contacts
- Add sync conditions to query parameters or request body (to be agreed in grooming meeting)
- Handle the response containing only filtered contacts

### Memory Optimization: Remove Profile Comparison Cache

[Erik Andersson] A secondary optimization issue must be addressed: the full sync currently stores all consent export results in memory, comparing incoming consent status against what's already in Apsis. For large loads, this crashes the system.

**Solution**: Remove this optimization. Instead of storing in-memory and comparing:
- Handle each message individually
- Stream profile updates directly to SQS
- Let the consumer process messages as they arrive

> There is no issue with a lot of profile updates being in SQS, there's a lot of problem having them in memory in the producer.

**Customer clarity benefit**: This also clarifies what's happening. If a customer runs a full sync processing 10 consent messages, the next full sync won't reprocess those same 10 if nothing changed. Currently, the customer sees "suddenly 10 less messages" with no explanation—confusing without the context.

### Priority Assessment

[Lukasz Grabowski] This is "the most important" epic and should be included in Q2 planning (Q1 scope is closed).

[Erik Andersson] Agrees—the sync conditions can wait "another half year or year," but customer size is a "big problem" now.

---

## Tribe Connector: Pagination Duplicates Bug

### Discovery Process

[Erik Andersson] A customer reported a discrepancy: "You downloaded 272 contacts from our CRM, but only 204 appear in Apsis. Why?"

**Investigation steps**:
1. Downloaded the list from Tribe directly via API—confirmed 272 contacts
2. Checked Apsis's last profile sync tracking (kept for tag/untag operations)—showed 204 entries
3. Replicated Apsis's pagination approach (50 contacts per page: pages 0, 1, 2, 3, 4, etc.)
4. **Finding**: Some CRM IDs appeared in the total count but not in the paginated result set—**Tribe was returning duplicate entries across pagination**

**Result**: Out of 272 reported contacts, 68 were duplicates—a 25% duplication rate that was "quite consistent."

### Root Cause and Apsis's Position

[Erik Andersson] The pagination issue appears to be on Tribe's side. Apsis happily accepts any IDs returned; duplicates don't cause functional issues for us because we just process them. However, this is a potential issue for customer data integrity and might affect Tribe's full sync operations.

[Lukasz Grabowski] The ball is in Tribe's court to investigate and fix their pagination implementation.

### Why Page Size Isn't a Solution

[Erik Andersson] Increasing page size doesn't solve the problem because Apsis doesn't know in advance how many profiles exist in a customer's CRM list. Pagination is necessary regardless of page size; the duplication issue persists.

### Implications for Troubleshooting

This is important context for support: if customers report discrepancies with Tribe connectors, pagination duplicates may be the culprit. The team should be aware this is an issue, even if Apsis can't fix it.

---

## Tribe Connector: Virtual Lead Entity Confusion

### The Architecture Problem

[Erik Andersson] In Tribe, outbound mappings require specifying which entity to create from form submissions (contact, lead, etc.). Tribe provides a dropdown list to select this.

However, the generic connector architecture needs static entity definitions at configuration time to:
- Know which entity to download attributes for
- Set up the mapping correctly
- Handle dynamic changes poorly (the connector prefers known, fixed entities)

This led to introduction of a "lead entity" configuration in Apsis roughly 1.5 years ago.

### The Discovery: Tribe Always Returns "Contact"

[Erik Andersson] Recently discovered: **No matter what entity type is selected in Tribe's dropdown, Tribe always responds to our API calls saying the entity is "contact."** Tribe handles the virtual entity logic internally—their API always returns contact type regardless of what you selected in their UI.

**Result**: The entire "lead entity" setup in Apsis is worthless and creates customer confusion.

### The Impact on Customers

[Erik Andersson] This is the single most asked question from Tribe customers: "What is the lead in Apsis? We don't have a lead in Tribe."

Explaining "virtual entity" to customers is not a fun conversation.

### The Solution (Blocked on Confirmation)

[Erik Andersson] The fix is simple but currently blocked:

**If Tribe confirms they always respond with contact**, Apsis should:
1. Remove all lead entity configuration from Tribe in the generic connector
2. Only handle contacts (remove the virtual lead entity setup)
3. Clean up related database entries (key spaces with leads)
4. Optionally hide Tribe lead attributes from customer accounts (via internal audience APIs or database switch)

**Expected effort**: 1 day to complete the core work (removing the lead entity configuration and verification).

[Erik Andersson] Has been waiting 3 weeks for confirmation from Tribe. Will write a story during the day to track this. If someone receives confirmation from Tribe, this can be executed after Erik's departure.

### Why Confirmation Is Mandatory

[Erik Andersson] Without confirmation, there's risk: if Tribe occasionally returns non-contact entities, removing the lead entity handling could cause:
- Form submissions to be processed as entities Apsis doesn't recognize
- Apsis unable to attach a CRM ID to the profile
- Full sync later creates a duplicate contact with the same data but no CRM linkage
- Two separate profiles in Apsis for the same person—one from form submission, one from full sync

> We will not do any merge. We will not add any CRM ID to it if it is an entity that we don't know how to process because then you will have the form submission which is completely disconnected from the integration.

---

## How Pagination Issues Are Debugged: The Internal Broker Service

### The Tool

[Erik Andersson] The team has access to an **internal broker service** that allows debugging CRM API calls without direct access to customer systems (credentials are decrypted from the database).

**Workflow**:
1. Determine the schema/structure needed (from generic connector collection or documentation)
2. Retrieve customer's CRM endpoint URL from the database
3. Build the API request in Postman (or similar)
4. Route the request through the internal broker service endpoints
5. The broker adds correct authorization headers and executes the request
6. View the raw response to inspect structure and behavior

[Erik Andersson] Used this approach to discover the Tribe pagination duplicates by requesting pages 0, 1, 2, 3, 4, 5, etc., and combining results to spot the duplicates.

### Importance and Frequency

[Erik Andersson] This is "a very important tool to remember that we have because it's quite often that we need to make like debug requests to the CRM system."

[Tomasz Kowalski] Asked how pagination was checked; Erik confirmed it was via the internal broker service (previous KT session covered this—recording available if needed).

### Learning Curve

[Erik Andersson] Finding and diagnosing issues like this becomes easier with experience using the tool, but it's invaluable for comparing "how does it look when I make it manually" vs. "how does it look inside Apsis?"

---

## Tribe Connector: Broader Strategic Issues

### System Isolation and Integration Challenges

[Erik Andersson] Tribe operates more like a "secluded system" within the FSC product family compared to other systems.

**Examples of Tribe's seclusion**:
- Refused to integrate with Maxo's central customer database
- Maintains separate customer records in Tribe vs. Maxo
- Tribe customer service cases must be registered in Tribe's system, not Maxo
- Currently building their own marketing module to reduce dependency on Apsis

[Erik Andersson] Despite these tensions, Apsis must maintain the Tribe integration because:
- Existing customers are integrated and need support
- New customers are being sold Tribe + Apsis bundled

### Current Communication Challenges

[Erik Andersson] Right now unable to get responses from Tribe's team on the lead entity confirmation, which is blocking progress on simplification.

### Connector Stability

[Erik Andersson] Tribe is "by far the most not stable connector of them all"—partly due to Apsis-side issues (the virtual lead entity setup doesn't help) but also partly due to Tribe's architecture. A broader connector revamp on both sides may be needed as Tribe gains more customers.

[Tomasz Kowalski] Noted that working with Tribe produces more issues than other integrations.

### No Direct Development Collaboration

[Michal Rosikiewicz] Asked if the teams have worked together on code level.

[Erik Andersson] No direct code collaboration, though Apsis worked extensively with Tribe when the integration was initially implemented. Tribe's preference for independent development limits this collaboration now.

---

## Prioritization and Next Steps

### Priority Ranking (Q1 Closed, Q2 Planning)

[Lukasz Grabowski] and [Erik Andersson] agreed on the following prioritization:

**1. Large Customer Handling (HIGH PRIORITY for Q2)**
- Handling 2M+ customer databases efficiently
- Includes removing the memory optimization in full sync
- Critical for retaining large deals and improving scalability
- Should start Q2 early

**2. Full Sync Report UI Updates (LOW PRIORITY, LOW EFFORT)**
- Separate profile and consent messages in the frontend display
- Easy task for an available frontend developer
- Non-blocking for functionality, improves customer clarity and reduces support questions
- Can be picked up when frontend capacity allows

**3. Extended Sync Conditions (LOWER PRIORITY, MEDIUM EFFORT)**
- Operator expansion (beyond equals) and AND/OR logic
- Requires testing and CRM system coordination (Site Shop, Enterprise, others)
- Current implicit AND structure has worked since integration release—can wait another 6-12 months
- Defer until other priorities complete

**4. Tribe Lead Entity Removal (BLOCKED, QUICK WIN IF UNBLOCKED)**
- 1-day effort if Tribe confirms they always return contact entity
- Reduces customer confusion and connector complexity
- Blocked pending 3-week-overdue confirmation from Tribe
- High priority IF confirmation received; can be done even after Erik's departure

### Grooming Meeting Tomorrow

[Erik Andersson] Tomorrow's grooming meeting will focus on writing stories for the extended sync conditions feature and estimating effort after reviewing the code.

Erik has already written a suggested approach for the implementation but isn't certain it's optimal—this will be discussed.

---

## Key Takeaways

1. **Frontend Resource Bottleneck**: Three major backend features completed but undeployed due to Josh's departure and competing email/audience priorities:
   - Full sync report UI (data model complete, frontend pending)
   - Extended sync conditions (backend complete with AND/OR support, frontend needs radical redesign)
   - Caused significant technical debt and customer confusion

2. **Large Customer Problem Is Urgent**: Current architecture cannot efficiently handle customers with 2M+ contacts. Solving this requires:
   - Participation from CRM systems (Site Shop, FSC Enterprise, others) to evaluate sync conditions at data retrieval time
   - Removal of memory optimization in full sync process (stream to SQS instead)
   - This is the highest priority for Q2 and critical for Apsis's enterprise strategy

3. **Tribe Pagination Is a Known Issue**: 25% duplication rate in paginated responses is on Tribe's side. Apsis processes duplicates without functional impact, but this is important diagnostic knowledge for support teams.

4. **Tribe Lead Entity Confusion Can Be Resolved Fast**: 1-day fix once Tribe confirms they always return contact type. This is the #1 customer question about Tribe integration. Blocked on 3-week-overdue confirmation.

5. **CRM System Evaluation Location Changes Everything**: Where sync conditions are evaluated (Apsis vs. CRM system) fundamentally changes the API design. V2 sync conditions work in Site Shop but not other systems yet. This is architectural coupling that must be managed carefully.

6. **Internal Broker Service Is Essential**: Debugging CRM integrations requires the internal broker service to route requests through proper authorization. This tool is how pagination bugs and other issues get discovered.

7. **Tribe Is Architecturally Independent**: Tribe operates as a "secluded system" within FSC (separate customer DB, separate case system, building independent marketing module). Integration maintenance is still necessary but collaboration is limited.

---

## Unresolved Questions & Blocked Items

1. **Tribe Lead Entity Confirmation** (BLOCKING): Has Tribe confirmed they always respond with contact entity type regardless of dropdown selection? Erik has been waiting 3 weeks for an answer. Status: A story will be written to track this. Cannot proceed with cleanup until confirmed.

2. **Extended Sync Conditions Approach** (TO DISCUSS): Erik has proposed an implementation approach but is unsure it's optimal. This will be discussed in tomorrow's grooming meeting.

3. **V2 Sync Conditions in FSC Enterprise** (TO COORDINATE): Enterprise customer accounts need support for V2 sync conditions (with AND/OR logic), but Enterprise must first verify they can parse the new structure. This will be part of tomorrow's grooming.

4. **Tribe Pagination Root Cause** (TRIBE'S RESPONSIBILITY): Investigation complete on Apsis side; the bug is on Tribe's pagination implementation. Tribe must investigate and fix. No action for Apsis until Tribe responds.

---

## Session Metadata

- **Duration**: 40m 19s
- **Date**: February 10, 2026, 8:33 AM
- **Key Attendees**: Erik Andersson (departing), Lukasz Grabowski (presumably taking over), Tomasz Kowalski, Michal Rosikiewicz
- **Follow-up**: Grooming meeting scheduled for next day to write stories and estimate extended sync conditions work