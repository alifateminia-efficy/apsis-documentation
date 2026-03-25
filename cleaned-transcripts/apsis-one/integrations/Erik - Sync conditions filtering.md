---
source_file: Erik - Sync conditions filtering.txt
domain: Apsis One Integrations
topics: [Sync Conditions Filtering, Full Sync Optimization, Query Parameter Implementation, CRM System Performance, Generic Connector Specification]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Generic Connector, FSC Enterprise, Microsoft Dynamics, APSIS, Mock Service, Delta Sync Manager, Full Sync Producer]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This knowledge transfer session focused on a critical optimization: moving **sync conditions filtering upstream to FSC Enterprise** to reduce the computational load of full syncs. The core problem is that APSIS currently evaluates sync conditions locally after downloading all contacts from the CRM system, which becomes infeasible for customers with millions of records (e.g., a French customer with 8 million contacts in their CRM). The team discussed the technical approach for passing sync conditions as query parameters to the CRM API, the specification changes needed, the implementation roadmap, and the testing requirements to ensure data integrity.

---

## The Problem: Current Sync Conditions Architecture

### Performance Crisis at Scale

[Erik Andersson]: The sync conditions are evaluated inside of APSIS today, and because they are evaluated inside of APSIS, it means that we need to download every contact from the CRM systems to know if they should be added to APSIS or not. This has not been a problem up until recently, but now we have French customers that have 8 million contacts in their CRM system and like this is not working with the way it is set up because we need to download 8 million contacts and then like at least 8 million concerns and then throw away 99% of it. It's like it's ridiculous and it is crashing APSIS full syncs and it is crashing the CRM systems.

### Current Business Logic

Today, sync conditions operate with an implicit **AND logic only** — there is no support for OR conditions yet. Typical sync conditions include:
- `contact is active`
- `country equals <value>`

These are the two most common filtering criteria that customers define.

---

## Current Stateful Implementation: Microsoft Dynamics Reference

[Erik Andersson]: The Microsoft Dynamics by site shop implementation has a stateful implementation where we store the sync condition locally and we send them the sync condition and when we ask for records in the full sync, they only give us the ones that are matching, like they have prepared this cached or paginated it or however they have solved it. But we only get what we should have. Same thing for webhooks — they are only sending us updates for contacts that are matching.

This proven approach demonstrates the desired outcome: the CRM system becomes aware of the sync conditions and only returns matching records.

---

## The Challenge: FSC Enterprise's Stateless API

[Erik Andersson]: The issue is that this doesn't exist in FSC Enterprise. FSC Enterprise does not have a stateful API; they have a **stateless API**, which means we need to do some minor tweaks. When we make a request to the CRM system, we should include the sync conditions as a query parameter or in the request body.

### Query Parameter vs Request Body

The team debated two approaches for passing sync conditions to FSC Enterprise:

**Option 1: Query Parameters (Preferred)**
- Adheres to REST conventions
- Allows CRM system to filter during the query itself
- Risk: URL length constraints (standard maximum ~2000 characters)

[Lukasz Grabowski]: Why are we worried about it because of the length of the URL?

[Erik Andersson]: I think it's 2000 by standard, but it depends on the limits of the CRM system. I don't think this is gonna be a problem in all honesty, because typically you have active is like contact is active and like country equals something. Those are the two most common ones.

[Lukasz Grabowski]: I think we can put some limitation like or written it down somewhere or just because we have we can define this conditions from the UI, right. So we can also limit the UI.

**Option 2: Request Body**
- Safer regarding URL length
- Not standard practice for GET/filter requests
- Less RESTful

**Decision: Query Parameters** — The team chose Option 1 because it maintains REST conventions and the typical number of sync conditions (2-3) will not exceed URL length limits.

---

## Proposed Sync Conditions Query Parameter Format

The sync conditions will be passed as a structured query parameter. The format combines all AND conditions into a single parameter:

```
syncConditions=firstName:equals:Eric,lastName:equals:Andersson,country:equals:Sweden
```

[Erik Andersson]: Sync condition and then we would like in theory like we would like to have it... like this, essentially... And like then this would look like this. And this is a very weird format for a query parameter, but I think it is still legal syntax wise.

[Lukasz Grabowski]: Yeah, but these are once again these are and or we are OK and this and and that's fine and this is what we we are aiming for if only and that's that's OK... Yeah, I think the more verbose is the first one, same conditions and first name, last name and et cetera.

The verbose format was preferred because it makes the intent explicit: these are sync conditions being passed to filter records, not arbitrary query parameters.

---

## Generic Connector Specification Update

### Current Get Records Method

The existing `getRecords` method in the generic connector already supports several parameters:

```
- entity: Which entity to download (e.g., "contacts" for FSC Enterprise, "persons" for E-deal)
- pageSize: Number of records per page
- page: Page number for pagination
- fields: Which attributes/properties are of interest
- IDs: Optional parameter for webhook callbacks to get data for specific updated profiles
```

### Required Additions

Two endpoints need to be extended:

1. **Get Records endpoint** — for full sync and ongoing data retrieval
2. **Get Consents endpoint** — for consent data retrieval

Both must support passing sync conditions as query parameters when the feature is enabled.

[Lukasz Grabowski]: OK, so we need to extend the specification, right? And give this, give this specification to enterprise guys to implement this.

---

## Implementation Strategy and Installer Option Flag

### New Installer Option

A new boolean flag will be added to the generic connector configuration:

```
supportsyncConditionsAsQueryParam: boolean
```

This flag indicates whether the specific CRM system (in this case, FSC Enterprise) supports receiving and processing sync conditions in the query parameters of API requests.

[Erik Andersson]: So here under the installer option here you would say supports... supports sync conditions as query param... Meters true.

### Conditional Logic in Code

When this flag is `true`:
- The generic connector will include sync conditions in API requests to `getRecords` and `getConsents`
- The local filtering logic in APSIS will be **disabled for full syncs** (optimization)
- Filtering will still occur for real-time syncs via webhooks because FSC Enterprise cannot retroactively apply sync conditions to webhook events it doesn't know about in advance

[Erik Andersson]: In the full sync, yes, because for enterprise they will still send webhooks for everything. In the first step you need... you need to keep that in the real time sync, but in the full sync you don't need to have it and it's in the full sync where you will have [the performance gains]... For enterprise they will still send webhook updates still and we need to validate that in the Delta Sync Manager, but that's separate because they have no idea about when they are about to send a webhook.

---

## Two-Phase Local Filtering Behavior

### Phase 1: Full Sync (After CRM-Side Filtering)

When the sync conditions are pushed upstream and the flag is enabled:
- APSIS receives only records matching the sync conditions from the CRM
- **Condition verification in Full Sync Producer is disabled** — no need to re-filter
- Significant computational savings

### Phase 2: Real-Time Sync via Webhooks

- FSC Enterprise will continue to send webhook events for **all** contact updates (it has no awareness of which conditions apply)
- **Delta Sync Manager must still validate incoming webhook updates** against the configured sync conditions
- This prevents non-matching records from being inadvertently added to APSIS

[Erik Andersson]: The full sync producer. You don't need to verify this anymore if you have this enabled and there you will save quite some computational time as well.

### Critical Testing Requirement

[Erik Andersson]: The the beauty of all of this is like this is not that much work to do in appsec side. I think honestly you can solve this in like 2 days tops. The the absolute heaviest work will be an enterprise and then it is a collaborative task to make sure that you are still getting the correct amount of profiles and you are not getting duplicates as we are doing with tribe because this is a very easy pitfall to miss out on contacts and apps will have no idea that this is happening because we will just get contact IDs if things are missing from the CRM like we will be blind for it, so this needs to be heavily tested.

---

## Implementation Roadmap

### Step 1: Update Generic Connector Specification
- Extend the specification to document the new `syncConditions` query parameter format
- This specification serves as the **shared source of truth** for both the APSIS team and the FSC Enterprise team
- Document the format, expected behavior, and pagination requirements

### Step 2: CRM System Implementation (FSC Enterprise Team)
- Raisel (or "Shell") and the FSC Enterprise team implement the sync conditions parameter handling in their API
- When `getRecords` or `getConsents` requests include the `syncConditions` parameter, FSC Enterprise filters results server-side before returning
- Ensure pagination works correctly with the filtered result set
- Thoroughly test that no records are duplicated and no matching records are dropped

### Step 3: APSIS Generic Connector Testing
- Implement support for the new query parameter in the mock service
- Deploy the mock service implementation to staging for manual testing
- Verify that the sync conditions parameter is correctly formatted and passed

### Step 4: FSC Enterprise Production Release
- FSC Enterprise releases the feature to production
- This must happen **before** APSIS releases its changes to production
- Otherwise, the sync conditions parameter will be sent to an endpoint that doesn't recognize it (no harm, but ineffective)

### Step 5: APSIS Production Release
- Merge the sync conditions query parameter support into the master branch of the generic connector
- Add the `supportsyncConditionsAsQueryParam: true` flag to the FSC Enterprise installer configuration
- Release to production
- APSIS will now begin utilizing sync conditions filtering on the FSC Enterprise connector

[Lukasz Grabowski]: So first we should extend specification right and then give this specification to... who to Rizel... Then of course you need to implement the logic. So anytime you make a get request for the records, you need to include this... And then after this we can release our change and we need to add this flag to the generate connector to the enterprise... Yeah, to the enterprise connector, right?

---

## Expected Performance Improvements

### Baseline: Current State (8 Million Contact Customer)
- Downloads: 8 million contacts from CRM
- Filtering: Evaluates 8 million records in APSIS
- Result: Keeps ~100,000 matching contacts
- Waste: 99% of downloaded data discarded
- Impact: Crashes full syncs, overloads CRM systems

### Target State (After Upstream Filtering)
- CRM receives: Sync conditions query parameter
- Downloads: ~100,000 matching contacts only
- Filtering: Only matching records returned from CRM
- Result: ~100,000 contacts added to APSIS
- Waste: Minimal; filtering happens at source
- Impact: Full syncs complete efficiently, CRM systems not overloaded

[Erik Andersson]: This should reduce like the 8 million contacts in the sync down to approximately 100,000 instead.

---

## Mock Service and Testing Strategy

During development, the team will leverage the mock service to test sync conditions filtering before FSC Enterprise has fully implemented support:

[Erik Andersson]: When you want to test this is that you add this to you add support for this in the mock service. So the mock service accepts these conditions and then when you can deploy something manually to staging while you while you test this...

This allows APSIS developers to:
- Test the query parameter formatting logic
- Verify that the generic connector correctly passes sync conditions
- Prepare for integration testing once the CRM system is ready

---

## Specification and Documentation Requirements

The team agreed that the specification updates must be documented in the shared generic connector specification (currently in the public integration bucket, not in internal docs staging).

[Lukasz Grabowski]: OK, can you send it on the chat like this this two and this notes you you have in your in your visual code?

Key items to document:
1. The two endpoints: `getRecords` and `getConsents`
2. The `syncConditions` query parameter format and semantics
3. The new installer option flag: `supportsyncConditionsAsQueryParam`
4. Expected behavior: pagination, result set size reduction, filtering precedence

---

## Key Takeaways

1. **Problem Scope**: Current sync conditions evaluation is inefficient for customers with millions of CRM records (8 million → 100,000 filtered result = 99% waste). This crashes both APSIS and CRM systems.

2. **Solution Approach**: Push sync condition evaluation upstream to FSC Enterprise's API layer by passing conditions as query parameters, allowing server-side filtering before data transfer.

3. **Reference Implementation**: Microsoft Dynamics already implements this pattern with a stateful architecture; FSC Enterprise will implement it with a stateless API approach using query parameters.

4. **Technical Decision**: Query parameters are preferred over request body because they are RESTful and the typical 2-3 sync conditions per customer won't exceed URL length limits (~2000 characters).

5. **Implementation Phases**:
   - Full syncs: Disable local filtering once upstream filtering is enabled (major performance gain)
   - Webhooks: Keep local validation in Delta Sync Manager (FSC Enterprise has no awareness of sync conditions for real-time events)

6. **Release Sequence**: Specification → FSC Enterprise implementation → APSIS testing → FSC Enterprise production → APSIS production release.

7. **Critical Risk**: Data loss is a silent failure mode—if sync conditions are not correctly implemented on the FSC Enterprise side, APSIS will be blind to missing records. Extensive testing and validation are mandatory.

8. **Estimated Effort**: ~2 days for APSIS side implementation; heavier lifting on FSC Enterprise side due to API changes and testing complexity.

---

## Unresolved Questions and Action Items

- [ ] Erik Andersson: Send detailed notes on the two endpoint changes (`getRecords` and `getConsents`) to the team via chat
- [ ] Lukasz Grabowski: Create Jira stories capturing the implementation steps once Erik's notes are received
- [ ] Team: Update the generic connector specification with the new `syncConditions` query parameter format
- [ ] Specification to be delivered to: Raisel/Shell (FSC Enterprise team lead)
- [ ] FSC Enterprise team: Implement sync conditions filtering on their API endpoints
- [ ] FSC Enterprise team: Release feature to production before APSIS's production release
- [ ] APSIS team: Implement mock service support for sync conditions parameter
- [ ] APSIS team: Staging manual testing with mock service
- [ ] APSIS team: Add `supportsyncConditionsAsQueryParam: true` to FSC Enterprise installer configuration
- [ ] Cross-team validation: Coordinate testing to ensure no duplicate records or missing data after upstream filtering is enabled