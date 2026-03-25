---
source_file: Erik - Handling Sync Conditions in Efficy Enterprise.txt
domain: Apsis One Integrations
topics: [Sync Conditions, Full Sync Performance, Large-Scale Data Synchronization, Efficy Enterprise API Design, Pagination with Filters, Contact and Consent Syncing, CRM Integration Optimization]
speakers: ["Erik Andersson", "Raziel Carvajal", "Lukasz Grabowski"]
key_components: ["Sync Conditions", "Full Sync", "Real-time Webhooks", "Efficy Enterprise API", "Get Records Endpoint", "Consent Syncing", "Query Parameters", "Pagination"]
session_type: architecture-review
subdomains: ["Architecture", "Efficy Enterprise 12.0", "Outbound Flow"]
---

## Session Overview

This session covers a critical performance problem in the Apsis One integrations domain: the handling of **sync conditions** when synchronizing large volumes of contacts from Efficy Enterprise CRM systems. Erik Andersson presents a use case where a customer has 8 million contacts in their CRM but only wants to sync 100,000 based on sync conditions (e.g., birthplace = Sweden). The current architecture downloads all contacts regardless of sync conditions, then filters them in APSIS, causing multi-day sync failures for large customers. The team discusses solutions involving passing sync conditions to the Enterprise API as query parameters to enable server-side filtering, reducing data transfer and processing time dramatically.

---

## The Sync Conditions Problem: Context and Scale

### What Sync Conditions Are and Why They Exist

[Erik Andersson]: In the sync flow, **sync conditions** regulate which contacts from the CRM system are actually synced to APSIS. For example, if you only want to sync contacts that originate from Sweden for a targeted marketing campaign, you store these conditions inside APSIS.

These conditions were designed this way from the start because we were integrating with CRM systems over which we had no control. We couldn't add logic anywhere else but inside of APSIS.

> "This has worked quite good up until quite recently"

### The Customer Problem: 8 Million Contacts, 22 Installations

The problem manifests with a customer in France who has:
- **8 million contacts** in their Efficy Enterprise CRM (far exceeding stipulated limits)
- **22 APSIS installations** (different sections, likely for organization, group, or region)
- Only **100,000 contacts should actually be synced** (those matching sync conditions)

When performing a full sync:

1. **Current behavior**: Download all 8 million contacts to evaluate sync conditions in APSIS, then discard 7.9 million
2. **Consent data multiplier**: Each contact has at least one consent, adding ~8 million consent records
3. **Multi-installation problem**: Multiply this by 22 installations = obscene data volume

[Erik Andersson]: "I did check on the initial sync for the customer. We did actually manage to download all of the contacts without the sync failing. When we came to the consent, however, then all hell broke loose."

The sync process failed with memory issues (an optimization in APSIS renders the process unusable), and even without that optimization, the Efficy Enterprise instance timed out after 1.5 days before terminating with a gateway timeout error.

### Historical Impact and Scope

[Raziel Carvajal]: This isn't a new problem—it's been an issue even with 1 million contacts. When the sync takes place, clients report that the CRM is slow. When checking logs, there's a sync condition evaluation happening simultaneously. While surprisingly few clients have complained, the issue remains.

---

## The Solution: Server-Side Filtering with Sync Conditions

### Precedent: Dynamics by Site Shop Implementation

[Erik Andersson]: For another CRM (Dynamics by Site Shop), while sync conditions are still configured inside APSIS to keep everything in one place, the conditions are also sent to the CRM system. The CRM stores these conditions and presents only matching contacts during full sync requests.

**Result**: Instead of downloading 8 million contacts, download only 100,000. Sync time reduces from 1.5 days to **5-10 minutes**.

The same approach was applied to webhooks—they only send webhooks for profiles matching sync conditions.

### Two Potential Implementation Approaches

#### Option 1: Configuration-Based (Stateful)
- APSIS sends sync conditions to Efficy Enterprise as a configuration parameter
- Enterprise caches or pre-generates filtered data
- Full sync request returns only matching contacts
- **Advantage**: Minimal API changes; simpler pagination handling
- **Disadvantage**: Enterprise must maintain state

#### Option 2: Query Parameters (Stateless)
- APSIS includes sync conditions as query parameters in the GET request
- Each request is independent (no state stored in Enterprise)
- Pagination continues to work normally
- **Advantage**: Follows API statefulness principle; no cache management needed
- **Disadvantage**: Query string can become long; must handle pagination carefully

[Erik Andersson]: "For us this would be like you flick a switch. We will send you the sync conditions and they're stored inside APSIS."

### Advanced Option: Pre-Flight Prefetch

[Erik Andersson]: For an extreme high-traffic implementation with Maxo (expecting 5 million contact IDs in 6 seconds), a pre-flight approach was used:

1. Send a request to the CRM: "We are about to sync. Please prepare your data."
2. CRM runs SQL queries and readies a cache
3. Full sync proceeds with pre-prepared data

This requires coordination but is viable if Enterprise can maintain up-to-date filtered lists.

---

## Technical Design Discussion: Stateless vs. Stateful

### The Stateless Preference

[Raziel Carvajal]: All API requests are currently treated as stateless. As soon as a request is received, it's processed and a response returned—no state is kept because stateless design avoids concurrency issues and inconsistencies.

> "The implementation shouldn't be dependent on state unless the specification explicitly requires it"

However, [Raziel Carvajal] acknowledges the stateless approach creates a pagination problem: if you send sync conditions in a stateless request, you don't know how many matching records exist without storing that information somewhere.

### Pagination and Pagination Concerns

[Raziel Carvajal]: If APSIS sends sync conditions + page parameters:
- Request 1: `conditions + page=0`
- Request 2: `conditions + page=1`
- Each request must independently filter and return the requested page
- This requires Enterprise to rerun the same query for each page

While this reduces *response time per request* (smaller result set), it doesn't reduce the *number of requests* or total *query load on the CRM*.

[Erik Andersson] counters: If the page size is designed so responses come back smaller than the page limit, APSIS can detect when it's reached the last page and stop requesting. This would still be more efficient than downloading all records.

> "We will always send one more request than we technically need to, but that should reduce the amount of requests for you"

### Resolution: Configuration Parameter Approach

The team converges on storing the sync condition as a configuration parameter (stateful), but keeping individual requests stateless:

1. **Configuration endpoint**: APSIS sends sync conditions once to a system information/configuration endpoint
2. **Get Records requests**: Enterprise uses the stored condition to filter results
3. **Pagination**: Sync conditions + page number sent in each request, but Enterprise doesn't need to maintain request-level state

[Raziel Carvajal]: "If we require a configuration, we can add such a parameter to the endpoint that we have the system information."

[Lukasz Grabowski]: This clarifies the "stateless" concern—the API itself remains stateless per request, but configuration is stored separately.

---

## API Contract Specifics

### Get Records Endpoint: Query Parameters for Sync Conditions

[Erik Andersson]: For the GET records request, sync conditions would be introduced as query parameters:

```
GET /records?firstName=<value>&country=<value>&page=0&pageSize=100
```

**Format considerations**:
- Enterprise field names (not APSIS field names) are specified
- Multiple conditions are combined with AND logic
- Page size and page number remain as existing parameters
- Fields to return can still be specified separately

[Erik Andersson]: "We have a limit of 10 sync conditions if memory serves, and typically customers only have two or three, so URL length shouldn't be an issue."

### Current Limitation: AND Logic Only

[Erik Andersson]: Today, sync conditions only support AND logic:
```
birthplace = 'Sweden' AND address = 'Stockholm' AND ...
```

A new version of sync conditions supporting OR and complex logic was implemented but never shipped (all frontend resources were reassigned, then the reorganization happened). So for now, everything is AND.

[Raziel Carvajal]: "As far as this is clearly written in the specification, it's fine with us. You tell us that in your documentation, then we can take a look and implement it."

### Contact vs. Consent: Separate Handling

**Contacts (Records)**: Sync conditions can be passed as query parameters to the GET records endpoint.

**Consents**: No sync conditions currently exist for consents, but applying the same approach is desirable for consistency.

[Erik Andersson]: "If we provide you with the conditions for consent too, we have one unified flow of handling it for both records and consent, and our sync process will be cleaner. Everything is messages for us—we don't separate consent or consents."

The consent endpoint would receive the same sync conditions as query parameters, returning only consents for contacts matching those conditions. This prevents downloading 16 million consent records (8 million contacts × 2 consents each) when only 200,000 are needed.

---

## Real-Time Webhooks: Secondary Concern

[Raziel Carvajal]: Webhooks are a lower priority for sync condition filtering because:

1. CRM webhooks trigger only when actual changes occur
2. The payload size for a single change is small
3. Enterprise periodically triggers webhooks to push buffered changes
4. Unless a service is stopped and many records accumulate, webhook volume is manageable

> "Unless I have difficulties to see a scenario in which a CRM updates thousands of consents in less than 5 minutes"

[Raziel Carvajal]: "In the webhook, the standard practice is the receiving system makes the filtering. That's how it's working right now."

However, if the same sync conditions are applied to webhooks, Enterprise would only send webhooks for profiles matching conditions, further reducing traffic. But this is not urgent compared to full sync performance.

---

## Performance Impact Analysis

### Full Sync with 100,000 Matching Contacts (Server-Side Filtering)

**Before** (with 8 million total contacts):
- Download all 8 million contacts
- Download ~16 million consent records (2 per contact)
- Repeat for 22 installations
- Time: 1.5+ days, system crashes

**After** (with query parameters):
- Download only 100,000 matching contacts
- Download only ~200,000 matching consent records (2 per matching contact)
- Repeat for 22 installations
- Time: ~5-10 minutes per installation
- Total for 22 installations: Much faster, parallel processing possible

### Request Volume Analysis

[Erik Andersson]: If a page size is 100 and there are 100,000 matching contacts:
- 1,000 requests needed (100,000 / 100)
- vs. 80,000 requests needed if all 8 million contacts were paginated

The reduction in request count is significant. With a page size that results in smaller responses, pagination can detect completion faster.

---

## Implementation Scope and Ownership

### APSIS Side
- Build query parameter support in the sync engine
- Maintain existing sync condition UI/configuration
- Send conditions in GET records and GET consents requests
- No major changes to the architecture

[Erik Andersson]: "For us this shouldn't be too hard. We just have to create a comma-separated list of conditions to use in query parameters."

### Efficy Enterprise Side (Lukasz Grabowski's Team)
- Accept sync condition query parameters in GET records endpoint
- Accept sync condition query parameters in GET consents endpoint
- Filter results based on these parameters
- Maintain pagination support
- Keep the API stateless per-request

[Lukasz Grabowski]: "Let me write an epic and try to describe it... I have a recording so I will try to rephrase it and if I miss something I'll ask you to review."

### API Contract Definition
- APSIS team defines query parameter names, format, and documentation
- Enterprise team implements according to specification
- [Raziel Carvajal]: "You update the API, you give the format, you give an example, and we have questions we ask. I don't see why we should really debate the name."

---

## Urgency and Business Impact

### Customer Impact

Three customers are currently suffering from this issue (identified as urgent by Erik):
1. The France customer with 8 million contacts and 22 installations
2. Other large CRM instances affected by similar issues

[Raziel Carvajal]: From a practical standpoint, if full syncs run overnight and complete by morning, it's less urgent. However, if large syncs occur during business hours or if customers expect faster sync times, it's critical.

[Erik Andersson]: "I would highly recommend to do this sooner rather than later, especially if customers' CRMs are already affected."

### Timeline Context

[Lukasz Grabowski]: "It is important for me to understand more and more this world... maybe we'll try to implement this before you [Erik] are leaving. Not sure if this is possible."

Implies Erik is soon departing the project, making knowledge transfer and prioritization urgent.

---

## Key Unresolved Questions and Action Items

1. **Query Parameter Format**: Exact names, structure, and documentation format to be defined by APSIS and reviewed by Enterprise
   - **Owner**: Erik Andersson / APSIS product team
   - **Dependent**: Lukasz Grabowski / Enterprise team

2. **Consent Sync Conditions**: Should the same sync conditions apply to consent endpoints?
   - **Status**: Agreed in principle, pending implementation prioritization
   - **Owner**: Both teams

3. **Webhook Sync Conditions**: Lower priority, but should consistency be maintained?
   - **Status**: Acknowledged as secondary concern
   - **Owner**: Deferred pending full sync implementation

4. **Epic/User Story Documentation**: Lukasz to create documentation for engineering tracking
   - **Owner**: Lukasz Grabowski
   - **Dependency**: Review by Erik Andersson

5. **Pagination Behavior with Small Result Sets**: Exact behavior when filtered results are fewer than page size
   - **Status**: Clarified (send one extra empty page, then stop)
   - **Owner**: Erik Andersson to document in API spec

6. **Backward Compatibility**: How will this work with existing sync configurations and customers not using the feature?
   - **Status**: Not explicitly discussed
   - **Owner**: APSIS product/engineering

---

## Key Takeaways

1. **The Problem**: Current sync condition evaluation in APSIS causes full syncs of large CRM databases (8M+ contacts) to download all records regardless of filtering needs, resulting in multi-day syncs or outright failures for customers.

2. **The Solution**: Pass sync conditions from APSIS to Efficy Enterprise as query parameters, enabling server-side filtering. This reduces downloaded data volume by ~99% in the customer's case (8M → 100K contacts).

3. **Performance Gains**: Estimated time reduction from 1.5 days to 5-10 minutes per installation; manageable request volumes even with pagination.

4. **API Design Decision**: Use stateless query parameters (not stored configuration) to maintain API purity, with configuration stored separately at system setup time. Each request includes sync conditions + page parameters.

5. **Unified Flow**: Both contacts and consents should use the same sync condition mechanism for consistency and to prevent consent download explosions (16M+ records when only 200K needed).

6. **Current Limitation**: Sync conditions only support AND logic today. OR/complex logic was built but never shipped.

7. **Webhooks Are Secondary**: Real-time webhooks have small payloads and don't urgently need filtering; full sync is the critical path.

8. **Immediate Next Steps**: 
   - APSIS defines query parameter spec
   - Lukasz documents epic for Enterprise implementation
   - Teams agree on exact API contract
   - Implementation proceeds (timeline uncertain due to urgent production issue, but marked important)

---

## Unresolved Questions

- **Exact query parameter names and format** — to be defined by APSIS and submitted for Enterprise review
- **Backward compatibility story** — how will existing customers migrate or opt into this feature?
- **Request timeout/gateway limits** — will pagination with sync conditions hit any Enterprise infrastructure limits?
- **Performance testing scope** — what are the benchmarks for success?