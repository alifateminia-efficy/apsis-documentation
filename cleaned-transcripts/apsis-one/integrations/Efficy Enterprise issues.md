---
source_file: Efficy Enterprise issues.txt
domain: Apsis One Integrations
topics: [Full Sync Performance Issues, Sync Conditions, Consent Handling, Large Dataset Processing, CRM System Integration Optimization, Generic Connector Capabilities]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Apsis One, FSC Enterprise 12.0, FSC Enterprise 12.1, Generic Connector, Sync Conditions, Consent Mappings, Full Sync, Integration Database, Back Office]
session_type: knowledge-transfer
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This session addresses critical performance issues encountered during full syncs with large enterprise customers using Efficy Enterprise (FSC Enterprise) connectors. The discussion centers on a specific customer with 8 million contacts and multiple consent entries that experiences timeout failures during sync operations. The team explores the architectural limitations of how sync conditions are currently implemented, proposes moving filtering logic to the CRM side, and discusses the trade-offs between client-side and server-side filtering. Key themes include managing customer expectations around scalability limits, understanding the generic connector's capabilities, and establishing proper support workflows.

---

## Customer Investigation and On-Premise Constraints

### Identifying Customer Integration Type

[Erik Andersson]: To find which integration a customer uses, you need to:
1. Take the customer's display name from the help channel
2. Go to the back office to retrieve the account ID (the display name is not sufficient)
3. Query the database, specifically the `installations` table
4. Run: `SELECT * FROM installations WHERE ID = <account_id>`

The `installations` table is the authoritative source for which integration is installed on which account because it's also used by the integration report store.

[Lukasz Grabowski]: Asks: How can we tell if a customer is on-premise from the integration side?

[Erik Andersson]: We cannot determine on-premise deployment from the integration logs. We're told by the account manager after investigation. From our perspective, it makes no difference whether FSC Enterprise is managed by FSC or self-hosted — we treat it identically. We have no metadata to distinguish deployment model.

### On-Premise Prerequisites and Limitations

[Erik Andersson]: On-premise deployments have recurring issues:
- **IP whitelisting requirements**: Customers often state they have on-premise but don't allow public access
- **Our core requirement**: We need public internet access to the environment for Apsis to function
- **We do not support VPN tunnels**: Unlike the Pro team (which has resources to maintain tunnels), the integration team cannot maintain VPN tunnel infrastructure
- **Hard constraint**: If a customer wants to use Apsis with on-premise FSC, the environment must be accessible from the internet. Period.

[Tomasz Kowalski]: Asks about updating proxy rules (squid proxy) for whitelisting.

[Erik Andersson]: IP whitelisting is added at installation time, but that's different from the VPN tunnel issue. The squid proxy is separate from the core constraint.

[Michal Rosikiewicz]: Confirms that on-premise deployments complicate the situation.

### Case Study: Customer with Customizations

[Erik Andersson]: One affected customer has extensive customizations and an on-premise environment where nothing works. However, this is not an Apsis issue. The Enterprise team took a copy of the customer's database, added it to a managed instance they maintained, ran the connector, and everything worked. This proves the issue is their custom setup, not our integration. We're waiting for the Operations team to confirm this before proceeding.

---

## Full Sync Performance with Large Datasets

### The 8 Million Contact Problem

[Erik Andersson]: A new customer was onboarded with **8 million contacts** in FSC Enterprise. This is 4x the largest dataset we've tested in load testing (2 million contacts). Previously, the largest real customer had 1 million entries.

Breaking down the data scope:
- **Total contacts in CRM**: 8 million
- **Contacts that should sync** (via sync conditions): 1.2 million across multiple sections
- **Contacts per section**: Approximately 100,000 each
- **Consent entries**: Estimated 8 million (roughly 1:1 with contacts)

Despite sync conditions being configured to filter down to 1.2 million profiles, **we still download all 8 million contact records** during full sync because filtering happens on the Apsis side after download.

### Download Performance Metrics

The contact download phase worked:
- Fetched **16,000 pages** of profiles
- No crashes; took a long time but completed
- Pagination handled gracefully

The consent download phase failed catastrophically:
- Requested 500 consent entries → **504 Gateway Timeout** from the CRM
- Timing measurements: 22 seconds to download just 500 entries
- Math: 8 million ÷ 500 × 22 seconds = weeks of processing time
- Full sync started at 4:00 PM, by 11:00 PM (7 hours later) still getting timeouts

[Erik Andersson]: We're seeing HTML 504 responses with a proxy server error, indicating the CRM system has a proxy in front of its business logic that's timing out. The pagination implementation on the CRM side appears to have performance issues.

---

## Architecture: Client-Side vs Server-Side Filtering

### Current Design: Apsis-Side Sync Conditions

Today, sync conditions work as follows:
- Configuration happens in **Apsis back office**
- Example: "Only sync contacts where email = eric@apsis-fake.com"
- **Enforcement location**: Still on Apsis side (we download everything, then filter)

**Design implication**: Even with sync conditions configured, Apsis downloads 100% of the data from the CRM, then discards 98.8% of it locally.

### The Site Shop / Dynamics Pattern: CRM-Side Filtering

[Erik Andersson]: For Microsoft Dynamics Site Shop, a new feature was developed specifically to avoid this problem:
- Configuration: Still set in Apsis back office (field mappings, sync conditions)
- Execution: When sync conditions are changed, we send a request to Site Shop with the filtering parameters
- **CRM-side logic**: Site Shop filters its own data before responding
- Result: Instead of returning 1 million contacts, Site Shop returns only the 1 contact matching the criteria
- Full sync execution time: Dramatically faster (minutes instead of hours/days)

**The generic connector supports this pattern** via the property `connector_supports_sync_condition: true`

[Erik Andersson]: For FSC Enterprise 12.1 and E-Deal to support customers of this scale, they need the same filtering capability. Without it, full syncs are simply not feasible for large datasets.

### Sync Condition Endpoint Specification

[Lukasz Grabowski]: Questions what endpoint is needed for the CRM system to receive filtering parameters.

[Erik Andersson]: The endpoint follows this pattern (using the example from Dynamics Site Shop):

```
POST /api/v1/sync_condition
Body: {
  "section_id": <section>,
  "entity": "contact",
  "delete_if_no_longer_matching": true/false,
  "conditions": [
    {
      "field": "first_name",  // CRM field name
      "value": "Elsa"
    },
    {
      "field": "sync_to_apsis",
      "value": true
    }
  ]
}
```

**For consents specifically**: Only return consent entries belonging to contacts that match the sync conditions.

The CRM must then:
1. Store these conditions
2. Apply them when generating full sync results
3. Apply them when sending webhook notifications (only notify for contacts matching conditions)

### Architectural Considerations for CRM Implementation

[Erik Andersson]: The CRM implementers face real complexity challenges:
- **Caching**: Need to pre-compute filtered datasets or cache them efficiently
- **Real-time vs batch**: Can't update sync conditions in real-time for all related contacts
- **Data relationships**: Need pagination and cache management for large result sets
- **SQL integration**: Must incorporate filtering into SQL queries or use prepared datasets

> "I fully acknowledge that handling this in an efficient way is not trivial because you have a lot of pagination updates or cache updates or however you handle this."

However, **the work is on the CRM side, not Apsis side**. We're getting off easy because they must implement the heavy lifting.

---

## Consent Handling and Memory Optimization

### Current Consent Optimization: The Audience Export Cache

During full sync, there's an optimization for consent comparison:
1. Export all existing consent status from Apsis for subscriptions with mappings
2. **Keep the entire export in memory**
3. Download consents from CRM (streamed, not held in memory)
4. For each CRM consent entry, compare to the cached export
5. Only send consent update messages if the status differs

This saves API calls and SQS messages by eliminating redundant "no change" updates.

### Memory Risk with 8 Million Contacts

[Erik Andersson]: If a customer has 8 million contacts, each with multiple consent entries, keeping the full Apsis audience export in memory will likely cause an **out-of-memory crash**.

**Proposed fix**: Remove this optimization and process all CRM consent data immediately:
- Stream consent data from CRM (already streamed)
- Place into SQS immediately without in-memory comparison
- Let the consumer handle all updates (no filtering)

This is safer because:
- CRM data is already streamed and not held in memory
- We remove the single point of memory failure
- Apsis audience export can be 7-8 million entries × multiple consents
- For this customer, we never reach this point anyway (CRM times out first), but it's a ticking time bomb for when CRM pagination is fixed

[Lukasz Grabowski]: Asks if removing the filter is a temporary solution.

[Erik Andersson]: No — it's a preventive measure. This customer isn't hitting the memory issue now (they're stuck on CRM timeouts), but when the CRM fixes their pagination, removing this optimization prevents future crashes at scale.

---

## Product Development History: Why Filtering Is on Apsis Side

### The Maxo Vision vs Current Reality

[Erik Andersson]: When Maxo was being developed, the vision was:
- **Desired state**: Configuration and enforcement in the CRM system only
- **Reality**: No one wanted to invest engineering time in CRM-side UI and configuration
- **Compromise**: Configuration stays in Apsis, enforcement moved to CRM (for connectors that support it)

The original desire was to eliminate the integration configuration page in Apsis entirely, moving field mappings, sync conditions, and consent mappings into Maxo (the CRM side).

**Why this didn't happen**: Prioritization. Everyone wanted to protect their own development time.

### Advanced Use Cases Only Possible in CRM

[Erik Andersson]: The current compromise has a limitation. Sync conditions that reference **related entity data** are impossible in Apsis but trivial in CRM:

Examples:
- "Only sync contacts from companies with annual revenue > 500,000 crowns"
  - We don't have organizational/related entity data in Apsis
  - CRM has direct access to this
  
- "Only sync contacts that have at least one consent"
  - Very common customer request
  - Can't be done in Apsis (we don't query consent data to filter contacts)
  - Easy in CRM (direct database query)

> "There is a point in actually setting up sync conditions inside the CRM system instead, because there you have access to a plethora of data that we don't have access to inside of APSIS."

**The ideal future**: Sync conditions configured in CRM, not Apsis. But that requires building UI and investment on the CRM side.

---

## Performance Math and Business Impact

### The Processing Cost

[Erik Andersson]: Current state for this 8-million-contact customer:
- **Expected full sync time (with CRM-side filtering)**: ~15 minutes for 100,000 contacts
- **Actual full sync time (without filtering)**: 30+ hours, eventually timing out
- **Cost**: Unnecessary compute, data transfer, CRM strain

If 3 customers now have 7+ million contacts each:
- Trend is worsening
- Current architecture doesn't scale
- Every full sync becomes a major incident

### Why the CRM Feels the Pressure

[Erik Andersson]: This isn't just an Apsis problem:
- We're requesting 8 million contacts, then discarding 98.8%
- We're requesting 8 million consent records, each in batches of 500, causing 16,000 API calls
- The CRM proxy server is timing out and returning 504 errors
- **The CRM is suffering more than we are**

Framing for executive discussions:
- **Money angle**: Less compute time = lower costs for Apsis
- **Money angle for CRM**: Less load on their systems = lower infrastructure costs
- **Stability angle**: CRM experiencing failures; this fix will stabilize their system
- **Customer satisfaction**: 20-minute full syncs vs. 1.5-day syncs = happier customers

---

## Generic Connector: Sync Condition Support

### Connector Capability Declaration

The generic connector supports the `connector_supports_sync_condition` property:

```
connector_supports_sync_condition: true
```

This flag indicates the CRM can handle the endpoint for receiving sync condition updates.

### FSC Enterprise Status

[Erik Andersson]: FSC Enterprise 12.1 uses the generic connector and would support sync condition filtering IF the CRM exposes the necessary endpoint and filtering logic.

[Lukasz Grabowski]: Questions whether toggling this property on the generic connector would work and what happens to the UI.

[Erik Andersson]: If you toggle `connector_supports_sync_condition: true`:
- The configuration UI in Apsis back office remains visible (sync conditions are still set there)
- The configuration flows through to the CRM endpoint
- The CRM is responsible for enforcing it
- No changes needed on the Apsis front-end for this connector

The property name in the generic connector settings controls whether the sync condition delivery mechanism is enabled.

---

## Support Process and Escalation Path

### Current Issue Routing

[Lukasz Grabowski]: References a ticket in Submariner (the support system).

[Erik Andersson]: Strong position on support workflow:
- We should **NOT** be proactive on issues that come through informal channels
- If issues come from product-help channel or internal chat, we can't track them properly
- **Single source of truth**: Submariner (support ticketing system)
- If support is slow, that's a support team capacity issue, not an R&D issue

> "No developer is expected to sit and follow this product help channel like this is a dangerous thing."

The risk of being "nice" and reactive on informal channels:
- Sets precedent that R&D will respond to all informal reports
- People exploit alternative routes to bypass support process
- Creates chaos and blocks actual R&D work

### Proper Escalation Flow

1. **Submit ticket to Submariner** (support system) if you're a customer or PS team member
2. **PS team** investigates and escalates to R&D with a Submariner ticket
3. **R&D prioritizes based on ticket system**, not chat urgency
4. **Product/Opri** gets involved for business-level decisions (like "this customer is important, we must fix this")
5. **Cross-functional alignment**: Product + R&D + CRM team on solution and timeline

[Erik Andersson]: For this specific issue, the steps are:
1. Erik will talk with Opri (Product) to frame the business and technical case
2. Establish expectations: We've tested up to 2 million contacts successfully; this 8-million case requires CRM-side changes
3. Product decides next steps (meeting with CRM team, formal request, etc.)
4. Not a quick R&D fix — this is a cross-organizational effort requiring CRM investment

[Lukasz Grabowski]: Highlights the complexity:
- On Apsis side: Small effort to enable sync condition filtering
- On CRM side: Significant effort to implement filtering and optimize pagination
- On communication side: **Hardest part** — convincing the CRM team to prioritize this
- Risk: CRM may refuse, customer may leave

[Erik Andersson]: This is always the bottleneck with cross-organization efforts — everyone protects their own roadmap.

**Leverage points in the conversation**:
- Cost savings (less compute)
- Stability (CRM is failing right now)
- Performance (customer happiness)
- Scale (we're seeing multiple large customers appear)

---

## Generic Connector's Current Capabilities and Limitations

### What We Have

The generic connector is built to support:
- Field mapping configuration (via Apsis)
- Sync condition configuration (via Apsis)
- The ability to send sync conditions to CRM endpoints
- Consent mapping configuration

### What Needs CRM Implementation

For the generic connector to actually use sync condition filtering, the CRM must:
1. Expose an endpoint to receive sync condition updates
2. Implement filtering logic in responses to full sync and webhook requests
3. Optimize pagination and caching for large datasets

---

## Proposed Actions and Next Steps

### Immediate Actions

[Erik Andersson]:
1. **Talk with Opri (Product Manager)** — frame the problem as:
   - Customer has 8 million contacts; we've only tested 2 million
   - CRM is experiencing 504 timeouts on consent pagination
   - Requires CRM-side filtering to solve
   - This will become a bigger issue as more large customers onboard

2. **Set customer expectations** — in communication with the customer, be clear:
   - We've verified the issue is CRM pagination, not Apsis
   - We're working with Product to escalate to CRM team
   - Timeline depends on CRM team's prioritization

3. **Highlight issue in Enterprise channel** — brief the Enterprise team on:
   - Contact downloads work fine (1.5+ hours for 8M is acceptable)
   - Consent downloads fail (timeouts)
   - CRM may be able to make quick pagination/caching optimizations even before filtering is implemented

### Longer-Term Resolution

[Lukasz Grabowski]: Product should organize a meeting with:
- R&D (to explain sync condition support)
- Product (to make business case)
- CRM team (to implement)
- Agreement on solution and timeline

The difficult part is convincing the CRM team that this investment is worth their effort.

### What We're NOT Doing

[Erik Andersson]: We are **not** proactively investigating issues submitted informally or in internal chat channels. This requires the support team to funnel these into Submariner first.

---

## Technical Debugging Insights

### How We Identified the Problem

[Erik Andersson]: The debugging chain:
1. Support tickets mentioned "full syncs taking a long time" or "consents not added"
2. Looked at sync logs: contacts downloading successfully, consents failing
3. Reproduced: Requested 500 consent entries, got 504 timeout
4. Personal test: 500 entries took 22 seconds
5. Extrapolation: 8 million ÷ 500 × 22 = 1,465 hours (60+ days)
6. Log inspection: Sync started 4 PM, still timing out at 11 PM
7. Error analysis: 504 Gateway Timeout is coming from a proxy, not directly from CRM logic

### Why Contact Download Succeeds, Consent Download Fails

[Erik Andersson]: Hypothesis — the CRM's pagination implementation differs between contact and consent endpoints:
- **Contact endpoint**: Optimized pagination, returns pages quickly
- **Consent endpoint**: Un-optimized pagination or caching issue, returns pages slowly

> "I happen to know that FC has a proxy server in front of their business logic because you can see here that we are actually getting an HTML request back with the 504."

The proxy is returning HTML error responses, indicating a full timeout, not a partial slowdown.

---

## Key Takeaways

1. **Scale is a new problem**: We've now onboarded customers 4x larger than our tested baseline. Current architecture doesn't scale to 8M+ contacts.

2. **Sync conditions must move to CRM**: Client-side filtering after full download is not viable for large datasets. For Site Shop (Dynamics), CRM-side filtering works beautifully. FSC Enterprise and E-deal need the same.

3. **The work is on their side**: Apsis already supports sending sync conditions to CRM. The CRM system must implement the endpoint and filtering logic. This is not a quick R&D fix.

4. **Consent handling is a memory risk**: The in-memory audience export optimization is safe today but will cause crashes once CRM pagination is fixed. We should remove it proactively.

5. **Support workflow is non-negotiable**: Issues must come through Submariner. Ad hoc reports in chat channels will be ignored to protect R&D capacity.

6. **Multiple angle attack needed**: Get buy-in from Product (Opri), frame this as cost/stability/performance issue, involve CRM team in a formal planning session.

7. **Generic connector is ready**: The infrastructure is in place. We just need CRM implementation on the other side.

---

## Unresolved Questions and Action Items

### Action Items

- [ ] **Erik Andersson**: Schedule meeting with Opri (Product) to discuss business case and escalation path for CRM team
- [ ] **Erik Andersson**: Post summary in Enterprise team channel highlighting consent download performance issue
- [ ] **Team**: Remove in-memory audience export optimization from consent handling (preventive)
- [ ] **Product/Operations**: Formal outreach to CRM team with sync condition filtering requirements

### Questions Requiring Follow-Up

- What is the specific endpoint path for FSC Enterprise 12.1 to receive sync conditions? (Erik mentioned needing Prem to confirm exact path)
- Can the CRM team make quick pagination or caching optimizations on the consent endpoint before full filtering is implemented?
- How many other customers might be at risk of hitting this 7M+ contact threshold?
- What is the CRM team's capacity and timeline for implementing sync condition filtering?