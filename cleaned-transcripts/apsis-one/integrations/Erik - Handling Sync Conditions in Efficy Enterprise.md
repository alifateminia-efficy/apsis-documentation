---
source_file: Erik - Handling Sync Conditions in Efficy Enterprise.txt
domain: Apsis One Integrations
topics: [Sync conditions, Full sync performance, CRM data filtering, Pagination, Consent synchronization, Stateless API design, Large-scale contact syncing]
speakers: [Erik Andersson, Raziel Carvajal, Lukasz Grabowski]
key_components: [Efficy Enterprise CRM, Sync conditions engine, Full sync process, Webhook real-time sync, Consent records, Get records endpoint, Get consents endpoint]
session_type: architecture-review
subdomains: [Architecture, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration]
---

## Session Overview

This session discusses the critical performance problems that arise when syncing large CRM databases (millions of contacts) with Apsis One using **sync conditions**. The core issue is that Efficy Enterprise currently evaluates sync conditions after downloading all contacts from the source CRM, leading to massive data transfer, memory exhaustion, and multi-day sync times. Erik Andersson proposes moving condition evaluation to the CRM side (similar to the approach used with Microsoft Dynamics), while the team discusses stateless API design tradeoffs and pagination strategy. The session covers technical solutions, implementation approaches, and priority assessment.

---

## Current Sync Conditions Architecture and the Problem

### What Sync Conditions Do

**Sync conditions** regulate which contacts from a CRM system are actually synced to Apsis. They are configured in Apsis and evaluated against incoming contact data—both in full syncs and real-time webhook scenarios.

Example use case: A customer wants to sync only contacts with `birthplace == Sweden` because they're running a targeted marketing campaign for a specific region.

> "These sync conditions they regulate like which contacts in the if it's the enterprise case are actually synced to APSIS."

Current behavior: When a contact arrives (via full sync or real-time sync), Apsis compares the contact's data against stored conditions. If the contact matches, it's synced; if not, it's discarded.

### Why This Design Exists

The current approach was designed for CRM systems over which Apsis has no control. Since Apsis cannot modify CRM-side logic or queries, filtering had to happen in Apsis after data arrival.

[Erik Andersson]: "This is how it was designed from the start for because we were integrating with CRM systems over which we had no control over, so we could not um. Add any logic anywhere else but inside of APSIS."

---

## The Scale Problem: Real-World Impact

### The French Customer Case Study

A French customer with approximately **8 million contacts** in their Efficy Enterprise CRM wants to configure **22 separate installations in Apsis** (likely for different regions or business units). Under the current architecture:

1. **Initial attempt**: Full sync of all 8 million contacts succeeded, but processing consents failed
2. **With consents**: Each contact has at least one associated consent record, creating ~8 million additional records. Multiply by 22 installations = **176 million consent records downloaded**
3. **Memory crisis**: An optimization within Apsis that filters unmatched records caused the sync to crash due to memory exhaustion
4. **Timeout failure**: Even without the optimization, the CRM instance timed out after **1.5 days** before completing the sync
5. **Gateway timeout**: Proxy layers also terminated requests, preventing completion

[Erik Andersson]: "We did actually manage to download all of the contacts without the sync failing. When we came to the consent, however, then all hell broke loose because we have an optimization inside of Appsys which will render this useless like it will crash the full sync. Because memory issues, but even if we remove that optimization, the FSC Enterprise instance in question kind of timed out for 1 1/2 day while this sync was ongoing before it terminated."

### Broader Customer Impact

[Raziel Carvajal]: "Yeah, it's been an issue. I mean even even with 1,000,000 even it's been an issue because all this time usually for for having access to the information in the CRM, well sometimes the disconnection takes place. Then clients report that the CRM is slow and then when you check, when you check the the logs and check, OK, there is a sync condition as a sync conversation taking place."

Even customers with 1 million contacts experience CRM slowdowns during sync because the entire database must be queried and processed.

---

## Existing Solution: Microsoft Dynamics Approach

### How Dynamics Handles This

For Microsoft Dynamics (referred to as "dynamics by site shop"), Apsis has already implemented a different pattern:

1. **Sync conditions are still configured in Apsis** (keeping centralized configuration)
2. **Conditions are sent to the CRM** as parameters
3. **CRM applies filtering server-side** — only contacts matching the condition are returned
4. **Result**: For the French customer example, instead of downloading 8 million contacts, only the 100,000 matching contacts are retrieved
5. **Performance**: Sync time reduced from 1.5+ days to **5-10 minutes**

[Erik Andersson]: "One way or another, anytime we ask for a full sync, they will only present us with the contacts that are matching the sync condition. So in the case of this customer now we would only download these 100,000 and the sync would be done in like 5-10 minutes pops."

### Webhook Handling in Dynamics

For real-time webhooks, the Dynamics connector:
- Only sends webhooks for profiles matching sync conditions
- Reduces real-time traffic burden
- Can be filtered server-side or on the Apsis side

---

## Proposed Solution for Efficy Enterprise

### Option 1: Server-Side Configuration (Stateful)

Similar to Dynamics, Efficy Enterprise would:

1. Store sync conditions as a configuration parameter in the CRM system
2. Generate a cached or pre-computed list of matching contacts when needed
3. Return only matching records on full sync requests
4. Return only matching consents on consent requests

**Advantage**: Drastically reduces data transfer and request count

**Disadvantage**: Requires the CRM to maintain state, which contradicts the API design principle of statelessness

[Raziel Carvajal]: "For a moment we have treated all these square, all these. The request of this API is stateless. We don't keep any state in the. As soon as we receive the request, we just attend the request and then we we return the answer without keeping any cash because we might have a concurrency issues later or inconsistencies."

### Option 2: Query Parameters (Stateless, Preferred)

Efficy Enterprise would receive sync conditions as query parameters in the API request:

**For records endpoint:**
```
GET /records?page=0&pageSize=100&condition[0].field=firstName&condition[0].value=Eric&condition[1].field=country&condition[1].value=Sweden
```

**Key characteristics:**
- Stateless: CRM doesn't store state between requests
- Pagination-aware: Each request includes `page` and `pageSize` parameters
- Filtering happens at query time: CRM applies WHERE clauses dynamically
- Request stops when fewer results than page size are returned

[Erik Andersson]: "An alternative approach, if that is more to your liking, could be that you we we you could add the same condition as query parameters. When we make the full sync request"

**Why pagination still matters:**
- Cannot know in advance how many contacts match the condition
- Need to fetch in batches to avoid memory exhaustion on either side
- Apsis will send one extra request that returns empty to confirm completion

### Option 3: Preflight Request (Pre-generation Cache)

Used for high-traffic scenarios (e.g., Maxo implementation with 5 million contact IDs in 6 seconds):

1. Apsis sends a preflight request: "Prepare to sync with these conditions"
2. CRM generates cached result set and optimizes database queries
3. Apsis then makes normal full sync requests, which hit the cache
4. After sync completes, cache is discarded

[Erik Andersson]: "We did like a method of like a pre flight when we developed. A like extreme high traffic implementation for Maxo where we expected like 5 million contacts, contact IDs sent to us in like 6 seconds essentially. So then in that case we sent a request to the CRM saying hello. We are about to do this sync. Please prepare your data."

---

## Sync Conditions: Current Limitations and Future Improvements

### Current Boolean Logic (AND-only)

All sync conditions are evaluated with AND logic. There is no support for OR conditions or complex boolean expressions.

Example:
```
birthplace == "Sweden" AND address == "Stockholm"
```

Cannot express:
```
(birthplace == "Sweden") OR (birthplace == "France")
```

### New Sync Condition Version (Unimplemented)

A newer version supporting full boolean logic (AND/OR combinations) was developed but never shipped:

[Erik Andersson]: "We did implement a new version of sync conditions. Where we do support exactly what you said, but this never came to life because every front end resource disappeared on us and then the reorg happened. So this feature isn't in place for any customer."

**Reason for non-delivery**: Loss of frontend engineering resources during organizational restructuring

### Specification Documents

Sync conditions specifications need to be clearly documented so Efficy Enterprise knows:
- How many conditions are supported (currently ~10 limit)
- Boolean logic rules (currently AND only)
- Supported field types and operators
- Format of query parameters or configuration

[Raziel Carvajal]: "This has to be a change in the specification, right? You change the specification, then you let us know and then we change accordingly."

---

## Consent Synchronization Strategy

### The Consent Problem

Consents are separate entities tied to contacts. In the current flow:

1. Full sync downloads ALL contacts (8 million)
2. Full sync downloads ALL consents (at minimum 8 million, potentially 16+ million if contacts have multiple consents)
3. Conditions are evaluated in Apsis, discarding unmatched contacts **but all consents have already been downloaded**

This is why the French customer's sync failed: the sheer volume of consent records overwhelmed both systems.

### Proposed Consent Filtering

Apply the same condition-based filtering to consents that will be applied to records:

[Erik Andersson]: "Naturally we might do the same for consents, no define the the same conditions because this is otherwise it's going to be also confusing for the client that the final users are."

The get consents endpoint would receive the same query parameters:
```
GET /consents?page=0&pageSize=100&condition[0].field=birthplace&condition[0].value=Sweden
```

Efficy Enterprise would only return consents for contacts matching the conditions.

### Webhook vs. Full Sync for Consents

**Webhook scenario** (real-time updates):
- Payloads are small (only changed consents)
- Raziel's position: Filtering at webhook source may not be critical; Apsis can filter incoming webhooks
- Risk: Low unless clients stop webhook service while database continues recording changes

[Raziel Carvajal]: "Unless I have difficulties to see a scenario in which a CRM updates thousand of consents in in less than 5 minutes, you know, for instance."

**Full sync scenario** (batch):
- Must handle 8-16 million consent records
- Filtering at source is critical for performance

---

## Implementation Roadmap and Considerations

### What Needs Agreement

The three teams (Apsis product, Efficy Enterprise, implementation team) must agree on:

1. **API contract format** for query parameters:
   - Parameter naming convention
   - Condition array format
   - Field name mapping between Apsis and Efficy

2. **Response handling**:
   - Pagination strategy (page number, page size)
   - Empty response behavior (signals end of results)

3. **Scope**:
   - Records endpoint: YES (urgent, primary pain point)
   - Consents endpoint: YES (secondary but important)
   - Webhooks: DEFER (less critical, can be filtered server-side in Apsis)

[Erik Andersson]: "You, Rachel, would just need to agree on how those query parameters would look like."

[Raziel Carvajal]: "I think it makes sense to keep this. You update the API, you give the number whatever you want, the format whatever you want and you just give an example and we have questions we ask"

### Implementation Complexity

**For Apsis**: Low complexity
- Serialize existing sync conditions into query parameters or configuration
- Expand current get records/consents calls with new parameters

[Erik Andersson]: "This shouldn't be too hard for us to implement because essentially like we have the list of the sync conditions creating like a comma separated list for this to use in the query parameters."

**For Efficy Enterprise**: Medium complexity
- Modify get records endpoint to accept and apply filter conditions
- Modify get consents endpoint similarly
- Ensure dynamic WHERE clause generation (SQL injection prevention critical)
- Test pagination behavior with various condition combinations

---

## Performance Impact Projections

### Scenario: French Customer (8M contacts, 22 installations)

**Current approach:**
- Data transferred: 8M contacts + (8M × 22 × 2) consents = ~352 million records
- Time per installation: 1-1.5 days
- Total time: 22-33 days
- Risk: Memory crashes, gateway timeouts, CRM slowdown

**With condition-based filtering:**
- Data transferred: 100K contacts + (100K × 22 × 2) consents = ~4.4 million records
- Time per installation: 5-10 minutes
- Total time: 1.8-3.7 hours
- Risk: Minimal

### Pagination Benefits

Even with stateless pagination, filtering reduces:
- **Number of requests**: From ~22 requests per installation to ~2-3 requests (fewer pages to paginate)
- **Time per request**: Dramatically reduced (smaller datasets)
- **Total sync time**: Order of magnitude improvement

---

## Open Questions and Decisions Needed

### Question 1: Stateful vs. Stateless Configuration

**Stateful approach** (Configuration stored in CRM):
- Pros: Fewer requests, better caching opportunity
- Cons: Violates REST API statelessness principle, potential concurrency issues

**Stateless approach** (Query parameters in each request):
- Pros: Standard API design, no state management needed
- Cons: Slightly more requests needed for pagination

[Raziel Carvajal]: "I really like the idea to keep it stateless because it's it's a general API. The implementation shouldn't be."

**Team consensus**: Lean toward stateless with query parameters, with configuration fallback if parameterization proves complex.

### Question 2: URL Length Limitations

If conditions are sent as query parameters, URL length could become an issue. Mitigation:
- Current limit: ~10 sync conditions per customer
- Most customers use 2-3 conditions
- Unlikely to exceed typical URL length limits (~2KB)

[Erik Andersson]: "I don't think that's gonna be an issue" (because condition count is limited)

### Question 3: Webhook Filtering Necessity

**Position 1** (Erik): Should also filter webhooks for consistency and to reduce payload size

**Position 2** (Raziel): Webhook payloads are small; filtering at receipt is sufficient; real webhook scenarios don't bulk-update thousands of consents

**Resolution**: Defer webhook filtering; prioritize full sync filtering first. Real-time sync can use Apsis-side filtering.

---

## Priority and Urgency Assessment

### Affected Customers

- **3 confirmed customers** currently suffering performance degradation
- **French customer**: Sync configuration failing entirely (8M+ contacts)
- **Risk**: If conditions prevent adoption of Apsis by large enterprise customers

[Erik Andersson]: "We we have 3 customers that are suffering from this."

### Urgency Debate

**Erik's position**: HIGH URGENCY
- Multiple customers affected today
- Prevents enterprise adoption
- Sync failures in production

[Erik Andersson]: "I would highly recommend to do this sooner rather than later, and especially if customers CRMS are already affected like today."

**Raziel's position**: MEDIUM URGENCY
- If syncs happen overnight, delayed completion is acceptable
- System still works (just slowly)
- Only critical if syncs conflict with business hours

[Raziel Carvajal]: "Practically speaking for the final user, frankly, if they do the synchronisation at night, let's say and it took the whole night, then in the morning they want to have the synchronised data. So in that point of view is not rather urgent"

**Pragmatic resolution** (Lukasz): Create an epic/story to document the issue with priority to be determined post-session. Implement before Erik's departure if possible, otherwise queue for team.

[Lukasz Grabowski]: "Let me write an epic and try to describe it as a, you know, exercise for me. Understanding this I have recording so I will try to you know rephrase it... then we'll prioritize it and maybe we'll try to implement this before you. So you're leaving."

---

## Architectural Principles Discussed

### Statelessness in APIs

The Efficy Enterprise team follows REST principles requiring stateless API design:
- No server-side session storage between requests
- Each request contains all necessary context
- Enables horizontal scaling and concurrency

**Tension**: Stateless design sometimes conflicts with performance optimization (caching), requiring explicit design choices.

### Separation of Concerns

**Current problematic design**:
- CRM returns all data
- Apsis filters based on conditions
- Result: Network and memory bottleneck in Apsis

**Proposed design**:
- Apsis specifies filter criteria
- CRM applies filters (where it has direct database access)
- Result: Only relevant data transferred

This follows the principle of processing data closest to where it lives.

### Backward Compatibility

Any API changes to get records/consents endpoints should maintain backward compatibility:
- New condition parameters should be optional
- Existing calls without parameters should still work
- Documentation changes required

---

## Next Steps and Action Items

1. **Lukasz**: Write an epic documenting the sync conditions performance issue
   - Rephrase technical details from this discussion
   - Define acceptance criteria (sync time targets, data volume thresholds)
   - Request review from Erik before he departs
   - Prioritize relative to other work

2. **Erik & Raziel**: Define API contract for query parameters
   - Parameter naming scheme
   - Condition format/structure
   - Example payloads
   - Error handling cases

3. **Efficy Enterprise team** (Lukasz + others): Evaluate implementation effort
   - Dynamic WHERE clause generation in get records endpoint
   - Dynamic WHERE clause generation in get consents endpoint
   - SQL injection prevention
   - Test scenarios: various condition combinations, large result sets, empty results

4. **Apsis Product**: Evaluate impacts
   - Risk of regression in existing sync scenarios
   - Interaction with new sync condition version (OR logic)
   - UI changes needed to communicate condition delegation to CRM

---

## Key Takeaways

1. **The Problem**: Current sync condition evaluation (download everything, then filter in Apsis) causes catastrophic performance degradation at scale—8M contact syncs fail after 1.5+ days

2. **The Pattern Exists**: Microsoft Dynamics connector already implements server-side condition filtering successfully (5-10 minute syncs for equivalent data)

3. **The Solution**: Move sync condition evaluation to Efficy Enterprise via query parameters (stateless) or configuration (stateful), filtering at the source

4. **The Tradeoff**: Stateless query parameters preferred for API design purity but require pagination logic; stateful configuration faster but violates REST principles

5. **The Scope**: 
   - Records filtering: CRITICAL and urgent
   - Consents filtering: IMPORTANT, same approach as records
   - Webhooks: DEFER for now

6. **The Effort**: Low for Apsis (serialize conditions into requests), medium for Efficy Enterprise (dynamic query building, pagination, testing)

7. **The Priority**: 3 customers suffering; implementation recommended before Erik's departure if possible, otherwise queued with epic documentation

---

## Unresolved Questions

1. **API parameter format**: Exact naming and structure of condition query parameters not finalized (to be designed by Erik & Raziel)

2. **Pagination behavior**: Exact semantics of page boundaries when conditions are applied—does page size remain constant or is it approximate? (Assumed page size remains page count regardless of filter selectivity)

3. **Configuration vs. Query Parameters**: Final decision between storing conditions in CRM configuration vs. passing per-request (leaning toward per-request for statelessness)

4. **Webhook filtering**: Whether webhooks should also be filtered server-side or continue to be filtered by Apsis (deferred, currently Apsis-side filtering acceptable)

5. **New sync condition boolean logic**: How the unimplemented OR/AND logic feature will interact with the new filtering approach (not discussed in detail; may need separate design)