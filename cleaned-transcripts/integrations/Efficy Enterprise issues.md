---
source_file: Efficy Enterprise issues.txt
domain: Integrations
topics: 
  - Full sync performance issues with large datasets
  - Sync conditions and filtering architecture
  - Consent data handling and memory optimization
  - On-premise deployment constraints
  - CRM system pagination performance bottlenecks
  - Generic connector capabilities
  - Support ticket routing and process management
speakers:
  - Erik Andersson (Integration Developer/Specialist)
  - Lukasz Grabowski (Engineering Lead/Manager)
  - Tomasz Kowalski (Team Member)
  - Michal Rosikiewicz (Team Member)
key_components:
  - FSC Enterprise 12.1
  - Generic Connector
  - Dynamics Side Shop (legacy)
  - Sync Conditions (v1)
  - Consent Mappings
  - Full Sync Consumer
  - APSIS Platform
  - Audience (subscription export)
  - SQS (message queue)
session_type: debugging-session
---

## Session Overview

This knowledge transfer session focuses on critical performance issues discovered with the Efficy Enterprise integration when handling unusually large customer datasets. The team discusses a specific case with 8 million contacts (4x larger than any previously tested customer) that is causing full sync failures, particularly around consent data retrieval. The root cause analysis reveals architectural limitations in how sync conditions are implemented, and the discussion covers both immediate workarounds and longer-term solutions requiring coordination with the FSC Enterprise team and product management.

---

## Customer Issues and Investigation Methodology

### Problem Statement

Three customers were identified as experiencing slow or failing full syncs. [Erik Andersson]: The most significant issue involves a customer with 8 million contacts in their CRM system, which is substantially larger than the 2 million contacts used in previous load testing. [Erik Andersson]: One customer has an on-premise FSC Enterprise installation with extensive customizations, making it difficult to isolate whether integration issues are caused by the APSIS connector or their custom setup.

### Identifying Customer Integrations

[Lukasz Grabowski]: When investigating these issues, the team needs to determine which integration a customer uses. [Erik Andersson]: The process involves starting with the customer display name and navigating to the APSIS back office to obtain the account ID. The next step requires querying the database, specifically the `installations` table, which serves as the source of truth for integration configurations.

The query structure is:
```
SELECT * FROM installations WHERE ID = [account_id]
```

> [Erik Andersson]: This must use the actual ID, not the display name. The `installations` table is the complete repository showing which integration is installed on which section and account.

For the affected customer, the query revealed: `FSC Enterprise 2` installation.

### On-Premise vs. Managed Deployments

[Erik Andersson]: The team has no way to determine from the integration side whether a deployment is on-premise or managed by FSC. This distinction matters because on-premise installations typically present different constraints.

[Erik Andersson]: A common constraint with on-premise deployments is IP whitelisting and public access requirements. Many customers request APSIS integration for on-premise FSC instances but resist opening public access to their systems.

> [Erik Andersson]: APSIS requires public Internet accessibility to on-premise environments. We do not support VPN tunnels for on-premise FSC deployments (unlike the Pro tier) because there is no internal team to maintain or support them. Without public access, on-premise integration is not feasible.

[Tomasz Kowalski]: Clarifying question about whether IP whitelisting rules need to be updated via the Squid proxy. [Erik Andersson]: IP whitelisting rules are configured at installation time, but they are separate from VPN tunnel requirements. The Squid proxy is not involved in VPN tunnel scenarios.

---

## Full Sync Architecture and Performance Baseline

### Historical Load Testing

[Erik Andersson]: Previous full sync load testing was conducted with 2 million contacts, which at that time represented the largest customer deployment. Testing showed that syncs took considerable time but completed successfully without crashes.

The current customer with 8 million contacts represents a 4x increase over any previously tested scenario, creating unknown performance characteristics.

### Data Volume Implications

With the affected customer:
- **Contact data**: 8 million total contacts in the CRM system
- **Synchronizable subset**: 1.2 million contacts across multiple sections, distributed via sync conditions
- **Per-section limit**: Approximately 100,000 contacts max per section
- **Consent entries**: Each contact typically has at least one consent; some customers have mapped up to 20 consent lists

---

## Sync Conditions: Current Architecture and Limitations

### How Sync Conditions Work Today

[Erik Andersson]: Sync conditions allow configuration of selective synchronization. For example, a sync condition might specify: "only sync contacts where email matches eric@absisfake.com".

**Critical architectural limitation**: Sync condition filtering happens on the APSIS side, not the CRM side.

> [Erik Andersson]: Even though a sync condition is configured, the full sync must still download ALL data from the CRM system and then filter it locally. If a CRM system presents 8 million contacts and only 1.2 million match the sync conditions, APSIS must still download and process all 8 million before filtering.

This applies to both contacts and their associated consent data:
- 8 million contact downloads occur
- 8 million consent entries must be retrieved
- Combined data volume creates enormous processing burden

### Pagination and Data Handling

[Erik Andersson]: The customer's full sync fetched 16,000 pages of profiles during pagination. While contact pagination handled this without crashing, consent pagination revealed critical performance issues.

When attempting to retrieve just 500 consent entries, the system received HTTP 504 Gateway Timeout responses. Testing revealed that 500 consent entries required 22 seconds to download from the CRM system.

**Calculation**: 8 million contacts ÷ 500 (batch size) × 22 seconds = approximately 352,000 seconds (97+ hours) just for consent retrieval.

> [Erik Andersson]: The sync started at 4 AM and by 11 AM (7 hours later) it was still receiving timeouts and had only progressed through a fraction of the required consent data.

### Root Cause Analysis

[Erik Andersson]: The FSC Enterprise system appears to have a proxy server in front of its business logic. The team is receiving HTML 504 error responses rather than API-level errors, indicating the proxy is timing out before the backend can respond.

> [Erik Andersson]: Contact downloads worked successfully (though time-consuming). Consent downloads are not working as expected because the CRM system has apparent pagination or caching implementation issues causing extremely slow response times.

---

## Impact on Audience Export Comparison

### Current Optimization Strategy

[Erik Andersson]: During full sync, there is an optimization step for consents:
1. The system exports all subscriptions from APSIS that have consent mappings
2. This export is loaded entirely into memory
3. For each consent received from the CRM, the system compares it against the in-memory export
4. Only changed consent statuses result in update messages; unchanged entries are skipped

This optimization works well for current customer sizes.

### Memory Risk with Large Datasets

[Erik Andersson]: If a customer has 7-8 million profile entries in APSIS, each with multiple consent entries, attempting to load the entire audience export into memory for comparison will likely cause the sync to crash due to memory exhaustion.

> [Erik Andersson]: If we need to support customers of this size, removing this optimization is probably necessary. We should process all CRM system data (which is streamed in) directly without the in-memory comparison against audience exports.

The trade-off: 
- **Current approach**: Uses memory optimization but fails with very large datasets
- **Proposed approach**: Stream all CRM data directly to SQS for consumer handling (this data is already streamed), eliminating the in-memory audience export bottleneck

[Erik Andersson]: For the current customer, this isn't the immediate breaking issue because the audience has no existing consent data, so the export is empty. The breaking issue is the CRM system gateway timeouts on consent retrieval itself.

---

## Solution: Server-Side Sync Condition Filtering

### The Side Shop Precedent

[Erik Andersson]: For the Dynamics Connector and Site Shop integration, a new feature was developed to address similar challenges. Instead of APSIS requesting all contacts and filtering them locally, the sync conditions are sent to Site Shop, which performs filtering server-side before returning results.

**Flow comparison**:
- **Current approach (FSC Enterprise)**: APSIS requests all 8 million contacts; CRM returns all 8 million; APSIS filters to 1.2 million
- **Side Shop approach**: APSIS sends sync conditions to Side Shop; Side Shop returns only the 1.2 million matching contacts

> [Erik Andersson]: When Site Shop handles filtering, a full sync that would normally process 1 million+ contacts to discard most of them becomes incredibly fast because only relevant data is transmitted.

### Generic Connector Support

[Lukasz Grabowski]: Clarification that Dynamics Side Shop is actually the generic connector. [Erik Andersson]: Confirmed: the generic connector supports this capability.

```
Connector supports sync condition: true
```

[Erik Andersson]: FSC Enterprise 12.1 uses the generic connector, so the configuration in APSIS is straightforward. The prerequisite is that FSC Enterprise must expose endpoints to support this functionality.

### Required CRM System Endpoints

[Erik Andersson]: The CRM system needs to implement an endpoint that receives sync conditions. When registering sync conditions per section, the payload includes:

```
V1 Sync Condition Payload:
- Entity type (e.g., Contact, Case)
- Delete request handling (whether to request deletion if contact no longer matches)
- Conditions array:
  - Field name (CRM system field name, e.g., "FirstName")
  - Operator and value (e.g., "equals 'Elsa'")
- Additional entity-specific conditions (e.g., "SyncToAPSIS equals true")
```

For consents specifically: Only consents belonging to contacts that match sync conditions should be presented.

### Implementation Complexity

[Erik Andersson]: This is not trivial to implement on the CRM side. They must handle:
- Pagination updates as sync conditions change
- Cache invalidation
- Real-time updates or periodic refreshes
- Potentially complex SQL integration depending on their architecture

> [Erik Andersson]: The heavy work falls on their side; APSIS receives the benefit of not processing unnecessary data. They need some kind of ready cache to post to us, or they need to incorporate sync conditions into SQL statements efficiently.

---

## Configuration Architecture: Where Sync Conditions Live

### Current Design Decision

[Erik Andersson]: Sync conditions are configured in APSIS back office, not in the CRM system itself. This design was a compromise during Maxo development.

**Maxo requirements**:
- Teams wanted field mappings configured in Maxo
- Teams wanted sync conditions configured in Maxo
- No one wanted to invest in building UI and backend support in the CRM system

**Result**: Configuration happens in APSIS, but enforcement happens on the CRM side. When sync conditions are updated in APSIS, a request is sent to the CRM system to enforce the new filtering.

### Ideally: Sync Conditions in the CRM System

[Erik Andersson]: In an ideal architecture, sync conditions would be configured directly in the CRM system because the CRM has access to rich data that APSIS does not, such as:

> [Erik Andersson]: A customer might want to sync only contacts belonging to companies with annual revenue exceeding 500,000 crowns. This organizational data isn't accessible in APSIS because related entity data is not supported. Alternatively, they might want to sync only contacts with at least one consent entry—very common request but not feasible in APSIS.

[Lukasz Grabowski]: If sync condition filtering is enabled on the CRM side and the property is changed in the generic connector, the front-end UI will still display the filter options, but they won't actually be used on the APSIS side—correct?

[Erik Andersson]: Yes. You configure the sync conditions in APSIS, and when configured, APSIS makes a request to the CRM system. The CRM system enforces the filtering. This is a middle ground because of legacy decisions, but it works.

### Historical Context

[Erik Andersson]: When Maxo was being developed, there was significant back-and-forth about where configuration should live. Everyone wanted to move configuration away from APSIS into Maxo, but no one wanted to actually implement the UI and backend logic in the CRM system because of resource constraints and prioritization issues.

> [Erik Andersson]: In the best case, you'd configure everything in one place. Today, field mappings and subscription mappings are still configured in APSIS, but from a logical perspective, it would make more sense to have sync conditions configured in the CRM system where the data actually lives.

---

## Path Forward: Implementation Steps

### Step 1: Communicate with Product Management

[Erik Andersson]: The first step is discussion with Operi (Product/PS team) to set expectations and present the issue. [Lukasz Grabowski]: Agreement that product management should be involved early to understand the scope.

> [Erik Andersson]: We need to set expectations so this doesn't get out of hand. We can say: "We tested this with 2 million contacts and it worked, but now with large customers moving forward, we need this fix because the CRM system is showing stress."

### Step 2: Cross-Organizational Coordination

[Lukasz Grabowski]: Product should then coordinate with FSC Enterprise to highlight the issue and agree on a solution. The challenge is that while APSIS implementation appears straightforward (just enabling the sync condition property on the generic connector), the CRM side requires significant work.

**Challenges in cross-organizational efforts**:
- Each organization wants to protect their own time
- Commitment resistance is typical
- Getting buy-in requires framing the issue effectively

### Step 3: Framing the Issue for FSC Enterprise

[Erik Andersson]: The case can be made on multiple dimensions:

1. **Cost/Resource**: This wastes enormous computing and processing time for both APSIS and FSC Enterprise
2. **Stability**: The CRM system is clearly experiencing stress (gateway timeouts), so optimization benefits their stability
3. **Customer Experience**: Full syncs would complete in 20 minutes instead of 1.5+ days, resulting in much happier customers
4. **Mutual Benefit**: Less stress on FSC Enterprise systems, faster syncs, happier customers

> [Erik Andersson]: We can present it as money issue and stability issue. We're putting less stress on their system, customers get faster syncs, and we use less computing resources.

### Step 4: Support Ticket Management

[Erik Andersson]: There is a ticket in Submariner (support ticketing system) for this issue. The team should not proactively work on this until product/support makes a formal request.

[Erik Andersson]: The team needs to be disciplined about support channel routing. The issue is that support has been overwhelmed and not responding quickly, leading to escalations through alternative channels (internal messages, product help channel, etc.).

> [Erik Andersson]: This teaches people to expect R&D to proactively pick up work, which sets a bad precedent. If support response times are the problem, the solution is scaling support, not having R&D bypass the formal process. We need a reliable, unified flow for support cases.

---

## Immediate Mitigations and Optimizations

### Short-Term: Remove In-Memory Audience Export Optimization

[Erik Andersson]: To prevent future crashes when customers have large consent datasets already in APSIS, the in-memory audience export comparison during full sync should be removed.

> [Erik Andersson]: This is not the breaking issue right now (because we can't even get the consents into APSIS due to CRM timeouts), but if FSC Enterprise fixes their pagination issues and we do get 16 million consent entries, the next full sync with existing APSIS audience data would likely crash due to memory exhaustion. By processing everything streamed from the CRM instead, we're safer going forward.

### Medium-Term: Engage FSC Enterprise on Pagination Optimization

[Erik Andersson]: While server-side sync condition filtering is the real solution, there might be simple optimizations FSC Enterprise can make to their pagination or caching to improve performance.

> [Erik Andersson]: Contact downloads are working fine (though taking 1.5+ hours for 8 million profiles). As soon as consent retrieval begins, "all hell breaks loose" due to pagination timeouts. If FSC Enterprise optimizes their pagination or caching, we might prevent syncs from taking two days, even without full sync condition filtering.

This could serve as a quick win while waiting for the larger architectural change.

---

## Support Process and Workflow Issues

### The Alternative Channel Problem

[Erik Andersson]: Multiple team members have been circumventing the formal support ticket process and contacting Erik directly through internal channels, product help channels, and emails. This creates several problems:

> [Erik Andersson]: I will not reply to internal messages. People are trying to circumvent the formal flow, and this is particularly bad because then you don't get to see what I'm doing. I'm steering everything to the help channel regardless of where it comes from. If this means I will frustrate people, sure, let them be frustrated, but we need one unified flow.

**Principles**:
- Formal support tickets are the single source of truth
- R&D should not monitor multiple informal channels
- Bypassing support creates bad precedent for future requesters
- One unified process prevents abuse of alternative routes

[Lukasz Grabowski]: Agrees that the formal process must be respected. The team will leave issues in the product help channel and wait for product/support to escalate formally rather than investigating proactively.

---

## Key Takeaways

1. **Scale Problem is Real**: With 8 million contacts and associated consent data, the current architecture breaks because all data must be downloaded to APSIS for local filtering, even though only 1.2 million should be synced.

2. **Sync Conditions Must Be Server-Side**: The generic connector and FSC Enterprise 12.1 technically support sending sync conditions to the CRM, but FSC Enterprise must implement endpoints and filtering logic to use them. This is the primary solution.

3. **Consent Retrieval is the Bottleneck**: Contact pagination works (though slow); consent pagination times out due to FSC Enterprise's pagination/caching implementation. Server-side filtering would dramatically reduce consent retrieval volume.

4. **Memory Optimization Must Be Removed**: The current in-memory audience export comparison during full sync should be removed to prevent memory exhaustion crashes when handling large datasets.

5. **Documentation Gap**: Platform limitations regarding dataset size should be documented (likely in GitHub or platform docs) so expectations are clear.

6. **Cross-Org Coordination Required**: The solution requires product/sales to work with FSC Enterprise to prioritize endpoint implementation. Cost, stability, and customer satisfaction are the primary levers.

7. **Support Process is Critical**: Formal ticket routing through product help/Submariner channels must be enforced. Alternative channels create unsustainable precedent and process chaos.

8. **Generic Connector Readiness**: APSIS side is ready—setting the sync condition property to true is straightforward. The blocker is entirely on FSC Enterprise's side.

---

## Unresolved Questions and Action Items

### Immediate Actions
- **Erik Andersson to discuss with Operi (Product)**: Present the issue, data volumes, and proposed solution path
- **Escalate to Product Team**: For formal coordination with FSC Enterprise about endpoint implementation
- **Documentation**: Add platform limitations regarding maximum contact volume to official documentation

### Technical Questions Pending
- What are FSC Enterprise's pagination implementation details? (They appear to have a proxy timeout issue)
- Can FSC Enterprise implement simple caching optimizations before full endpoint support?
- Timeline for FSC Enterprise to implement sync condition endpoint and filtering?

### Process Decisions
- How will the team handle future large-customer onboarding without proper sync condition support?
- Should there be pre-sync validation to warn customers about expected duration with certain data volumes?