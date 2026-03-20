---
source_file: Efficy Enterprise issues.txt
domain: Integrations
topics: [Full Sync Performance, Large Dataset Handling, Sync Conditions, Consent Data Processing, CRM Integration Optimization, Memory Management]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [FSC Enterprise, Generic Connector, APSIS, Sync Conditions, Consent Mappings, Full Sync Consumer, Integration Report Store, installations table]
session_type: debugging-session
---

## Session Overview

This session addresses critical performance issues with full syncs in the Integrations domain, specifically triggered by a customer with 8 million contacts in their CRM system—4x larger than any previously tested scenario. The team identified two interconnected problems: gateway timeouts when requesting consent data from the source CRM system, and a memory-intensive optimization in APSIS that compares full consent exports against incoming data. The root cause is architectural: sync condition filtering currently happens on the APSIS side after downloading all data, rather than on the CRM side before transmission. The discussion covers the technical limitations, workarounds, and necessary changes to support enterprise-scale customers.

---

## Customer Issue Investigation and On-Premise Considerations

[Erik Andersson]: Three customers are affected by full sync failures or timeouts. One customer with an on-premise FSC Enterprise environment with extensive customizations has been particularly problematic.

### How to Identify Customer Integrations

To determine which integration a customer uses:

1. Obtain the customer name from the product help channel
2. Navigate to back office to get the account ID (the name alone is just a display name)
3. Query the database `installations` table, which is the authoritative source for integration mapping

```
SELECT * FROM installations WHERE account_id = [ID]
```

This shows which integration version is installed on which section and account. In the problematic case, the customer has **FSC Enterprise** installation.

### On-Premise Environment Constraints

[Erik Andersson]: While we know this customer is on-premise from external communication, the integration system has no way to detect this. From the integration perspective, it's simply an FSC Enterprise environment whether it's self-hosted or managed by FSC.

On-premise deployments typically surface specific architectural issues:

- **IP whitelisting requirements** — customers often request VPN tunnel support for private environments
- **Public access requirement** — APSIS requires internet-accessible endpoints to function; VPN tunnels are not supported for the Integrations domain (unlike in other products like Pro)
- **Customization complexity** — this customer has significant database customizations that may mask integration issues

> "If you want to use APSIS integrations with an on-premise CRM, we need public access to the environment. We don't support VPN tunnels because we have no one to maintain or support them."

[Erik Andersson]: The enterprise team created a test instance by copying the customer's database and testing the connector—it worked fine. This indicated the issue was likely with their specific setup rather than a fundamental integration problem.

---

## Full Sync Architecture and Current Limitations

### Data Scale Challenge

The problematic customer scenario:
- **8 million total contacts** in source CRM
- **1.2 million contacts** that should sync via sync conditions across multiple sections
- **100,000 contacts max** per section after filtering
- **Multiple consent entries per contact** (minimum 1, commonly 2, up to 20 in extreme cases observed)

This creates a total data payload of approximately **8 million contact entries + 8 million consent entries** that must be processed in a single full sync cycle.

### Current Sync Condition Implementation: APSIS-Side Filtering

**The design issue:** Sync conditions are configured in APSIS but filtering happens on the APSIS side after data retrieval, not on the CRM side.

When a sync condition is set (e.g., "only sync contacts where email = eric@appsis-fake.com"):

1. The full sync request is sent to the source CRM
2. The CRM returns **all 8 million profiles**
3. APSIS downloads all profiles and applies the filter locally
4. Only the matching 1.2 million contacts (or 100,000 per section) are actually synchronized

This means that even with aggressive filtering, APSIS must handle the full dataset of millions of records. The CRM's pagination mechanism returns data in chunks—in this case, **16,000 pages of profiles**—which still completed successfully despite the load.

### Data Download and Processing Flow

Profiles are downloaded with pagination to avoid memory overload:
- Each request returns a subset of profiles
- Messages are placed into **SQS** (Simple Queue Service)
- The full sync consumer processes messages asynchronously
- This streaming approach prevents holding all profile data in memory simultaneously

---

## Consent Processing Bottleneck

### The Gateway Timeout Problem

When the sync attempts to fetch consent data:

- **Attempted batch size:** 500 consent entries per request
- **Observed response time:** 22 seconds to download 500 entries
- **Result:** HTTP 504 (Gateway Timeout) errors from the CRM system

**Calculation of total time impact:**
```
(8,000,000 contacts ÷ 500 entries per batch) × 22 seconds = ~352,000 seconds
= ~98 hours of processing time
```

[Erik Andersson]: The observed sync logs showed the process was still running with timeout errors after 8+ hours. The HTTP 504 responses came back as HTML from a proxy server in front of the CRM's business logic, indicating the backend pagination implementation is extremely inefficient for large datasets.

### Why Consents Fail While Contacts Succeed

**Contact pagination works:** The same full sync successfully downloaded 16,000 pages of profiles, indicating the contact endpoint can handle large-scale data retrieval.

**Consent pagination fails:** The consent endpoint appears to have an inefficient pagination or caching implementation that degrades severely when requesting large numbers of entries. The exact cause is unknown—possibilities include:
- Cursor-based pagination that requires iterating through all records
- Sequential processing of consent lookups
- Proxy timeout thresholds being exceeded

**Visible impact:** Customers can see the synced contact data in APSIS (because profile downloads succeeded), but consent status information is missing because the consent data never arrived.

---

## Memory Optimization That Becomes a Liability

### The Existing Consent Comparison Optimization

During full sync, APSIS performs a consent comparison step to avoid redundant updates:

1. **Export existing consent status** from APSIS audience for all mapped subscriptions
2. **Load the entire export into memory** for the comparison operation
3. **Compare incoming consent data** from the CRM against the in-memory export
4. **Only send updates** if the consent status differs from what's already in APSIS

**Current state with 8M contacts:** The export is empty (no existing consents), so this optimization doesn't cause issues yet. However, the process is structured dangerously.

### Future Risk: Memory Exhaustion

If the CRM successfully delivers 8 million contacts with multiple consent entries each (16 million+ total consent records), and APSIS already has consent data in the audience platform, the comparison would:

1. Load 16+ million consent records into memory
2. Attempt to iterate through all incoming CRM consent records (also in memory conceptually)
3. Likely exhaust heap memory and crash the sync process

[Erik Andersson]: Unlike the contact data (which is streamed via SQS and processed asynchronously), the consent export comparison loads the entire dataset into memory, making it unsuitable for enterprise-scale customers.

### Recommended Fix

Removing this optimization and processing all incoming consent records without comparison would be safer:
- Incoming consent data is already streamed and not held entirely in memory
- The consumer processes records as they arrive
- This trades some redundancy (potential duplicate update messages) for stability with large datasets

---

## Solution Architecture: Server-Side Sync Condition Filtering

### What Works: Generic Connector Sync Condition Support

The APSIS codebase already contains infrastructure to support server-side filtering. The **Generic Connector** (used by FSC Enterprise 12.1 and Microsoft Dynamics via Site Shop) has a `syncConditionSupported: true` property.

### Implementation Reference: Site Shop Success

Microsoft Dynamics customers using Site Shop demonstrated the solution:

1. **Configuration phase:** Sync conditions are configured in APSIS as usual
2. **Registration:** Sync conditions are registered per section in the CRM system
3. **Enforcement:** When Site Shop receives a full sync request, it filters results based on registered conditions
4. **Result:** Instead of returning 1,000,000 contacts to filter locally, Site Shop returns only the matching contacts

> "For Site Shop, they only give us one contact instead of 1,000,000, making the full sync incredibly fast."

### CRM-Side Endpoint Requirement

To implement sync condition filtering in FSC Enterprise and E-Deal:

**Endpoint structure (based on existing patterns):**
```
/v1/sync-conditions/{section}
```

**Request parameters:**
- Section ID
- Entity type (contact, case, etc.)
- Field conditions in CRM format
- Boolean flag: should delete requests be sent for records that no longer match?

**Example condition registration:**
```json
{
  "section": "abc123",
  "entity": "contact",
  "conditions": [
    {
      "field": "firstname",  // CRM field name
      "operator": "equals",
      "value": "Elsa"
    },
    {
      "field": "sync_to_appsis",
      "operator": "equals", 
      "value": true
    }
  ],
  "send_delete_on_mismatch": true
}
```

**Full sync request behavior after implementation:**
- CRM applies conditions server-side before pagination begins
- Only matching records are returned in paginated chunks
- Consent records are also filtered to only include consents for matching contacts

### Impact on This Customer

With 1.2 million syncable contacts out of 8 million total:

**Current behavior (no filtering):**
- Download 8M contacts × [pages × request overhead]
- Download 16M consent entries (8M × 2 average consents)
- Process time: 1.5+ days
- CRM system stress: extreme (causing timeouts)

**With server-side filtering:**
- Download 1.2M contacts ÷ multiple sections = smaller batches
- Download only consents for matching contacts: ~1.2M-2.4M entries
- Process time: ~15-20 minutes
- CRM system stress: minimal

---

## Why Server-Side Filtering Is Architecturally Necessary

### Limitations of APSIS-Only Filtering

APSIS cannot replicate all CRM business logic:

**Unsupported filter types:**
- **Related entity conditions** — "only sync contacts from companies with revenue > 500,000 crowns/year"
- **Consent existence checks** — "only sync contacts that have at least one consent" (commonly requested)
- **Complex multi-table joins** — APSIS has no access to organizational hierarchy or derived fields

**Why this matters:** Customers increasingly want sophisticated segment conditions that require CRM schema knowledge and relational data that APSIS cannot access.

### Organizational Reality: Configuration vs. Enforcement

[Erik Andersson]: The current design is a compromise from failed attempts to move configuration entirely into the CRM system (Maxo project):

> "Everyone wanted the integration page in APSIS to disappear. They wanted to set up everything in Maxo. But no one wanted to spend time implementing the UI and configuration interface in the CRM system. So we settled for a middle ground: configuration stays in APSIS, but enforcement happens in the CRM."

**Ideal architecture:** Set up field mappings, subscription mappings, and sync conditions within the CRM system where you have direct schema access. But this requires significant UI development on the CRM side, which was never prioritized.

**Current compromise:** Keep configuration in APSIS for now, but send the conditions to the CRM system via a new endpoint where they're enforced during data retrieval.

---

## Performance Implications and ROI

### Processing Cost Analysis

[Erik Andersson]: This is fundamentally a money and stability issue:

**Current state:** 1.5+ day sync with 8M contacts means:
- Extended compute resources in use
- High data transfer costs between APSIS and CRM
- Proxy/gateway exhaustion on CRM side (causing timeouts)
- Customer frustration with slow sync times

**With server-side filtering:** 15-20 minute sync with 1.2M contacts means:
- Reduced compute time proportional to data reduction
- Lower bandwidth utilization
- Minimal stress on CRM infrastructure
- Happier customers with faster sync completion

> "This full sync which went on for 1.5 days could have been done in 15 minutes. With 100,000 contacts, it's nothing for us. But 8 million plus 8 million consents is a problem for us."

### CRM System Perspective

The CRM system is suffering equally:
- Proxy servers returning 504 errors under load
- Pagination cursors straining under millions of record iterations
- Inefficient consent lookup queries accumulating

Implementing server-side filtering benefits the CRM vendor as much as APSIS:
- Fewer incoming requests (one filtered request instead of paginating 8M records)
- Reduced query complexity per request
- Better overall system stability

---

## Immediate Mitigations vs. Long-Term Fix

### Immediate: Memory Optimization Removal (APSIS-only change)

**Action:** Remove the in-memory consent export comparison for full syncs

**Benefit:** Prevents future memory exhaustion crashes when enterprise customers have large existing consent datasets

**Limitation:** Does not solve the CRM timeout issue; only future-proofs against a different failure mode

**Code change:** Likely a configuration flag or conditional logic in the full sync consumer

### Interim: CRM Pagination/Caching Optimization (CRM-side optimization)

[Erik Andersson]: The CRM vendor might be able to make quick performance wins without full architectural changes:

- Optimize pagination cursor implementation
- Improve caching for sequential consent lookups
- Increase proxy timeout thresholds temporarily
- Pre-compute consent matrices during off-peak hours

**Benefit:** Could reduce sync time from 1.5 days to a few hours

**Limitation:** Still fundamentally downloads all 8M records; doesn't solve the scaling problem for future even-larger customers

### Long-Term: Server-Side Sync Condition Filtering (CRM-side implementation)

**Requirements:**
1. CRM vendor implements the `/v1/sync-conditions/{section}` endpoint
2. CRM vendor implements filtering logic in full sync endpoint
3. CRM vendor implements filtering logic in webhook/change notification handlers
4. APSIS enables the `syncConditionSupported: true` flag for the connector

**Effort distribution:**
- APSIS: Minimal (mostly configuration and flag enabling)
- CRM vendor: Significant (new endpoint, filtering logic, cache invalidation)

**Timeline:** Unknown; dependent on CRM vendor's roadmap and willingness to deprioritize other work

---

## Communication and Escalation Strategy

### The Process Issue: Multiple Support Channels

[Erik Andersson] identifies a structural problem impeding the solution:

> "I see support tickets in the product help channel and then expect things to move along, but that's not going to happen. No developer is expected to follow the product help channel. We have stories from product help, from internal chat, from emails—it will be way too much to keep track on."

**Current state:** Issues are reaching R&D through multiple channels:
- Product help channel
- Internal Slack messages
- Direct messages to engineers
- Email tickets

**Problem:** This prevents visibility, accountability, and prioritization. Engineers feel obligated to respond to everything, leading to:
- Context-switching
- Invisible work (not tracked in tickets)
- Inconsistent response times
- Circumvention of support workflows

### Recommended Workflow

1. **Product/support receives customer request** → tickets it in a centralized system
2. **Product team triages** → determines if integration issue
3. **Escalates to R&D** → via formal ticket in the help channel or equivalent system
4. **R&D responds only to formal tickets** → not internal messages or product help reactions

[Erik Andersson]: Enforce this by steering all inquiries back to the help channel:

> "I am steering everything right now to the help channel, regardless of where it comes from. And if this means I will upset people, sure, let them be upset. But we need one unified flow for it."

### Steps to Resolution for This Issue

**Step 1 (immediate):** Erik discusses with Product/PS team to contextualize the problem
- Explain the 8M contact + 8M consent scale scenario
- Show the load test data (previously tested with 2M, now dealing with 4x scale)
- Present the CRM timeout evidence
- Frame as both a performance and stability issue

**Step 2 (collaborative):** Convene a meeting with:
- APSIS R&D (Erik/Lukasz)
- Product team
- Customer Success/Professional Services (PS)
- Possibly CRM vendor representatives

**Step 3 (CRM vendor engagement):** Product/PS team leads the conversation with the CRM vendor about:
- The architectural limitation of APSIS-side filtering
- The need to implement server-side sync condition filtering
- Timeline and resource requirements
- Interim optimizations they could make

**Step 4 (expectation setting):** Regardless of CRM vendor's timeline:
- Set customer expectations about full sync duration with large datasets
- Document that APSIS has been tested with up to 2M contacts
- Explain the architectural reasons for CRM-side filtering requirement
- Provide interim workarounds if available (manual sync scheduling, incremental syncs, etc.)

---

## Technical Details: Sync Condition Endpoint Specification

### Endpoint Definition

**Path:**
```
POST /api/v1/sync-conditions/{section}
```

**Request body:**
```json
{
  "section": "section-id",
  "entity": "contact",  // or "case", etc.
  "conditions": [
    {
      "field": "firstname",
      "operator": "equals",
      "value": "Elsa"
    },
    {
      "field": "sync_to_appsis",
      "operator": "equals",
      "value": true
    }
  ],
  "send_delete_on_mismatch": true,
  "mapped_subscriptions": [
    "newsletter_technical",
    "updates_product"
  ]
}
```

**Response:**
```json
{
  "status": "success",
  "registered_conditions": 2,
  "affected_records_estimate": 1200000
}
```

### Where Conditions Are Used

1. **Full sync request:** CRM applies conditions to initial data retrieval, returns only matching records
2. **Webhook handlers:** For change notifications, CRM only sends webhook events for contacts matching conditions
3. **Consent-specific:** Only include consent records for contacts that match the sync conditions

### Filtering Logic Example (pseudocode)

```
FUNCTION apply_sync_conditions(contact, conditions):
  FOR EACH condition IN conditions:
    IF NOT evaluate_condition(contact[condition.field], condition.operator, condition.value):
      RETURN false
  RETURN true

FUNCTION full_sync_fetch():
  conditions = get_registered_sync_conditions(section_id)
  FOR EACH page IN paginated_contact_fetch():
    filtered_contacts = FILTER(page.contacts, apply_sync_conditions, conditions)
    FOR EACH contact IN filtered_contacts:
      FOR EACH consent IN contact.consents:
        yield(consent)  // Already filtered by contact match
```

---

## Historical Context: Why Configuration Remains in APSIS

### The Maxo Integration Effort

When Maxo (the CRM system's unified integration platform) was being developed, there was ambition to centralize all integration configuration:

- Field mappings to be set up in Maxo (not APSIS)
- Sync conditions to be configured in Maxo
- Subscription mappings to be configured in Maxo

**What went wrong:** Everyone wanted the APSIS integration page to disappear, but no one wanted to invest in building the UI and configuration interfaces within Maxo. This required:
- UI development in an unfamiliar codebase
- Changes to Maxo's data model
- New approval workflows
- Training and documentation

**Result:** The effort was shelved; APSIS remains the source of truth for configuration.

### The Compromise: Hybrid Configuration Model

Today's model reflects this:
- Configuration: APSIS (has working UI, accessible to all customers)
- Enforcement: CRM system (has the data and schema context)
- Transmission: API calls from APSIS to CRM to register conditions

This works but creates architectural misalignment. The ideal end state is still unclear:
- **Option A:** Move all configuration to CRM (Maxo vision)
- **Option B:** Move all configuration to a neutral platform (external data governance tool)
- **Option C:** Accept hybrid as permanent and optimize the API contract between systems

---

## Limitations and Caveats

### Testing Gaps

[Erik Andersson]: Current load testing has only covered up to 2 million contacts:

> "We have done load tests of the full sync before with 2 million contacts. That was the biggest customer at the time. Now we have someone who has onboarded with 8 million contacts—4 times more than we've tested."

**Implication:** Unknown performance degradation between 2M-8M scale. The exponential nature of consent processing (N contacts × M consents per contact) means the curve might be steeper than linear.

### On-Premise Environment Unknowns

With on-premise deployments:
- APSIS cannot determine the infrastructure capacity
- Custom CRM modifications may affect sync behavior
- Network latency between customer's data center and APSIS is unknown
- Firewall/proxy configurations vary widely

**Recommendation:** For on-premise customers of this scale, require:
- Dedicated connection with SLA guarantees
- CRM performance baseline testing before sync
- Incremental sync strategy (don't attempt 8M contacts in single full sync)

### CRM Vendor Implementation Risk

The fix depends entirely on the CRM vendor's engineering priorities:
- They may deprioritize this work
- They may implement it inefficiently
- They may require APSIS to pay for development
- They may demand we commit to testing/support

---

## Key Takeaways

1. **Scale problem is real:** 8M contacts with multiple consents per contact creates a 16M+ entry data payload that overwhelms current architecture. This is 4x larger than previous testing.

2. **Consent endpoint is the bottleneck:** Contact pagination succeeds; consent pagination times out. CRM's consent endpoint has inefficient pagination/caching that needs urgent optimization or architectural redesign.

3. **Architectural issue:** APSIS-side sync condition filtering forces us to download all data, then discard most of it. This is wasteful and doesn't scale. Server-side filtering on the CRM is necessary for enterprise customers.

4. **Infrastructure exists in codebase:** Generic Connector already has `syncConditionSupported` property. We need the CRM vendor to implement the corresponding endpoint and filtering logic.

5. **Two parallel efforts needed:**
   - **Immediate:** Remove memory-intensive consent comparison optimization to prevent future crashes
   - **Long-term:** Work with CRM vendor to implement server-side sync condition filtering

6. **Communication structure is broken:** Issues are reaching R&D through multiple channels. Enforcing a single formal ticket channel is necessary to maintain visibility and prevent context-switching.

7. **Expectation management is critical:** Before CRM vendor makes changes, we need to set customer expectations about limitations with large datasets and the timeline for fixes.

8. **This is both a technical and organizational problem:** The technical solution (server-side filtering) requires CRM vendor effort. Success depends on cross-organization prioritization and willingness to invest.

---

## Unresolved Questions and Action Items

### Questions Requiring Further Investigation

1. **CRM pagination details:** What exact mechanism is causing the 22-second response times for 500 consent entries? Is it cursor-based iteration, cache misses, or proxy slowness?

2. **Memory threshold:** At what number of consent entries would the in-memory export comparison fail? What's the actual heap limit in production?

3. **Alternative filtering:** Could APSIS push down filter conditions via GraphQL or other query language if the CRM supports it, rather than relying on a new endpoint?

4. **Interim workarounds:** Can we split the 8M customer into multiple smaller syncs, or use incremental syncs to avoid the full sync bottleneck?

### Action Items

- **Erik:** Schedule meeting with Product/PS team to discuss the 8M scale issue and CRM vendor communication strategy
- **Erik:** Highlight the consent endpoint issue in the enterprise channel to see if CRM can make quick pagination/caching optimizations
- **Team:** Confirm the consent export comparison is the correct memory optimization to remove and understand its impact if removed
- **Product/PS:** Engage CRM vendor to prioritize implementation of `/api/v1/sync-conditions/{section}` endpoint
- **Documentation:** Add platform limitation documentation stating: "Full sync has been tested up to 2M contacts. For larger datasets, server-side sync condition filtering is required (CRM-side implementation needed)."
- **Testing:** Plan load testing for 8M contacts once CRM vendor optimizations are in place to validate improvements

### External Dependencies

- CRM vendor must implement sync condition filtering endpoint
- CRM vendor must optimize pagination/caching for consent queries
- Product team must deprioritize other work to support CRM vendor discussions
- Customer/PS team must manage expectations until fixes are delivered