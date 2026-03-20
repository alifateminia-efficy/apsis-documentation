---
source_file: Erik - Sync conditions filtering.txt
domain: Apsis One - Integrations
topics: [sync conditions filtering, full sync optimization, CRM API integration, FSC Enterprise integration, query parameters vs request body, webhook filtering, performance optimization]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Generic Connector, FSC Enterprise API, Delta Sync Manager, Full Sync Producer, Mock Service, Apsis Core]
session_type: architecture-review
---

## Session Overview

This session focuses on a critical performance optimization for the Apsis One integrations domain: moving sync conditions filtering upstream to CRM systems (specifically FSC Enterprise) to reduce full sync load. The problem stems from customers with millions of contacts in their CRM systems causing APSIS to download and discard 99% of records. The team discusses the technical approach to push filtering to the CRM via query parameters, the implementation timeline, testing strategy, and required changes to both the specification and codebase.

---

## Problem Statement: Current Sync Conditions Filtering Bottleneck

### The Core Issue

**Sync conditions are currently evaluated inside APSIS**, which requires downloading every contact from the CRM system to determine whether they match the sync criteria. [Erik Andersson]: 

> The sync conditions are evaluated inside of APSIS today, and because they are evaluated inside of APSIS, it means that we need to download every contact from the CRM systems to know if they should be added to APSIS or not.

### Real-World Impact

This design became problematic with large customer bases. [Erik Andersson]:

> We have French customers that have 8 million contacts in their CRM system and like this is not working with the way it is set up because we need to download 8 million contacts and then like at least 8 million concerns and then throw away 99% of it.

The consequences are severe:
- **APSIS full syncs are crashing**
- **CRM systems are crashing** from the load of unnecessary data transfer
- Extremely inefficient resource utilization

### Target Optimization

The goal is to **reduce the 8 million contacts in the sync down to approximately 100,000** by filtering at the source (CRM system) before data is transmitted to APSIS. [Erik Andersson]: 

> APSIS should never ever receive these contacts that are not matching the same condition.

---

## Existing Solutions in Other CRM Systems

### Microsoft Dynamics and Shopify Implementation

Some CRM integrations already have stateful implementations that solve this problem. [Erik Andersson]:

> The Microsoft Dynamics by site shop implementation. They have like a stateful implementation where we store the sync condition in the in like in like. They store it locally and we send them the sync condition and when we ask for records in the full sync, they only give us the ones that are matching.

How this works:
- Sync conditions are cached or prepared locally in the CRM system
- APSIS sends the sync condition to the CRM
- The CRM returns only matching records during full sync
- **Webhooks also respect filtering** — only updates for matching contacts are sent

### FSC Enterprise Challenge

FSC Enterprise, where the problematic customers are located, does not have this capability. [Erik Andersson]:

> If it is in FSC enterprise that the problematic customers are. But FSC enterprise does not have as like stateful API, they have a stateless API.

The stateless nature of FSC Enterprise's API means a different approach is required.

---

## Proposed Solution: Query Parameter-Based Filtering

### High-Level Approach

Since FSC Enterprise has a stateless API, APSIS must send sync conditions with each request. [Erik Andersson]:

> When we make a request to the CRM system, we should include the sync conditions as a query parameter or in the request body. And that's what I need to discuss with you what is best here.

The CRM system will then:
- Receive the sync conditions
- Execute a SQL query filtering based on those conditions
- Return only matching records
- Handle pagination on their side

### Query Parameter vs Request Body Design Decision

#### Option 1: Query Parameters (Recommended)

**Rationale**: RESTful standard practice for specifying filters.

Example format:
```
sync_condition=first_name equals Eric AND last_name equals Andersson
```

[Erik Andersson]: This would look very weird as a query parameter format, but is still syntactically legal.

**Advantages**:
- Follows REST conventions
- Cleaner API design
- Existing query parameter infrastructure in the Generic Connector

**Concerns**:
- URL length limits (standard 2000 characters, but depends on CRM system)
- Complex encoding of complex conditions

**Length Risk Assessment**: [Lukasz Grabowski and Erik Andersson agreed] this is **not a practical concern** because:
- Typically only 2-3 sync conditions exist (e.g., "contact is active" AND "country equals something")
- The UI can define and limit the number of conditions
- 2000 character limit provides ample room for typical use cases

#### Option 2: Request Body (Not Recommended)

**Rationale**: Potentially safer for long filter strings.

**Disadvantages**:
- Not RESTful for GET requests (non-standard practice)
- More complex implementation
- Inconsistent with standard API design

**Decision**: The team chose **query parameters** as the standard approach.

---

## Generic Connector Specification Changes

### Current `get_records` Endpoint Parameters

The Generic Connector currently supports:

```
get_records method:
- entity: which entity to download (e.g., "contacts" for FSC Enterprise, "persons" for other systems)
- page_size: pagination size
- page: page number
- fields: which fields/attributes are of interest
- IDs: specific record IDs (used for webhook callback validation to fetch updated data)
```

### New Required Parameter

A new sync conditions parameter must be added to both:

1. **`get_records` endpoint** — for full sync filtering
2. **`get_consents` endpoint** — for consent data filtering (also subject to sync conditions)

### Proposed Specification Format

[Erik Andersson described the intended format]:

```
sync_condition=first_name equals Eric AND last_name equals Andersson
```

The conditions are sent as-is with AND operators because:
- Currently, only AND logic exists in sync conditions (OR doesn't exist yet)
- Each sync condition is implicitly AND'd together
- This is simple to communicate to the CRM system

### Stateful Option Requirement

Because not all CRM systems support sync conditions as query parameters, the Generic Connector needs a new installer configuration option:

```
supports_sync_conditions_as_query_param: boolean
```

**Usage**:
- When `true`: APSIS sends sync conditions as query parameters
- When `false`: APSIS handles filtering locally (legacy behavior, less performant)

---

## Implementation Plan and Sequencing

### Critical Sequencing Issue

[Erik Andersson emphasized]: **The CRM system MUST implement support BEFORE APSIS releases the feature.**

If APSIS sends query parameters that FSC Enterprise doesn't understand, the request will fail or return incorrect results.

### Step 1: Extend the Generic Connector Specification

- Document the new sync conditions query parameter format
- Specify the format for both `get_records` and `get_consents` endpoints
- Define the installer option: `supports_sync_conditions_as_query_param`
- This is the **single source of truth** for both APSIS and CRM teams

**Responsibility**: APSIS team (likely [Lukasz Grabowski])

### Step 2: Communicate Specification to FSC Enterprise Team

**Send to**: Raisel (FSC Enterprise team)

FSC Enterprise must implement:
- Accept the new query parameter in their API
- Parse and filter records based on sync conditions
- Ensure pagination works correctly with filtered results
- Handle both `get_records` and `get_consents` endpoints

### Step 3: APSIS Implementation (Parallel with Step 2)

Two components need changes:

#### A. Generic Connector Logic

Modify the `get_records` and `get_consents` methods to:
- Check the installer option `supports_sync_conditions_as_query_param`
- If true: include sync conditions as query parameters in the request
- Include these conditions in ALL calls to fetch records/consents

**Estimated effort**: [Erik Andersson]: 2 days maximum for APSIS side

#### B. Remove Redundant Filtering Logic

Once sync conditions are pushed upstream, APSIS no longer needs to filter records:

**In the Full Sync Producer**:
- Skip condition verification logic in full sync if the feature is enabled
- This saves significant computational time

**Important caveat**: [Erik Andersson]:
> In the real time sync, they will still send webhooks for everything. In the first step you need to keep that in the real time sync, but in the full sync you don't need to have it.

So filtering logic removal is **only for full sync**, not real-time sync via webhooks.

**In the Delta Sync Manager**:
- Real-time webhook updates still need validation
- FSC Enterprise doesn't know about webhook filtering, so will continue sending all updates
- APSIS must validate webhook updates against sync conditions (until FSC Enterprise implements webhook filtering)

### Step 4: Mock Service Testing

[Erik Andersson]:
> When you want to test this is that you add this to you add support for this in the mock service. So the mock service accepts these conditions and then when you can deploy something manually to staging while you test this.

Before relying on the live FSC Enterprise endpoint:
- Update the mock service to accept and process sync condition query parameters
- Test the full flow locally/on staging
- Validate that filtering works correctly

### Step 5: FSC Enterprise Release to Production

FSC Enterprise team must:
- Complete implementation on their side
- Deploy to production
- Ensure the endpoint is stable and ready

### Step 6: Toggle Feature in Production

Once FSC Enterprise has released:
- Set the installer option `supports_sync_conditions_as_query_param = true` in production
- This activates APSIS to start sending sync conditions
- Release the APSIS changes to master branch

---

## Critical Testing and Risk Mitigation

### High-Risk Pitfall: Silent Data Loss

[Erik Andersson emphasized the danger]:
> This is a very easy pitfall to miss out on contacts and apps will have no idea that this is happening because we will just get contact IDs if things are missing from the CRM like we will be blind for it, so this needs to be heavily tested.

**Why this is dangerous**:
- If FSC Enterprise's filtering is wrong, APSIS will silently receive fewer contacts than it should
- No error is thrown — contacts are just missing
- APSIS won't know that the CRM filtered out records incorrectly
- Data consistency would be compromised without visibility

**Mitigation**:
- Collaborative testing between APSIS and FSC Enterprise teams
- Validate record counts match expectations
- Test with realistic sync condition combinations
- Monitor for discrepancies between CRM and APSIS contact counts

### Secondary Risk: Pagination Errors

[Lukasz Grabowski and Erik Andersson]:
> The pagination is on their side that they they have to make sure that they don't mess up that result.

Pagination logic becomes more complex when filtering:
- FSC Enterprise must correctly page through filtered results
- Offset-based pagination with server-side filtering can cause gaps
- Must ensure no duplicates or missing records across pages

---

## Endpoints and Specification Details

### Two Endpoints Requiring Changes

1. **`get_records`** — primary endpoint for fetching contact/record data during full sync
2. **`get_consents`** — for fetching consent data, also subject to sync condition filtering

Both endpoints must support the new `sync_conditions` query parameter.

### Conditions on Existing Query Parameters

The sync conditions parameter works alongside existing parameters:
- `fields`: Still specifies which attributes to return
- `page` and `page_size`: Still control pagination
- `IDs`: Still allows fetching specific records by ID (used for webhook callbacks)

The sync conditions add an additional filter on top of these.

---

## Current Sync Condition Examples

[Erik Andersson]:
> Typically [you have] active is like contact is active and like country equals something. Those are the two most common ones.

Realistic sync condition examples:
- `contact_is_active equals true`
- `country equals France`
- `segment equals premium_customers`

These are simple enough that query parameter encoding is not a practical concern.

---

## Timeline and Deliverables Summary

### Immediate Deliverables (This Session)

1. **Specification update** — document sync conditions query parameter format
2. **Communication to FSC Enterprise** — send specification and request implementation
3. **Story/task creation** — define APSIS-side implementation work

### Short Term (Before FSC Enterprise Release)

1. Update Generic Connector code to support the new parameter
2. Update mock service to accept sync condition parameters
3. Prepare testing plan

### Medium Term (FSC Enterprise Delivers)

1. Collaborative testing
2. Validate data integrity
3. Monitor for silent failures

### Release Phase

1. Enable the feature in production via the installer option
2. Monitor sync performance and data accuracy
3. Decommission old filtering logic if stable

---

## Key Takeaways

1. **Problem Scale**: Large customers (millions of contacts) are experiencing APSIS and CRM system crashes due to unnecessary data transfer from full sync.

2. **Solution Architecture**: Move sync condition filtering to the CRM system by sending conditions as query parameters on each request, rather than downloading and filtering all data in APSIS.

3. **Design Decision**: Use query parameters (not request body) for sync condition specifications, following REST conventions.

4. **Stateless API Handling**: FSC Enterprise's stateless API requires conditions to be sent with every request, unlike stateful systems that cache conditions.

5. **Implementation Sequencing**: FSC Enterprise must implement their side BEFORE APSIS releases the feature; wrong sequencing causes failures.

6. **Dual Endpoint Updates**: Both `get_records` and `get_consents` endpoints need the new sync conditions parameter.

7. **Configuration Flag Required**: Use installer option `supports_sync_conditions_as_query_param` to enable/disable the feature per system.

8. **Filtering Logic Asymmetry**: Full sync filtering can be removed from APSIS once upstream filtering is enabled, but real-time webhook validation must remain because FSC Enterprise won't filter webhooks initially.

9. **Silent Failure Risk**: Missing validation could silently lose contact records; this is the highest risk and requires extensive collaborative testing.

10. **Performance Impact**: Reducing 8 million downloads to 100,000 will dramatically reduce load on both APSIS and CRM systems.

11. **Effort Estimation**: ~2 days for APSIS implementation; bulk of work is on FSC Enterprise side.

12. **Testing Strategy**: Mock service must be updated first to allow local/staging testing before relying on live FSC Enterprise endpoint.

---

## Unresolved Questions and Action Items

### Action Items

1. **[Lukasz Grabowski]**: Update specification document with sync conditions query parameter format
2. **[Lukasz Grabowski]**: Send specification to Raisel (FSC Enterprise team)
3. **[Lukasz Grabowski/Erik Andersson]**: Create implementation stories/tasks with clear steps
4. **[Lukasz Grabowski]**: Update mock service to accept and process sync condition parameters
5. **[FSC Enterprise Team - Raisel]**: Implement sync condition query parameter support in `get_records` and `get_consents` endpoints
6. **[Lukasz Grabowski]**: Create collaborative testing plan with FSC Enterprise team
7. **[Team]**: Monitor for silent data loss during testing phases

### Clarifications Needed

- Exact format/syntax for complex sync conditions (already clarified: single AND-joined list)
- Confirmation of maximum sync conditions per customer (assumed ~2-3)
- FSC Enterprise pagination strategy with filtered results (delegated to their team)
- Timeline for FSC Enterprise implementation (not discussed, to be confirmed with Raisel)