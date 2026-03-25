---
source_file: Erik - Handling Sync Conditions in Efficy Enterprise.txt
domain: Apsis One Integrations
topics: [Sync Conditions Architecture, Performance Issues with Large Contact Databases, Full Sync Optimization, Consent Data Handling, Stateless API Design, Pagination with Filters]
speakers: [Erik Andersson, Raziel Carvajal, Lukasz Grabowski]
key_components: [Sync Conditions, Full Sync, Real-time Webhooks, Contact Profiles, Consent Data, Efficy Enterprise, Microsoft Dynamics, Query Parameters, Pagination]
session_type: architecture-review
subdomains: [Architecture, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Outbound Flow]
---

## Session Overview

Erik Andersson walks through the critical performance problem with **sync conditions** in Efficy Enterprise integrations when handling extremely large contact databases (8 million+ contacts). The core issue is that today, sync conditions are evaluated inside APSIS after downloading all contact and consent data from the CRM system, leading to massive data transfer, memory exhaustion, and sync timeouts. The session explores two potential solutions: (1) passing sync conditions as configuration to the CRM system for server-side filtering, or (2) adding sync conditions as query parameters to the get_records API calls to enable stateless filtering. Raziel Carvajal and Lukasz Grabowski discuss the technical trade-offs, particularly around maintaining stateless API design versus configuration-based approaches.

---

## The Sync Conditions Problem: Current Architecture and Scale Challenges

### What Are Sync Conditions

[Erik Andersson]: Sync conditions regulate which contacts from the CRM system (in the enterprise case) are actually synced to APSIS. They act as a filter—for example, a customer might want to sync only contacts where `birthplace == Sweden` to run region-specific marketing campaigns.

These conditions are stored inside APSIS and evaluated on every sync:
- **Full sync**: Downloaded contacts are evaluated against the stored conditions
- **Real-time sync**: Incoming webhooks are evaluated as they arrive

If a contact matches the sync conditions, it's synced to APSIS. If not, it's discarded.

### Original Design Rationale

[Erik Andersson]: This architecture was chosen from the start because we were integrating with CRM systems over which we had no control. We couldn't add filtering logic in the CRM itself, so all filtering had to happen in APSIS.

> This has worked quite good up until quite recently.

### The Problem Manifests at Scale: The 8 Million Contact Case

A customer in France with approximately **8 million+ contacts** in their CRM system (far exceeding documented limits) wants to install **22 installations in APSIS** to handle different sections by organization, group, or region.

**The math of the problem:**
- CRM has 8 million contacts
- Expected to sync: ~100,000 contacts (those matching sync conditions)
- **Must download**: All 8 million contacts just to evaluate conditions
- **Discard**: 7.9 million contacts

**With consent data:**
- Typically at least 1 consent per contact
- Must download: ~8 million consent records (assuming 1 per contact)
- Actual estimate: ~16 million consent records with 2 consents per contact

**Multiply by 22 installations:**
- Total data transfer: roughly 176 million+ records in a single full sync cycle

[Erik Andersson]: This is an obscene amount of data to sync between the systems.

### Real-World Impact: Sync Failures and Timeouts

**Contact sync:** The customer's initial contact download completed successfully (the contacts alone didn't crash the system).

**Consent sync:** Complete failure. An optimization in APSIS that would normally help actually renders the sync useless—it crashes due to memory issues.

**Even without the optimization:** The Efficy Enterprise instance timed out after 1.5 days of sync before terminating with a gateway timeout. There may be a proxy in the request chain that terminates requests even earlier than APSIS's business logic timeout.

[Erik Andersson]: This consent sync did not feel well for that customer, and now do this 22 times. Even if it does go through, the sync takes approximately one day for a single sync run. I don't think this is anyone's fault because we haven't designed it for this quantity. But this way of doing it is gonna be an issue going forward.

### Known Issues with Smaller Databases

[Raziel Carvajal]: Even with 1 million contacts, this has been an issue. The sync conditions sometimes cause enough load on the CRM that it triggers disconnections. Clients report "the CRM is slow" when actually a sync condition evaluation is taking place. We haven't had many other clients complaining, but this remains a known issue.

---

## Solution Approach 1: Server-Side Filtering via Configuration (Microsoft Dynamics Example)

### How It Works for Microsoft Dynamics

[Erik Andersson]: For Microsoft Dynamics (referred to as "dynamics by site shop"), we've already implemented a solution. While sync conditions are still configured inside APSIS to keep everything in one place, we also send those conditions to the CRM system.

The CRM stores the conditions and:
- Either has a cache ready, or
- Generates the filtered results on-demand during a full sync
- (Internal implementation details are unknown; could be either approach)

**Result:** When we request a full sync, the CRM only returns contacts matching the sync condition.

**Performance improvement:**
- Instead of downloading 8 million contacts, download only 100,000
- Sync completes in 5–10 minutes instead of 1+ day
- Same applies to webhooks: only send webhooks for profiles matching the conditions

### Advantages of This Approach

1. **Immediate win on data transfer**: Only download what you need
2. **Consistency**: Same approach works for both webhooks and full syncs
3. **APSIS simplicity**: We just flip a switch; we don't manage the filtering logic

### Disadvantages and Trade-offs

[Erik Andersson]: Unfortunately, this approach requires work on the CRM side. We have two options:
1. Send the sync conditions along with each request (real-time evaluation)
2. Store them in the CRM's configuration (cache-based)

You would either provide the conditions to Enterprise when we request data, or configure them once in your system.

---

## Solution Approach 2: Stateless Filtering via Query Parameters

### The Stateless Design Principle

[Raziel Carvajal]: In Enterprise, we've historically kept all API operations stateless. We don't store any state on the Enterprise side. When we receive a request, we process it and return the answer without keeping cache. This avoids concurrency issues and inconsistencies.

The question is: **Can we apply sync conditions as query parameters and maintain stateless behavior?**

### How It Would Work

Instead of storing conditions in Enterprise's configuration, APSIS would pass them as query parameters in the API request:

```
GET /get_records?syncCondition=birthplace:Sweden&syncCondition=status:active&pageSize=100&pageNumber=0
```

**Process flow:**
1. APSIS sends the full sync request with sync conditions as query parameters
2. Enterprise evaluates the conditions in SQL (or equivalent) on each request
3. Returns only matching contacts for that page
4. APSIS requests the next page with the same conditions
5. When a page returns fewer results than page size, the sync knows it's reached the end

### Pagination Handling

[Erik Andersson]: We need pagination because we can't fetch all 100,000 contacts in one request. We typically download pages in parallel (multiple threads requesting pages 0, 1, 2, 3 simultaneously).

If any page comes back empty, we know we've gone through all pages and stop requesting.

[Raziel Carvajal]: The advantage here: **We still reduce the number of results per request**, which improves response time. However, **the number of total requests stays the same** because we still need to paginate through the filtered results.

### Example Calculation

If the CRM has 200 contacts, and we filter for `name='Eric'`:
- With configuration/cache: Enterprise pre-generates a view of matching contacts; pagination happens on the filtered set
- With stateless query parameters: Enterprise still pages through the filtered results, but has to re-apply the filter on every request

[Raziel Carvajal]: The difference is:
- **Configuration approach**: Fewer total requests + faster response per request = radical improvement
- **Stateless approach**: Same number of total requests + faster response per request = moderate improvement

### Why Stateless Alone Isn't Ideal

[Raziel Carvajal]: If we keep it purely stateless with query parameters, we need to specify the page number each time. But Enterprise doesn't know how many pages exist until it runs the query. So:
- Request 1: `pageNumber=0` → returns page 0
- Request 2: `pageNumber=1` → must re-run the same filter to get page 1
- Request N: `pageNumber=N` → must re-run the same filter again

This is inefficient but **technically works** if you're willing to accept the extra CPU cost of re-evaluating the filter for each page request.

### Trade-off Discussion

[Raziel Carvajal]: I like the idea of keeping it stateless because it's a general API principle. But you're right—the implementation and efficiency are different.

[Erik Andersson]: The key insight is: if we just add the conditions to the query parameters and keep pagination, we should still see a meaningful improvement in response time per request. We wouldn't need a cache or configuration at all.

---

## Sync Conditions Format and Current Limitations

### Current Condition Syntax

Today, sync conditions support **AND logic only** (`condition1 AND condition2 AND condition3`):

```
birthplace = Sweden
address = Stockholm  
status = active
```

All conditions must be true for a contact to sync.

### Missing: OR Logic and Complex Expressions

[Erik Andersson]: We did implement a new version of sync conditions that supports OR logic and union of intersections (e.g., `(condition1 OR condition2) AND (condition3 OR condition4)`), but this never came to life.

> Every front end resource disappeared on us, then the reorg happened. So this feature isn't in place for any customer.

[Raziel Carvajal]: As long as it's clear in the API specification that everything is AND, it's fine with us. We can implement accordingly.

### Field Mapping: APSIS to Enterprise Fields

When we send conditions to Enterprise, we provide:
- The Enterprise field name
- The value it should match

For example:
```
Enterprise Field: "birthplace_country"
Value: "SE"
```

[Erik Andersson]: In the enterprise case, we would only send sync conditions for the `contact` entity, even though the contract supports multiple entities.

### Limit on Number of Conditions

[Erik Andersson]: If memory serves, we have a limit of 10 sync conditions. Typically customers only have 2–3, so URL length shouldn't be an issue even if we pass them as query parameters.

---

## Handling Consents: A Parallel Problem

### The Consent Sync Issue

Consent data has the same scaling problem as contacts:
- 8 million contacts × (at least 1–2 consents each) = 8–16 million consent records to download
- We must currently download **all** consents, then filter by contact
- Multiply by 22 installations = 176–352 million consent records

### Current Approach: No Consent-Level Sync Conditions

[Raziel Carvajal]: Do we have synchronization conditions for consents? No.

[Erik Andersson]: Not today. But if we're implementing sync conditions for contacts, we should probably apply the same approach to consents.

### Why Consent Filtering Is Different from Contact Filtering

[Raziel Carvajal]: Webhooks for consent changes are typically small because the number of changes happening in a time window is narrow. We trigger webhooks periodically, and the payload is minimal.

The problem isn't webhooks—it's **full sync**, where we download all historical consent data.

**Consent webhook scenario:** A client updates consent for one contact → small webhook payload → APSIS can still filter it locally

**Consent full sync scenario:** We request all 16 million consent records → massive data transfer → APSIS runs out of memory

### Proposed Solution for Consents

[Erik Andersson]: We would introduce sync conditions for the consent endpoint the same way we do for records. The get_consents request would accept the same sync condition query parameters, and Enterprise would return only consents for contacts matching those conditions.

Instead of downloading 8 million × 2 = 16 million consent records, we'd download 100,000 × 2 = 200,000 consent records (per installation). Multiply by 22 installations = 4.4 million instead of 352 million.

---

## Implementation Details: Query Parameter Format and API Contract

### What APSIS Needs to Send

[Erik Andersson]: We would provide in the `get_records` request:
- A list of sync conditions (first name equals X, country equals Y, etc.)
- The fields would be Enterprise field names
- Page size
- Page number
- Optionally, a list of specific fields to return (we already do this to reduce data)

### The API Contract Challenge

[Raziel Carvajal]: When you have an API, you update it, give it a number, define the format, provide examples. If we have questions, we ask. But I don't see why we need to debate the name or format now. Just let us know when the API is written, and we'll review and implement.

[Erik Andersson]: Fair. The three of us (APSIS side, Efficy Enterprise side) would need to agree on:
- Query parameter names
- Format of the conditions
- Whether conditions are comma-separated, JSON, or something else
- Exact pagination semantics

But that's implementation detail work.

### Potential URL Length Concern

[Erik Andersson]: With a limit of 10 sync conditions and customers typically having 2–3, URL length shouldn't be an issue. Modern web servers handle reasonable query string lengths.

---

## Caching and State Considerations

### Why Caching Isn't Ideal for Enterprise

[Raziel Carvajal]: If we introduce caching on the Enterprise side, we have a problem: concurrency. If multiple APSIS installations request data simultaneously with the same sync conditions, we need to ensure the cache is up-to-date and consistent. That's complex.

The stateless approach avoids this entirely.

### Why Configuration Works for Some Systems

[Erik Andersson]: For Microsoft Dynamics, they've chosen to keep state (the cached/pre-generated filtered data). This works for them because they can manage consistency. But not all systems want that responsibility.

For Enterprise, it sounds like you prefer stateless, which is reasonable.

---

## Performance Implications and Urgency

### Different Scenarios, Different Urgency

[Raziel Carvajal]: From a user perspective, if synchronization happens at night and takes the whole night, the customer gets their synced data by morning. Not really urgent.

If a customer does a sync in the middle of the working day and it locks up the CRM, that's urgent because everything slows down.

**Real-world context:** If a customer has scheduled their syncs for off-peak hours, the performance hit is annoying but not critical. If they're doing adhoc syncs during business hours, it's blocking.

### Current Customer Impact

[Erik Andersson]: We have **3 customers currently suffering** from this problem. This is why I've already escalated it to:
- APSIS product team
- Efficy Enterprise team (this session)
- This broader discussion

The question is: **who prioritizes this?**

[Lukasz Grabowski]: It's important, but let me jump into a production issue first (the MA incident). I can write an epic based on this recording to make sure I capture the requirements.

[Erik Andersson]: I would **highly recommend doing this sooner rather than later**, especially if customer CRMs are already affected today.

---

## Proposed Next Steps and Action Items

### APSIS Responsibilities

1. Write a clear API specification for how sync conditions should be passed as query parameters
2. Define the exact format (naming, structure, examples)
3. Submit to Enterprise for review

### Efficy Enterprise Responsibilities

1. Review the API specification when provided
2. Write an epic documenting the requirements (Lukasz will do this)
3. Assess implementation effort and complexity
4. Prioritize against other work

### Open Questions

[Raziel Carvajal]: For the stateless approach with query parameters:
- Should we pass pagination info (page number, page size)?
- Should APSIS tell us how many results to expect per page?
- How do we handle the case where filtered results are large but we still need pagination?

[Erik Andersson]: I think pagination with the sync condition as a query parameter should work. When results come back empty or smaller than page size, we know we're done.

[Raziel Carvajal]: The difference in efficiency between stateless (query params) and configuration (pre-cache) is worth understanding before final implementation. We can discuss after APSIS provides the API spec.

### Timeline Considerations

- **Blocking issue**: Microsoft Dynamics solution already exists; Efficy Enterprise needs similar capability
- **Affected customers**: At least 3 known customers with large databases
- **Estimated effort**: Lukasz to clarify after writing epic
- **Priority**: Escalated; waiting for team decision

---

## Key Takeaways

1. **Sync conditions architecture has a fundamental scaling problem**: Evaluating conditions in APSIS after downloading all data is untenable for databases with 8M+ contacts.

2. **Two viable solution paths exist:**
   - **Configuration/Cache approach** (Microsoft Dynamics model): Store conditions in CRM, return pre-filtered results. Reduces requests and response time. Requires CRM to maintain state.
   - **Stateless query parameter approach**: Pass conditions as API parameters, re-evaluate per page. Maintains stateless API design. Reduces response time per request, but keeps total requests constant.

3. **Consent data has the same problem**: Must also apply sync conditions to the consent endpoint to avoid downloading 16M+ consent records on full sync.

4. **Current limitations**: Today's sync conditions support AND logic only; OR logic was designed but never implemented due to resourcing.

5. **Implementation is straightforward on APSIS side**: We already have sync conditions defined; we just need to pass them along in the API request instead of storing them in APSIS.

6. **Enterprise must agree on API contract**: The three teams need to align on query parameter names, format, and pagination semantics before Enterprise can implement.

7. **Urgency is moderate but real**: 3 customers affected; not critical if syncs are scheduled off-peak, but blocks business if done during work hours.

8. **Stateless design matters to Enterprise**: Raziel prefers avoiding cached state per API design principles, which influences whether configuration or query parameters are used.

---

## Unresolved Questions and Action Items

### Action Items

1. **Lukasz Grabowski**: Write an epic documenting the sync conditions scaling problem and proposed solutions based on this recording. Include:
   - Current behavior (download 8M contacts to sync 100k)
   - Problem statement (memory exhaustion, timeouts)
   - Proposed solution (stateless query parameters for get_records and get_consents)
   - Affected customer count (3 known)

2. **Erik Andersson / APSIS**: Draft API specification for sync conditions as query parameters:
   - Query parameter names (e.g., `syncCondition`, `filterCondition`, etc.)
   - Format (JSON, comma-separated key:value, etc.)
   - Example requests
   - Pagination semantics when combined with filters

3. **Raziel Carvajal / Enterprise**: Review specification when provided; identify any concerns about stateless implementation or efficiency.

4. **All**: Decide on priority—should this be done before Erik leaves the team (timeline unknown) or scheduled as a roadmap item?

### Open Questions

- **Configuration vs. Stateless trade-off**: Is Enterprise willing to accept slightly lower performance (multiple requests re-evaluating the filter) to maintain stateless design? Or should we explore configuration-based caching?
- **Consent conditions scope**: Should sync conditions for consents be identical to contact conditions, or are there consent-specific variations needed?
- **OR logic revival**: Should we revisit implementing OR logic and complex expressions for sync conditions while doing this work, or keep it simple (AND only) for now?
- **Backwards compatibility**: If we add sync condition parameters to get_records, do we need to support the old behavior (no parameters) for existing installations?