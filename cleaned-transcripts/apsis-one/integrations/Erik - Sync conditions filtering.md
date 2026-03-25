---
source_file: Erik - Sync conditions filtering.txt
domain: Apsis One Integrations
topics: [Sync Conditions Filtering, Query Parameter Implementation, Full Sync Optimization, CRM System Performance, Generic Connector Specification]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Generic Connector, FSC Enterprise API, APSIS Full Sync, Delta Sync Manager, Full Sync Producer, Mock Service, Installer Options]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.0]
---

## Session Overview

This session focused on a critical performance optimization: moving **sync condition filtering upstream from APSIS to FSC Enterprise**. The team discussed how current implementation downloads all contacts from CRM systems and filters them in APSIS, causing severe performance degradation (example: 8 million contacts downloaded to use ~100,000). The solution involves passing sync conditions as query parameters in API requests, allowing FSC Enterprise to filter at the source. The team aligned on implementation approach, specification changes needed, and rollout strategy.

---

## Performance Problem Statement

### Current Architecture and Scale Issues

[Erik Andersson]: Today's sync conditions are evaluated inside APSIS. Because of this, we need to download every contact from the CRM systems to know if they should be added to APSIS or not.

This has not been a problem historically, but now there are French customers with 8 million contacts in their CRM system. The current approach is failing because:
- We download 8 million contacts
- We throw away 99% of them
- This crashes APSIS full syncs
- This crashes the CRM systems themselves

### Why Current Workarounds Exist

[Erik Andersson]: Microsoft Dynamics by site shop implementation has a **stateful implementation** where:
- The sync condition is stored locally in the CRM system
- When we request records during a full sync, they only return records matching the conditions
- Results are cached/paginated appropriately
- Same pattern applies to webhooks—they only send updates for matching contacts

The problem: **FSC Enterprise does not have a stateful API. It has a stateless API**, which means we cannot rely on the CRM maintaining filter state between requests.

---

## Proposed Solution: Sync Conditions as Query Parameters

### High-Level Approach

[Erik Andersson]: When making a request to the CRM system, we need to include sync conditions as either:
1. Query parameters (preferred)
2. Request body (alternative, less RESTful)

When FSC Enterprise receives the request with conditions, they will:
- Run a SQL query filtering based on provided sync conditions
- Return only matching results
- Handle pagination on their side

**Expected result**: 8 million contacts reduced to approximately 100,000.

### Why Query Parameters Over Request Body

[Lukasz Grabowski]: Using query parameters is more RESTful. The request body approach for filter parameters is non-standard for GET requests.

[Erik Andersson]: The primary concern with query parameters is URL length. Standard maximum is ~2000 characters, but:
- Sync conditions are typically limited (usually 2-3, e.g., "contact is active" AND "country equals something")
- Can be constrained further via UI
- Should not present a practical problem

### Query Parameter Format

[Erik Andersson]: The format would look like:
```
sync_conditions=first_name equals Erik AND last_name equals Andersson
```

This is unusual syntax for a query parameter but remains legally valid. The structure includes:
- Field name from the CRM system
- Operator (equals, etc.)
- Value

Currently only AND conditions are supported (implicit). OR does not yet exist in this model.

---

## Specification and Implementation Requirements

### Generic Connector Specification Updates

Specifications are stored in a public integration bucket (not in internal docs). The **get records method** for contacts needs to be extended.

Current parameters:
- Entity type (e.g., contacts for FSC Enterprise, persons for E-deal)
- Page size
- Page number
- Specific fields of interest
- Specific IDs (for webhook callbacks)

**New parameter to add**: Sync conditions (either as query parameter or in request body)

### Installer Option Flag

[Erik Andersson]: Need to add an installer option:
```
supports_sync_conditions_as_query_param: true/false
```

This flag indicates whether a given CRM system supports receiving sync conditions as query parameters. Implementation should:
- Only send conditions if this flag is true
- Cannot assume all systems support this feature
- Allows for graceful fallback

### Two Affected Endpoints

[Erik Andersson]: Two critical endpoints need changes:
1. **get_records** - for contact/entity synchronization
2. **get_consents** - for consent data synchronization

Both must support the sync conditions query parameter when the installer option is enabled.

---

## Implementation Workflow and Testing Strategy

### Phase 1: Specification Extension
- Extend the generic connector specification with the sync conditions query parameter
- Document the format and expectations
- This serves as the shared source of truth between APSIS and FSC Enterprise teams

### Phase 2: FSC Enterprise Implementation
[Erik Andersson]: FSC Enterprise team (Raisel/Rizel) must:
- Implement endpoint support for sync conditions as query parameters
- Ensure pagination works correctly with filtered results
- Validate that no duplicate contacts are returned (noted as easy pitfall from Tribe integration)

**Critical warning**: Missing contacts are dangerous because APSIS will be blind to them. APSIS receives contact IDs and has no way to detect if the CRM failed to return all matching records.

### Phase 3: APSIS Implementation (2 days estimated work)
[Erik Andersson]: On the APSIS side, anytime you make a get request for records, include the sync conditions if supported.

Implementation involves:
- Sending sync conditions as query parameters in the get_records call
- Sending sync conditions in the get_consents call
- Checking the installer flag before sending conditions

### Phase 4: Filtering Logic Removal
[Lukasz Grabowski]: Once FSC Enterprise has the feature deployed, remove sync condition filtering logic from the full sync:

**For Full Sync**:
- Skip condition verification in Full Sync Producer (significant computational savings)
- Condition filtering is no longer needed because CRM returns only matching contacts

**For Real-Time Sync (webhooks)**:
- **Keep** condition filtering in Delta Sync Manager initially
- Reason: FSC Enterprise doesn't know about sync conditions when sending webhooks proactively
- They will continue sending webhook updates for all contact changes
- APSIS must validate webhook payloads match sync conditions

### Phase 5: Testing with Mock Service
[Erik Andersson]: Before FSC Enterprise releases to production:
- Add sync conditions support to the mock service
- Deploy changes manually to staging environment
- Test filtering behavior thoroughly

### Phase 6: Release Coordination
Release sequence:
1. FSC Enterprise deploys sync conditions query parameter support to production
2. APSIS adds the feature to master branch and releases (only after FSC Enterprise is ready)
3. Enable the `supports_sync_conditions_as_query_param` installer option
4. APSIS begins utilizing the feature

[Lukasz Grabowski]: Alternative option: APSIS could send query parameters before FSC Enterprise handles them (they would simply ignore unknown parameters), but it's cleaner to wait for FSC Enterprise readiness.

---

## Design Alternatives Considered

### Alternative: Attribute-Based Query Parameters
[Erik Andersson]: Instead of sending sync conditions as structured data:
```
sync_conditions=first_name equals Erik
```

Could send individual query parameters:
```
?first_name=Erik&last_name=Andersson
```

This treats filter parameters like individual attributes, more aligned with generic REST patterns.

**Why rejected**: [Lukasz Grabowski] The structured `sync_conditions` approach is preferred because:
- More explicit that we're sending sync conditions, not arbitrary filters
- Clearer intent
- Better documented
- Less ambiguous than relying on query parameter names to mean filtering

---

## Performance and Resource Impact

### Computational Savings

[Erik Andersson]: The implementation saves computational time in two ways:
1. **Network bandwidth**: Only relevant contacts transmitted from CRM
2. **APSIS processing**: Full Sync Producer no longer needs to verify sync conditions after receiving data

### Scalability Improvement

Moving from downloading 8 million contacts to 100,000 addresses the immediate crisis (French customer). More broadly, this allows APSIS to scale to CRM systems with massive contact databases without performance degradation.

---

## Risk Factors and Caveats

### Missing Contact Detection Blind Spot
[Erik Andersson]: If FSC Enterprise's filtered result is incomplete (missing records that should match the sync condition), APSIS receives only contact IDs and has no mechanism to detect the gap. This is a critical testing concern.

Mitigation: Collaborative testing between teams with heavy validation of result counts and coverage.

### Webhook Filtering Complexity
[Erik Andersson]: During real-time sync, FSC Enterprise cannot pre-filter webhooks because they don't know the sync conditions at webhook trigger time. APSIS must continue filtering webhook payloads in Delta Sync Manager. This creates potential for:
- Synchronizing contacts that shouldn't be synced
- Missing contacts that should be synced
- Inconsistency between full sync and incremental updates

Mitigation: Delta Sync Manager condition filtering remains active; condition removal only applies to full sync.

### Historical Precedent: Tribe Integration
[Erik Andersson]: Similar duplicate/missing contact issues occurred with Tribe integration. This points to difficulty in coordinating filtered sync across systems. Requires careful testing.

---

## Implementation Checklist

1. **Extend Generic Connector specification** with sync_conditions query parameter
2. **Create installer option** `supports_sync_conditions_as_query_param` for FSC Enterprise connector
3. **Update get_records endpoint** to include sync conditions as query parameters
4. **Update get_consents endpoint** to include sync conditions as query parameters
5. **Add support to mock service** for testing
6. **FSC Enterprise implementation**: Accept and filter by sync_conditions query parameter
7. **Remove filtering logic** from Full Sync Producer (full sync only, keep for Delta Sync Manager)
8. **Comprehensive testing** on staging with mock service before production FSC Enterprise release
9. **Release FSC Enterprise changes to production**
10. **Release APSIS changes to master and production**
11. **Enable installer option** for production customers

---

## Key Takeaways

- **Scale problem is real**: 8 million CRM contacts creating cascading failures in both APSIS and CRM systems
- **Stateless API limitation**: FSC Enterprise's stateless API means filtering cannot be stored server-side; must be passed per-request
- **Query parameters approach**: Preferred solution despite unusual format; URL length constraints are not a practical concern
- **Architectural split**: Full sync filtering moves upstream; real-time (webhook) filtering remains in APSIS Delta Sync Manager
- **Testing criticality**: Missing contacts from filtered results are invisible to APSIS; heavy validation required to prevent silent data loss
- **Specification-first approach**: Extending generic connector spec is the shared source of truth; both teams build against it
- **Release coordination required**: FSC Enterprise must deploy first; APSIS waits for production readiness before enabling the feature
- **Estimated effort**: ~2 days for APSIS side; significant work on FSC Enterprise side with heavy testing collaboration needed

---

## Unresolved Questions and Action Items

### For Erik Andersson:
- Send sync conditions format examples to chat/documentation (detailed format for complex conditions)
- Send endpoint specifications to chat (get_records and get_consents details)

### For Lukasz Grabowski:
- Create story/stories for implementation phases
- Coordinate with FSC Enterprise team (Raisel/Rizel) on specification handoff
- Plan testing strategy with Erik before staging deployment

### For FSC Enterprise Team:
- Implement sync_conditions query parameter support in get_records endpoint
- Implement sync_conditions query parameter support in get_consents endpoint
- Validate pagination and result completeness
- Release to production before APSIS changes go live

### Open Design Questions:
- Exact syntax for complex sync conditions with multiple fields (implicit AND assumption confirmed, but needs documentation)
- Behavior when sync conditions length approaches 2000 character limit (edge case, likely won't occur in practice)
- Whether to allow early APSIS deployment before FSC Enterprise is ready (decided: cleaner to wait, but technically possible)