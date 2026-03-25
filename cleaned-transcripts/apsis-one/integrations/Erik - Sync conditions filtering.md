---
source_file: Erik - Sync conditions filtering.txt
domain: Apsis One Integrations
topics: [Sync conditions filtering, Full sync optimization, CRM query parameters, Enterprise API integration, Performance scaling]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Generic Connector, FSC Enterprise API, Apsis One Core, Delta Sync Manager, Full Sync Producer, Mock Service]
session_type: architecture-review
subdomains: [Architecture, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration]
---

## Session Overview

This knowledge transfer session discusses a critical architectural change to push **sync conditions filtering upstream** from Apsis One to FSC Enterprise systems to address severe performance issues with large-scale customer data syncs. The team reviewed the performance problem (8 million contacts being downloaded then 99% discarded), evaluated implementation approaches (query parameters vs. request body), and outlined a phased rollout plan involving specification updates, enterprise development, testing, and feature flag deployment.

---

## Problem Statement: Full Sync Performance Bottleneck

### The Core Issue

[Erik Andersson]: Currently, sync conditions are evaluated **inside Apsis One after downloading all contacts from CRM systems**. This approach requires downloading every single contact record to determine whether it matches the sync conditions before filtering. For French customers with 8 million contacts in their CRM, this means:

- Downloading 8 million contact records
- Downloading at least 8 million associated concerns/related data
- Discarding 99% of the data after retrieval

> "This is ridiculous and it is crashing APSIS full syncs and it is crashing the CRM systems."

### Impact on Current Customers

The problem has only recently become acute because customers now have:
- Large contact volumes (8 million+) that break the current architecture
- Full sync operations that crash both Apsis One and the CRM systems
- Unacceptable data transfer volumes and latency

---

## Current Workarounds in Other Integrations

### Microsoft Dynamics Implementation (Stateful)

[Erik Andersson]: The Microsoft Dynamics implementation already handles this correctly through a **stateful approach**:
- Sync conditions are stored locally in the Dynamics system
- Apsis One sends sync conditions to Dynamics
- Dynamics caches or paginates filtered results
- Only matching records are returned on full sync requests
- Webhooks only send updates for contacts matching the conditions

This model works well but requires the CRM system to maintain state about which conditions apply.

### FSC Enterprise Challenge (Stateless)

FSC Enterprise has a **stateless API**, which means:
- No ability to store sync condition state between requests
- Each API call must be self-contained
- Conditions must be included in every request to get proper filtering
- Requires a different implementation approach than Dynamics

---

## Proposed Solution: Sync Conditions as Query Parameters

### Architecture Decision

[Erik Andersson]: The solution is to pass sync conditions as parameters in the API request itself, allowing FSC Enterprise to filter data server-side before returning results.

**Expected outcome**: Reduce 8 million contacts down to approximately 100,000 by applying filtering at the source.

### Two Implementation Approaches Considered

#### Approach 1: Sync Conditions as Structured List (Recommended)

Pass all conditions as a single `syncConditions` query parameter containing the full filter specification:

```
syncConditions=firstName equals Eric AND lastName equals Andersson AND country equals Sweden
```

**Advantages:**
- Explicitly named parameter makes clear what is being sent
- Follows semantic clarity (not ambiguous generic parameters)
- Easy to document and understand
- More maintainable long-term

**Concerns:**
- URL length could become problematic with many conditions
- Requires URL encoding for special characters
- Looks unusual in URL format

#### Approach 2: Individual Attribute Query Parameters (Alternative)

Pass each attribute name as a separate query parameter:

```
firstName=Eric&lastName=Andersson&country=Sweden
```

**Issues with this approach:**
[Erik Andersson]: "This goes a bit outside of the generic connector standard... it would be more specific to FSC Enterprise rather than truly generic."

[Lukasz Grabowski]: "I would prefer the first version... it's more clear that we sent sync conditions and not just some parameters."

### Decision: Query Parameters Chosen Over Request Body

[Erik Andersson]: The team debated whether to use query parameters (REST-standard) or request body (non-standard for GET requests).

> "The query parameters is... it's not restful to use request body for a GET request. Query parameters are the standard approach."

**URL Length Not a Limiting Factor:**
- Standard URL length limit: ~2000 characters
- Typical sync conditions: 2 maximum (e.g., "contact is active" AND "country equals X")
- Concern dismissed as unlikely to be problematic in practice

[Lukasz Grabowski]: "How many conditions can they have? Typically two... we can set some limitation... we can also limit the UI."

### ALL vs. OR Logic Limitation

[Erik Andersson]: Current implementation assumption: **All conditions are implicitly AND'ed together.**

> "The one precondition here: we are doing this now with the old sync conditions in mind where everything is AND, so you can just create one big list of every sync condition because it's implicit that this is AND."

**Current scope**: OR logic does not yet exist in this filtering system. If OR logic is needed in future, the approach may need revision.

---

## Technical Specification and Implementation

### Generic Connector Specification Location

The specification is not stored internally, but is available in the **public integration bucket** for download by all team members.

### Two Endpoints Requiring Changes

[Erik Andersson]: "You need to... two of them."

1. **`getRecords` endpoint** - for fetching contact records
2. **`getConsents` endpoint** - for fetching consent records

Both must support passing sync conditions as query parameters.

### Current Generic Connector Parameters

The `getRecords` method currently accepts:

```
- entity: which entity to download (e.g., "contacts" for FSC Enterprise, "persons" for E-deal)
- pageSize: number of records per page
- page: pagination offset
- fields: which attributes/properties are of interest
- IDs: specific record IDs (used for webhook callbacks to fetch updated data)
```

**New parameter to add:**
```
- syncConditions: filter specification (conditional, based on CRM system capability)
```

### Installer Option Flag

A new installer configuration flag must be added to the **Enterprise connector configuration**:

```
supports_sync_conditions_as_query_param: boolean
```

[Erik Andersson]: "You need to have a installer option that says like supports sync conditions as query parameters, and if that is true then this should be added to the request."

This flag allows conditional behavior:
- If `true`: Send sync conditions as query parameters to the CRM system
- If `false`: Fall back to old behavior (download all, filter in Apsis)

---

## Implementation Workflow and Testing Strategy

### Phased Rollout Process

[Lukasz Grabowski]: The team outlined a 5-step deployment sequence:

1. **Update Generic Connector Specification**
   - Extend the specification document to include sync conditions parameter definition
   - Serve as the "shared source of truth" for both Apsis and Enterprise teams
   - Document expected format, required fields, and behavior

2. **Transmit Specification to FSC Enterprise Team**
   - Assign to: Rizel (or equivalent Enterprise team member)
   - Provide clear requirements for parameter handling

3. **Enterprise Development and Release**
   - Enterprise team implements sync condition parameter handling in their API
   - Must support filtering in both `getRecords` and `getConsents` endpoints
   - Must maintain pagination integrity when filtering
   - Must be released to production before Apsis can use it
   - [Erik Andersson]: "They need to add support for it first. When they have added support for it and released it, then that's when you would add this to the master branch."

4. **Apsis Development (Can Start in Parallel)**
   - Implement sync condition parameter building logic in Generic Connector
   - Add conditional logic: only send parameters if `supports_sync_conditions_as_query_param == true`
   - Implement changes in two places: full sync requests and consent sync requests
   - Development can proceed in parallel but deployment waits for Enterprise

5. **Testing Before Production Release**
   - Deploy to staging environment with mock service
   - [Erik Andersson]: "What you can do if when you want to test this is that you add support for this in the mock service. The mock service accepts these conditions and then when you can deploy something manually to staging while you while you test this."
   - Verify with actual Enterprise team testing in their environment
   - Ensure correct data is returned and no duplicates are introduced
   - [Erik Andersson]: "This is a very easy pitfall to miss out on contacts and apps will have no idea that this is happening because we will just get contact IDs if things are missing from the CRM... This needs to be heavily tested."

### Critical Testing Caveat

[Erik Andersson]: There is a significant risk if filtering is incomplete or incorrect:

> "This is a very easy pitfall to miss out on contacts and apps will have no idea that this is happening because we will just get contact IDs if things are missing from the CRM like we will be blind for it, so this needs to be heavily tested."

If Enterprise filters too aggressively and some matching contacts are excluded, Apsis will not detect the discrepancy because:
- Apsis relies on receiving contact IDs from Enterprise
- Missing data is invisible if the ID was never sent
- No alerting mechanism would catch silent data loss
- Duplicate profiles could be created if sync is incomplete

---

## Code Changes in Apsis One

### Two Sync Paths Affected

#### Full Sync Path (Condition Filtering Logic Removed)

[Erik Andersson]: "In the full sync, yes [remove condition filtering]... in the full sync you don't need to have it and it's in the full sync where you will have [the biggest benefit]."

When `supports_sync_conditions_as_query_param == true`:
- **Send**: Sync conditions as query parameters to Enterprise
- **Remove**: Condition verification logic in **Full Sync Producer**
- **Benefit**: Saves significant computational time by not re-filtering already-filtered data

#### Real-Time Sync Path (Delta Sync Manager - Condition Filtering Remains)

[Erik Andersson]: "For enterprise they will still send webhooks for everything. In the first step you need to keep that in the real time sync... because they have no idea about when they are about to send a webhook."

When `supports_sync_conditions_as_query_param == true`:
- **Send**: Sync conditions as query parameters to Enterprise
- **Keep**: Condition verification logic in **Delta Sync Manager**
- **Reason**: Enterprise webhooks have no awareness of sync conditions and will send updates for all contacts. Real-time filtering is still necessary.

[Lukasz Grabowski]: "And the second also we skip the condition filtering logic here... we will expect the exact data from them... in the full sync."

### Computational Efficiency Gains

[Erik Andersson]: "The full sync you don't need to verify this anymore if you have this enabled and there you will save quite some computational time as well."

By removing the filtering logic from the full sync path (since Enterprise now filters), Apsis avoids:
- Loading and evaluating conditions for every contact
- Computing filter matches across millions of records
- Processing overhead on condition evaluation

---

## Why This Matters: Historical Context

The current architecture was adequate when customers had smaller contact volumes, but breaks down at scale. This change represents a fundamental shift from a **pull-then-filter model** to a **filter-at-source model**, following the pattern already successfully implemented in Microsoft Dynamics.

---

## Unresolved Questions / Action Items

1. **Specification Documentation**: Erik Andersson to send detailed specification notes to chat/documentation system, including exact format examples for sync condition query parameters.

2. **Enterprise Team Assignment**: Confirm assignment to Rizel/Enterprise team for implementation of sync condition parameter support in FSC Enterprise API.

3. **Mock Service Updates**: Ensure mock service in testing environment can accept and process sync condition query parameters before integration testing begins.

4. **Validation Queries**: Define exactly how Apsis will validate that Enterprise is returning the correct filtered data set (especially critical given the "blind spot" risk mentioned above).

5. **OR Logic Support**: Clarify future roadmap for OR logic in conditions—current implementation assumes AND-only logic. Document whether OR support is planned and how it would affect the query parameter format.

6. **Installer UI**: Determine where and how the `supports_sync_conditions_as_query_param` flag appears in the installer UI for Enterprise connector configuration.

---

## Key Takeaways

1. **Upstream filtering is mandatory at scale**: The current architecture cannot handle 8 million contact syncs. Moving filter logic to the CRM system is not optional for large customers.

2. **Query parameters are the approach**: Despite longer URLs for complex conditions, query parameters are the correct REST-standard choice for passing sync conditions to a stateless API like FSC Enterprise.

3. **Testing is critical and difficult**: Silent data loss is possible if Enterprise filtering fails. Because Apsis operates on contact IDs returned, missing data is invisible. Heavy testing with the Enterprise team is non-negotiable.

4. **Implementation is split between systems**: Apsis development can proceed in parallel, but production deployment must wait for Enterprise to release their implementation first.

5. **Dual sync paths require different handling**: Full sync removes condition filtering (Enterprise handles it); real-time sync keeps filtering (Enterprise webhooks are condition-unaware).

6. **Effort distribution**: Erik estimated the Apsis-side work at 2 days maximum; the heaviest work is on the Enterprise system, making this a collaborative engineering effort.