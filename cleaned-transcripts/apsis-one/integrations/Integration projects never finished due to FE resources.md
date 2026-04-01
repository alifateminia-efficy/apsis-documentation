---
source_file: "Integration projects never finished due to FE resources.txt"
domain: Apsis One Integrations
topics: [full-sync-report-improvements, sync-conditions-extension, large-customer-scalability, tribe-crm-pagination-bug, tribe-lead-entity-removal, internal-broker-service-debugging, prioritization-discussion]
speakers: ["Erik Andersson (outgoing developer/domain expert)", "Lukasz Grabowski (team lead)", "Tomasz Kowalski (developer)", "Michal Rosikiewicz (developer)"]
key_components: [Apsis One Integrations, Generic Connector, Full Sync, Sync Conditions, Tribe CRM, Microsoft Dynamics (SiteShop), FSC Enterprise, SQS, Internal Broker Service, Audience]
session_type: knowledge-transfer
---

## Session Overview

This is a knowledge transfer session led by Erik Andersson (outgoing domain expert) covering incomplete integration features and active bugs in the Apsis One Integrations platform. The session covers three unfinished features — improved full sync reporting, extended sync conditions, and large-customer scalability — all of which have completed back-end work but no front-end implementation due to resource deprioritization. Additionally, two Tribe CRM-specific issues are discussed: a pagination duplication bug and a misguided "lead entity" abstraction that should be removed. The session closes with a prioritization discussion and a note on using the internal broker service for CRM debugging.

---

## Improved Full Sync Report: Back-End Complete, Front-End Missing

### Background and Customer Confusion

The current full sync report groups **profile update messages** and **consent messages** together under a single "successful" count. Customers see a number like 39,000 and assume it means 39,000 contacts or profile updates. In reality, it's profile updates *plus* consent messages combined. This causes significant confusion for customers and also for NPSs and account managers.

### What Was Done in Back-End

The data model was changed to separate the statistics. Instead of just a total and a successful count, the sync record now stores:

- Amount of profile messages (created, successful, failed, skipped)
- Amount of consent messages (created, successful, failed, skipped)

This enables granular reporting. Backwards compatibility is maintained — the old `total` and `successful` fields are still present.

### Why Front-End Was Never Updated

> "Josh left the company and the integration stores were always deprioritized in favour of email and audience."

The front-end change was never made. The UI still shows the old combined view.

### Recommendation

[Erik Andersson]: This is a low-hanging fruit for a front-end developer. It will meaningfully reduce support questions about full syncs. However, it needs more thorough testing as it hasn't been fully validated beyond the back-end release check.

[Lukasz Grabowski]: Low priority, low effort — can be added to the backlog without urgency. Q1 scope is closed; could fit in Q2 if there is space.

---

## Extended Sync Conditions: Back-End Complete, Front-End Missing

### Current Limitations

The current sync condition UI has two significant weaknesses:

1. **All conditions are implicitly AND-combined** — there is no support for OR logic.
2. **Only "equals" operator is supported** — no "not empty", "contains", "starts with", "ends with", etc.

### What Was Built in Back-End

The back-end was extended to support the **segment builder format** for sync conditions, adding:

- AND / OR logical grouping
- Full operator set: equals, not equals, contains, starts with, ends with, and others matching the segment builder

[Erik Andersson]: Prem tested this. All back-end work is done. The front-end needs to be changed "quite radically" to support the new condition structure.

### Two API Versions Now Exist

There are now **two sets of operations** under the sync conditions endpoint:

- **V1 (current):** Uses only the section discriminator — implicit AND, equals only.
- **V2 (new):** Per-entity structure supporting AND/OR. Significantly more complex.

### CRM-Side Dependency: SiteShop

When sync conditions change, Apsis sends them to the CRM system via an endpoint in the **general connector**. SiteShop is the only CRM system that has been given the V2 endpoint (tailored for the new AND/OR structure).

> **Important caveat:** Before enabling V2 sync conditions for any account, this must be verified with SiteShop. Erik does not believe SiteShop has actually implemented support for the new structure, because the feature never went live inside Apsis.

### CRM-Side Dependency: FSC Enterprise

Me and Walker had discussions that FSC Enterprise will also need to support sync conditions, given the upcoming large-customer requirement (see next section). They need to be involved in V2 sync condition rollout.

### Prioritization

[Lukasz Grabowski]: Extended sync conditions is "nice to have" — good feature, but given the team's current situation, it is **lower priority** than the large-customer scalability work.

[Erik Andersson]: Agreed. Sync conditions have worked the same way since release. No urgency in waiting another six months to a year.

---

## Large Customer Scalability: Full Sync Performance for 2M+ Contacts

### The Core Problem

Sync conditions are currently evaluated **inside Apsis**. This means that for a full sync, Apsis must **download all contacts** from the CRM system before filtering.

Example: A customer with 8 million contacts where only 100,000 match the sync conditions. Apsis currently downloads all 8 million, which is:
- A massive data waste
- A massive time waste
- A hard blocker for serving large enterprise customers

### Two Approaches to Push Condition Evaluation to the CRM

**Approach 1 — Configuration-based (SiteShop model):** Apsis stores the sync conditions as configuration on the CRM side. The CRM evaluates the conditions and generates pages of matching contacts itself.

**Approach 2 — Stateless API with query parameters (FSC Enterprise model):** FSC Enterprise has a stateless API. For these customers, Apsis will need to include sync conditions as query parameters or in the request body when requesting contacts during full syncs and consent fetches.

Most of this work is on the CRM side, but there is also work required on the Apsis side: implementing the logic to take sync conditions and attach them to the outbound request (query parameter or request body — TBD, subject of an upcoming grooming meeting).

### In-Memory Optimization That Must Be Removed

There is a current **optimization in the full sync producer** that stores consent export results in memory. The full sync flow is:

1. Download all contacts from CRM → produce profile update messages
2. Trigger consent exports from Audience for each mapped subscription
3. Store consent results **in memory**
4. Compare incoming consent status against the in-memory Apsis state

**Problem:** With a very large contact set, this crashes the full sync due to out-of-memory errors.

**Fix:** Remove the in-memory optimization entirely and switch to a **streaming approach**:

- Take each profile update message → convert to correct format → push directly to **SQS**
- Let the SQS consumer process messages as they arrive
- Do not hold any contact/consent data in memory in the producer

**Side effect of removing the optimization (currently a UX confusion point):** With the optimization active, if you run a full sync and handle 10 consent messages, the *next* full sync won't re-process those 10 if nothing changed. From the customer's perspective, the processed count appears to drop unexpectedly with no visible reason. Removing the optimization makes behavior more transparent — every message is processed on every run.

> [Erik Andersson]: "There is a lot of problem having them in memory in the producer. There is no issue with a lot of profile updates being in SQS."

After this change, Apsis-side should be in good shape to handle very large customers.

### Prioritization

[Lukasz Grabowski]: **Highest priority of the three epics.** Should be planned for **Q2** (Q1 scope is closed). Work should start at the beginning of Q2.

[Erik Andersson]: Agreed. This is the most important one. The current workaround of "we can't handle customers this size" is becoming a big loss risk for Apsis given active 8-million-contact enterprise prospects.

---

## Tribe CRM: Pagination Duplication Bug

### Discovery

A customer reported that the sync report showed 272 contacts downloaded from Tribe CRM, but only 204 tags appeared in Apsis. Erik investigated:

1. Downloaded the list directly from the CRM system — confirmed 272 contacts.
2. Checked the last profile sync tracking record (Apsis tracks which contacts were processed for tagging/untagging) — also showed 204.
3. Downloaded contacts via pagination (50 per page, same method Apsis uses in profile syncs) — found duplicate CRM IDs across pages.

**Root cause: Tribe is returning duplicates in its pagination responses.** Out of 272 records, approximately 68 were duplicates. The duplication was consistent across checks.

### Impact

- Affects **profile list syncs** for certain Tribe customers now.
- **May also affect full syncs** — this has not been fully investigated.
- Apsis currently accepts IDs without deduplication, so duplicates silently disappear from processing.

### Workarounds Considered and Rejected

Increasing page size was considered but rejected: we don't know the maximum number of profiles a list might contain, and pagination will always be required regardless of page size.

### Ownership

> The ball is in Tribe's ballpark. They need to check and fix their pagination implementation. There is nothing Apsis can do about this fundamentally.

The team should be aware this issue exists and may receive customer questions about it.

---

## Tribe CRM: Lead Entity Removal (Awaiting Confirmation)

### Background — Why "Lead" Was Introduced

Approximately 1.5 years ago, a virtual **lead entity** was introduced in the Tribe integration inside Apsis. In the Tribe outbound mappings UI, there is a dropdown where users select which entity to create from form submissions. The generic connector requires knowing which entity to download attributes for in order to set up the mapping — it cannot handle fully dynamic entities.

### The Discovery: Lead Entity Is a Fiction

**Tribe always responds with `contact` regardless of what is selected in the dropdown.** Tribe handles the entity type internally. The lead entity configuration in Apsis is therefore:

- Completely useless for Tribe
- A source of significant customer confusion

> [Erik Andersson]: "This is the single most common question I've gotten regarding Tribe — 'what is the lead in Apsis? Because you don't have lead in Tribe.'"

### What Needs to Be Done

1. Remove the lead entity configuration in the Tribe integration (the entity configuration shown on the consent mapping configuration page)
2. Remove lead entity from the additional entities loading
3. Database cleanup: remove existing lead entity entries (e.g., key spaces)
4. Optionally: hide the Tribe-created lead attributes on customer accounts in Audience — this can be done via **delegation tokens** (each installation has one) calling internal Audience APIs, or via a direct database switch. Erik estimates this covers ~10% of the work; the entity config removal is ~90%.

### Risk If Done Without Confirmation

If Tribe is, in some edge case, returning non-contact entities and Apsis stops processing them:
- Form submissions will still create a profile in Apsis.
- Apsis will not add a CRM ID to that profile (unknown entity type).
- A subsequent full sync will add the same contact from Tribe under a different key space entry.
- Result: **two duplicate profiles in Apsis** representing the same person, permanently disconnected.

### Why It Hasn't Been Done Yet

[Erik Andersson]: "I want confirmation from tribe that I can actually go ahead and do this because I have previously explained to every consultant why we need to have this lead entity."

After 3 weeks, no response from the Tribe team. Erik will write a story/ticket during the day for handover. The actual code change is estimated at **one day of work** once confirmation is received.

### Tribe's Organizational Context

[Erik Andersson]: Tribe is a more **secluded system** than most others within FSC. They refused to integrate with the central customer database in **Maxo**, so there are separate customer sets in Tribe vs. Maxo, and customer support cases for Tribe go into Tribe's own system, not Maxo. Tribe is also currently building their own marketing module to reduce dependency on Apsis — but existing Apsis-Tribe customers still need to be maintained, and new customers are still being sold Tribe + Apsis bundles.

Communication channel with Tribe exists, but responsiveness is currently poor.

---

## Internal Broker Service: Debugging Tool for CRM Requests

### What It Is

There is an **internal broker service** that allows developers to construct and route requests to a customer's CRM system without needing direct credentials or access.

### Why It's Needed

CRM credentials are stored encrypted in the database. Developers very seldom have direct access to a customer's CRM system.

### How to Use It

1. Retrieve the customer's CRM system URL from the database.
2. Construct the request (endpoint, method, parameters) for whatever you want to check (e.g., get schema, paginate contacts, inspect a specific entity).
3. Send the request to the **internal broker service endpoint** — it will add the correct authorization headers and proxy the request to the CRM.

A Postman collection exists for the generic connector with standard requests. This collection is designed for use with the broker service.

### Practical Example

This was used to diagnose the Tribe pagination bug: Erik made requests for page 0, 1, 2, 3, 4, 5 manually via the broker, combined the results, and identified duplicate CRM IDs across pages.

> [Erik Andersson]: "This internal broker service is a very important tool to remember that we have because it's quite often that we need to make debug requests to the CRM system."

A recording of this workflow was made in a previous KT session. Erik offered to demonstrate it again if needed.

---

## Prioritization Summary

| Feature / Issue | Back-End Status | Front-End Status | Priority | Suggested Timing |
|---|---|---|---|---|
| Improved Full Sync Report (separated profile/consent counts) | ✅ Done | ❌ Not started | Low | Q2, if FE capacity allows |
| Extended Sync Conditions (AND/OR, operator set) | ✅ Done | ❌ Not started | Lower | After scalability work; CRM coordination required first |
| Large Customer Scalability (push sync conditions to CRM, remove in-memory optimization) | 🔄 Partially done (Apsis side work remaining) | N/A | **Highest** | Q2, start of quarter |
| Tribe Pagination Bug | N/A (Tribe's bug) | N/A | Awareness only | Tribe's responsibility to fix |
| Tribe Lead Entity Removal | ✅ Ready to implement | Minor validation | Medium (once confirmed) | When Tribe confirmation received — ~1 day of work |

---

## Key Takeaways

1. **All three major unfinished features have completed back-end work.** The blocker in each case was front-end deprioritization after a key developer (Josh) left the company.

2. **Large customer scalability is the most urgent.** The current architecture downloads all CRM contacts into memory before filtering, which is untenable for customers with millions of contacts. The fix requires both pushing sync condition evaluation to CRM systems and switching the producer to stream directly to SQS rather than holding data in memory.

3. **V2 Sync Conditions exist in back-end but must not be enabled without CRM-side verification.** SiteShop has the new endpoint but may not have implemented it. FSC Enterprise also needs to be involved. Do not enable V2 for any account without confirming CRM-side support.

4. **The Tribe lead entity is a 1.5-year-old mistake** based on a misunderstanding of how Tribe works. Removing it is ~1 day of work but requires written confirmation from Tribe before proceeding, due to the duplicate profile risk.

5. **Tribe's pagination returns duplicates** — a known bug on their side. Apsis cannot fix this; the team should be aware when handling customer support questions about Tribe list sync discrepancies.

6. **The internal broker service is essential for CRM debugging.** Any time you need to make manual requests to a customer's CRM system to understand its behavior, use the broker service rather than trying to obtain direct credentials.

---

## Unresolved Questions / Action Items

- **[Blocked — awaiting Tribe]** Confirmation from Tribe team that they always return `contact` entities (never `lead` or other) — required before lead entity removal can proceed. Erik has been waiting 3 weeks. He will write a ticket/story during the day.

- **[Erik — to hand over]** Written story/ticket for Tribe lead entity removal to be created before Erik's departure.

- **[Tomorrow — grooming meeting]** Detailed solution design and story writing for the large customer scalability epic (pushing sync conditions to CRM, SQS streaming approach). Erik has a written suggestion but acknowledges it may not be the best approach.

- **[SiteShop verification needed]** Before V2 sync conditions are enabled for any account, SiteShop must confirm they have implemented the new AND/OR structure in their endpoint.

- **[FSC Enterprise]** Enterprise team needs to be brought into the sync conditions V2 rollout — they will need to accept sync conditions as query parameters in their stateless API for full syncs and consent fetches.

- **⚠️ [Ambiguity]** It is unclear exactly what "most of this work is on the CRM side" means for the FSC Enterprise scalability approach — the split of work between Apsis and Enterprise was not fully detailed and is presumably the subject of tomorrow's grooming meeting.