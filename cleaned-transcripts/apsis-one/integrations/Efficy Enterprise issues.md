---
source_file: Efficy Enterprise issues.txt
domain: Apsis One Integrations
topics: [Full Sync Performance Issues, Sync Conditions Architecture, Consent Data Handling, Large Dataset Optimization, On-Premise Deployment Constraints]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Apsis One, Efficy Enterprise, Generic Connector, Full Sync Process, Sync Conditions, Consent Mappings, Field Mappings, Integration Report Store]
session_type: debugging-session
subdomains: [Architecture, Generic Connector, Outbound Flow, Efficy Enterprise 12.0, Efficy Enterprise 12.1]
---

## Session Overview

This session focused on diagnosing severe full sync performance issues affecting customers with large contact datasets (7-8 million contacts) in Efficy Enterprise CRM environments. The discussion revealed that while contact downloads work acceptably, consent data retrieval triggers gateway timeouts from the CRM system, causing syncs to run 1.5+ days without completion. The root cause is that sync conditions are currently enforced on the Apsis One side after downloading all data from the CRM, rather than being applied at the source. The team identified that implementing server-side filtering in Efficy Enterprise (similar to existing Microsoft Dynamics functionality) is the primary solution needed, though this requires cross-organizational effort and CRM-side changes.

---

## Customer Issues and Scope

### Initial Problem Statement

[Erik Andersson]: Three customers have been reported as experiencing full sync failures or extreme slowness. One customer has an on-premise Efficy Enterprise environment with extensive customizations. The issues stem from discussions in the product help channel where multiple sync failures were documented.

### Identifying Customer Integration Type

[Lukasz Grabowski]: Asked how to determine which integration a customer uses.

[Erik Andersson]: The process requires:
1. Take the customer's display name from the issue
2. Go to back office to retrieve the account ID (the display name alone is insufficient)
3. Query the database `installations` table, which serves as the "complete repository of which integration is installed on which section and account"
4. Query: `SELECT * FROM installations WHERE ID = <account_id>` (must use ID, not display name)

For the affected customer: The query revealed they have an **FSC Enterprise 2** installation.

### On-Premise vs. Managed Deployment

[Lukasz Grabowski]: How do you know this is on-premise?

[Erik Andersson]: External communication from the customer/vendor indicated on-premise status. From the integration platform's perspective, there is no technical way to distinguish on-premise from managed deployments. Both appear as FSC Enterprise environments in the system. However, on-premise deployments create operational challenges not applicable to managed versions.

### On-Premise Deployment Constraints

[Erik Andersson]: On-premise Efficy Enterprise customers typically face these prerequisites:
- **Public internet access is mandatory** — Apsis One cannot support VPN tunnel configurations (unlike some other integrations)
- **IP whitelisting must be configured** — This is added at installation time
- **No VPN tunnel support** — This is a conscious design decision due to lack of maintenance resources

> "We need to be able to call it from the Internet for now period. Otherwise you can't use Apsis."

[Tomasz Kowalski]: Clarified whether the squid proxy rules or whitelist addresses need updates.

[Erik Andersson]: Whitelisting happens at installation time and is separate from VPN tunnel discussions. VPN tunnel setup is not an option.

[Michal Rosikiewicz]: Acknowledged that on-premise environments complicate matters.

---

## Full Sync Performance: Root Cause Analysis

### Baseline Capacity Testing

[Erik Andersson]: Previous load testing of full syncs was conducted with 2 million contacts at maximum. The largest customer previously onboarded had 1 million entries, and syncs, while slow, completed successfully.

> "We have now like someone has onboarded like this monster here where you have 8 million contacts in which is like 4 times more than we have actually tested the system with."

This is the first customer to exceed tested capacity by a factor of 4x.

### Sync Conditions: Current Architecture and Performance Impact

#### How Sync Conditions Work Today

[Erik Andersson]: Sync conditions allow filtering which contacts to synchronize. Example: "Only sync contacts where email = eric@appsis-fake.com"

**Critical Design Issue**: The filtering happens on the Apsis One side **after** data is downloaded from the CRM, not before:
- All 8 million contacts are downloaded from the CRM
- Filtering is applied in Apsis One
- Only the matching subset (e.g., 100,000 contacts) is actually synced
- The remaining 7.9 million contacts are discarded

This means for the problematic customer:
- 8 million contact records are downloaded and transferred across the network
- Only 1.2 million are actually synchronizable across multiple sections
- Each section receives a maximum of 100,000 contacts after filtering
- Every full sync re-downloads all 8 million records and re-filters

### Consent Data Handling and Cascading Failures

#### Data Volume Multiplier Effect

[Erik Andersson]: Consents compound the problem significantly:
- Typical customers: 1-2 consents per contact
- Some customers: Up to 20 consent lists mapped
- For this customer calculation: 8 million contacts × 1 consent (conservative) = 8 million consent entries

Total data to process per full sync:
- 8 million contact records
- 8 million consent entries
- **16 million total data elements**

#### Paginated Download and Timeout Cascades

The full sync downloads contacts paginated:
- This customer's sync fetched **16,000 pages of profiles** without critical errors
- The sync remained stable during the contact phase despite the volume

However, when requesting consents:
- **Each batch of 500 consent entries took 22 seconds to download**
- The CRM system (Efficy Enterprise) returned 504 Gateway Timeout errors
- Calculation: 8,000,000 ÷ 500 × 22 seconds = approximately **44,000 seconds (≈12+ hours)** just for consent retrieval

[Erik Andersson]: Provided observed timing:
- Sync started at 4 AM
- Continued with timeouts through 8 AM
- Still timing out at 11 AM (7 hours elapsed)
- Total sync duration: exceeded 8 hours before terminating

> "They wondered like why the full sync did take a long time."

#### Root Cause: CRM-Side Pagination Issue

[Erik Andersson]: The 504 responses are HTML error pages, indicating a proxy server in front of Efficy Enterprise's business logic. The CRM system has inefficient pagination implementation causing extraordinarily long response times for consent data retrieval.

> "I happen to know that FC has a proxy server in front of their business logic because you can see here that we are actually getting an HTML request back with the 504."

**Current State**: Contacts are reaching Apsis One successfully, which is why customers see contact records. Consents are not being added because timeout errors prevent consent retrieval.

---

## Comparison: Microsoft Dynamics Approach (Best Practice Model)

### Site Shop Implementation

[Erik Andersson]: For Microsoft Dynamics/Site Shop, a different approach was developed:

**Current Apsis One Approach (Filter After Download)**:
- Request: "Give us all contacts"
- CRM Response: 1,000,000 contacts
- Apsis One Action: Filter to matching conditions
- Result: Use 1,000 contacts, discard 999,000

**Site Shop Approach (Filter Before Download)**:
- Request: "Give us only contacts where email field matches eric@appsis-fake.com"
- CRM Response: 1 contact (pre-filtered)
- Apsis One Action: Receive only relevant data
- Result: Dramatic speed improvement in full sync

> "When you configure it here inside of Apsys, we make a request to site shop where we say you should only give us contacts where the e-mail field matches this value on site shop. They are then doing this filtering when we make a request."

This makes the full sync "incredibly, incredibly fast."

### Why This Model Must Be Extended to Efficy Enterprise

[Erik Andersson]: For customers with 8+ million contacts:
- Server-side filtering is not a nice-to-have optimization
- It is **mandatory for feasibility**
- Without it, Apsis One cannot practically support customers of this size

> "If we are to support customers of this size, the same thing has to be done inside of FSC Enterprise and E deal because it is not going to be feasible for us to sit and download 8 million entries, throw away 7,900,000 of them and only utilize these ones and then doing the same thing for consents."

---

## Technical Solution: Sync Condition Endpoint Implementation

### Generic Connector Support

[Erik Andersson]: The Generic Connector (which powers Efficy Enterprise 12.1 and Site Shop integrations) already has the capability:

Configuration property: `"connectorSupportsPreSyncCondition": true`

This boolean flag indicates the CRM system supports receiving sync conditions before the full sync request.

### Endpoint Requirements

[Lukasz Grabowski]: Asked what endpoint Efficy Enterprise needs to implement.

[Erik Andersson]: The endpoint follows this structure:

**V1 Sync Condition Endpoint** — sends configuration when sync conditions are set in Apsis One:
- Per-section registration
- Fields include:
  - **Entity type**: What data entity the condition applies to (e.g., Contact, Case)
  - **Delete request handling**: Boolean flag indicating whether to send delete requests when contacts no longer match conditions (configurable in Apsis One back office)
  - **Condition fields**: Array of field conditions
    - CRM field name (e.g., "FirstName")
    - Operator (e.g., equals)
    - Value (e.g., "Elsa")

**Example configuration**:
```
Entity: Contact
Field: FirstName
Condition: FirstName = "Elsa"
Sync to Apsis: true
```

### Implementation Requirements on CRM Side

[Erik Andersson]: Efficy Enterprise must implement:
1. A new API endpoint to receive sync condition definitions
2. Filtering logic to apply conditions to:
   - **Full sync requests**: Only return contacts matching conditions
   - **Webhook events**: Only send create/update/delete events for contacts matching conditions
3. Consent filtering: Only return consents that belong to contacts matching sync conditions

[Lukasz Grabowski]: Clarified that this requires exposing a new endpoint on the CRM side.

[Erik Andersson]: Yes, they need to expose an endpoint. When Apsis One changes sync conditions in the back office, a request is sent to this new CRM endpoint. The CRM then enforces filtering both in full sync responses and webhook events.

---

## Consent Export Optimization: Secondary Issue

### Current Optimization Strategy

During full sync, Apsis One performs this optimization:
1. Export all subscription mappings from Apsis One audience to memory
2. For each contact's consent from CRM: Compare with existing Apsis One state
3. If different: Create consent update message for consumer queue
4. If same: Skip (optimization — no update needed)

This is efficient for small datasets but has a hidden risk.

### Memory Risk at Scale

[Erik Andersson]: If the customer's entire audience were loaded (8 million contacts × multiple consents):
- Loading the full audience export into memory would likely cause out-of-memory crashes
- The sync would fail before reaching the CRM timeout issues

> "If we were to load that whole export into memory and do that comparison, I am fairly sure the sync will crash because of memory issues."

However, for the current customer, this is **not the breaking issue** because:
- Consent data never reaches Apsis One (CRM timeouts prevent it)
- The audience export remains empty
- Memory comparison never executes

### Recommended Fix: Remove the Optimization

[Lukasz Grabowski]: Asked if removing the optimization would solve the consent issue.

[Erik Andersson]: No — removing it would only prevent *future* crashes. The current customer is blocked by CRM timeouts, not memory issues.

**However**, it should be removed because:
1. Future-proofs the system for customers with millions of existing consents in Apsis One
2. CRM consent data is streamed incrementally (memory-efficient)
3. Audience export is loaded entirely into memory (memory-expensive)
4. Preventing memory crashes is valuable even if it doesn't solve the immediate timeout

> "If we should support customers of this size, it is probably an optimization we should remove and just handle anything that the CRM system gives us because that is streamed in in contrast to this export comparison."

---

## Complex Sync Conditions: Apsis One Limitations

### What Apsis One Cannot Filter Today

[Erik Andersson]: The system has fundamental limitations for complex filtering scenarios:

**Example 1 — Company-Level Filtering**:
- Business rule: "Only sync contacts from companies with revenue > 500,000 crowns annually"
- Why it fails: Apsis One doesn't have access to company/organizational relationship data
- Where it would work: Native in the CRM system (has direct access to company records)

**Example 2 — Consent Filtering**:
- Business rule: "Only sync contacts that have at least one active consent"
- Why it fails: Apsis One doesn't have a native way to express "exists" conditions
- Where it would work: Native in the CRM system (can query "WHERE consents.count > 0")

[Lukasz Grabowski]: These use cases are very common customer requests.

[Erik Andersson]: Absolutely. Customers frequently ask for these. Setting them up in the CRM would be straightforward because the CRM has direct access to related entity data that Apsis One doesn't expose.

### Historical Context: Maxo vs. Current Architecture

[Erik Andersson]: When Maxo (another integration platform) was being developed, there was a vision to:
- Remove the integration configuration page from Apsis One
- Set up field mappings in Maxo
- Set up sync conditions in Maxo
- Everything configured natively in the CRM system

This would have been ideal because:
- Complex business logic lives where the data exists
- Apsis One becomes a simpler consumer
- Operators have one place to configure integrations

However, it didn't happen because:
- No team prioritized implementing the UI/UX in the CRM system
- Everyone wanted to keep Apsis One as the configuration hub
- A compromise was reached: Configure in Apsis One, enforce in CRM

> "But no one wanted to actually spend the time of implementing the UI etcetera in the CRM system because like no one wanted to prioritize like anything. So we had to like settle for a middle ground where it was being enforced inside the CRM, but we still kept the configuration inside of inside of APSIS."

### Current State: Hybrid Approach

Today:
- Field mappings: Configured in Apsis One, enforced in CRM
- Subscription mappings: Configured in Apsis One, enforced in CRM
- Sync conditions: Configured in Apsis One, can be sent to CRM for enforcement (if supported)

The best solution would be native CRM configuration, but this requires prioritization and resources that weren't available.

---

## Network and Cost Impact

### Current Full Sync Resource Waste

[Erik Andersson]: For this customer, the full sync statistics reveal inefficiency:

- Contact count: 8 million
- Contacts actually synced: 1.2 million across sections
- Percentage utilized: ~15%
- Waste: ~85% of downloaded data discarded

For a 100,000-contact subset:
- Current approach: 8 million downloads × filtering = ~1.5+ days
- With server-side filtering: 100,000 downloads = ~15 minutes

**If server-side filtering were implemented**:
- Sync time: Reduction from 1.5+ days to ~15 minutes
- Customer satisfaction: Vastly improved (fast syncs)
- System cost: Reduced compute time and data transfer
- CRM load: Dramatically reduced stress on Efficy Enterprise

> "This full sync which went on for like 1 1/2 day before terminating itself like this could have been done in 15 minutes, 100,000 contacts. It's like nothing for us."

### Business Case for CRM-Side Changes

[Erik Andersson]: The implementation benefits everyone:
1. **Apsis One**: Reduced computing and bandwidth costs
2. **Customer**: Faster syncs, happier users
3. **Efficy Enterprise**: Less load on their system (currently being hammered by our requests)

> "It is not only a big issue for us, it is right now a bigger issue for the CRM system because they are giving us time out."

---

## Process and Escalation Path

### Current Workflow Issues

[Erik Andersson]: The integration team has been inundated with requests through multiple channels:
- Product help channel (public)
- Internal Slack messages (direct)
- Email
- Support tickets

This fragmentation creates problems:
- No unified tracking
- Developers cannot follow all channels
- Work gets duplicated or missed
- Support expectations become unrealistic

> "No developer is expected to sit and follow this product help like this is a dangerous thing."

[Lukasz Grabowski]: Acknowledged that some account managers and support staff are filing tickets in the product help channel without following proper routing.

### Recommended Process

[Erik Andersson]: Strict adherence to the support channel flow:
1. Issues must come through the official support channel (Submariner integration)
2. R&D responds only to support-routed issues
3. Internal routing to R&D is discouraged
4. This prevents scope creep and maintains predictability

> "I will not reply to internal messages because people are still trying to circumvent everything and coming directly to me."

The goal is to create precedent:
- If R&D doesn't respond to internal messages, people stop trying
- If support grows slow, the organization recognizes it needs more support staff
- This forces prioritization at the right level rather than R&D context-switching

[Lukasz Grabowski]: Agreed completely. The process must be enforced consistently.

### Escalation for This Customer Issue

**Step 1**: [Erik Andersson] will discuss with Opri (PS/Product team lead)
- Present the 8 million contact scenario
- Show the 1.5+ day sync failure with timeout cascades
- Explain that this is the first customer to exceed tested capacity (2 million limit)
- Set expectations: "We tested up to 2 million successfully, but this is 4x larger"

**Step 2**: Product team decides on escalation path
- May involve customer communication
- May require prioritization conversation with Efficy Enterprise
- May need cross-organizational agreement on timeline

**Step 3**: Negotiate CRM-side changes
- This is the "most difficult thing" according to Erik
- Requires Efficy Enterprise to:
  - Expose a new endpoint for sync conditions
  - Implement filtering logic
  - Prioritize work on their roadmap
- Apsis One has leverage: "Without this, we fail at scale"
- But customer may also have leverage: "If we can't support your CRM, we may need to discontinue"

[Lukasz Grabowski]: Emphasized that product team should be central to this discussion, not R&D driving it proactively.

[Erik Andersson]: Agreed. R&D can present the technical details, but product team owns the customer relationship and prioritization.

---

## Documented Limitations Gap

[Lukasz Grabowski]: Noted that there should be a documented limitations page (Docs or GitHub) that clearly states:
- Tested capacity: 2 million contacts
- Maximum contacts per full sync before performance degradation: X
- Consent data limitations: Y

This would prevent future miscalibrations and set customer expectations upfront.

[Erik Andersson]: Currently, no such documentation exists. This should be added.

---

## Efficy Enterprise Optimization Opportunities

### Contact Download Performance (Acceptable)

[Erik Andersson]: Contact downloads are working reasonably well:
- 8 million contacts downloaded in ~1.5 hours
- No crashes or failures
- Pagination is stable
- Network transfer works

This is acceptable even at scale because:
- It's a one-time operation
- Pagination handles large volumes
- Memory usage is controlled (streamed, not loaded entirely)

### Consent Download Performance (Critical Issue)

Consent retrieval is the failure point:
- CRM returns 504 Gateway Timeout errors
- Response time: 22 seconds per 500 consent batch
- This is likely due to:
  - Inefficient pagination in Efficy Enterprise
  - Proxy server caching issues
  - Database query performance on consent tables

[Erik Andersson]: Highlighted the issue in the enterprise channel to prompt CRM-side investigation:
- Can Efficy Enterprise optimize pagination?
- Can they improve consent query performance?
- Can they improve caching?

Even partial improvements would help:
- Not a perfect solution, but prevents 1.5+ day syncs
- Gives customers something better than complete failure
- Might prevent need for emergency support

> "It might be that they can do some simple optimizations. So we can still maybe have not a perfect really well working solution, but maybe at least we can prevent the sinks from taking two days if they just optimize their pagination or caching or whatever."

---

## Key Takeaways

1. **Capacity Threshold Exceeded**: This customer (8 million contacts) is the first to exceed the tested limit (2 million contacts) by 4x. Current architecture was not designed for this scale.

2. **Two-Tier Problem**:
   - Immediate blocker: CRM system timeouts on consent retrieval (not Apsis One's fault)
   - Underlying issue: Apsis One downloads all 8 million contacts then filters, rather than filtering at source

3. **Sync Condition Server-Side Filtering is Mandatory**: Without it, customers of this size cannot be effectively supported. This feature already exists for Microsoft Dynamics/Site Shop and the Generic Connector architecture supports it.

4. **Implementation Path**:
   - Apsis One side: Set `"connectorSupportsPreSyncCondition": true` in Generic Connector configuration for Efficy Enterprise (code is ready)
   - CRM side (Efficy Enterprise): Implement endpoint to receive sync conditions, apply filtering to full sync and webhook events
   - This is the heavy lift and requires CRM vendor prioritization

5. **Secondary Optimization**: Remove the in-memory audience export comparison optimization to prevent future memory crashes at scale, though this won't solve the current customer's timeout issue.

6. **Cost/Performance Equation**: With server-side filtering, the affected customer's 1.5+ day sync would become a 15-minute operation, reducing costs for Apsis One and load on Efficy Enterprise.

7. **Process Enforcement**: Support requests must flow through the proper channel (Submariner/support tickets). R&D will not respond to internal Slack messages or product help channel posts. This forces organization-level recognition of support capacity gaps.

8. **Documentation Needed**: Platform limitations (tested capacity: 2 million contacts) should be documented to set customer expectations and prevent future capacity miscalibrations.

---

## Unresolved Questions and Action Items

### Open Questions
- What is Efficy Enterprise's pagination implementation? Can simple optimization resolve timeout issues?
- How receptive will Efficy Enterprise be to prioritizing the sync condition endpoint implementation?
- Should the tested capacity limit (2 million) be formally documented before more large customers are onboarded?

### Action Items
- [Erik Andersson]: Discuss with Opri (PS/Product team) about this customer and the 8 million contact scenario
- [Erik Andersson]: Highlight the consent download timeout issue in the Efficy Enterprise channel to prompt optimization investigation
- [Team]: Document platform capacity limitations (2 million tested limit, consent handling requirements) in Docs/GitHub
- [Product team]: Coordinate with Efficy Enterprise about implementing sync condition endpoint and filtering logic
- [R&D - future]: If Efficy Enterprise implements the endpoint, enable `"connectorSupportsPreSyncCondition": true` in Generic Connector configuration

### Blocked/Waiting
- This customer's issue remains with product help channel / support team pending Opri's assessment
- No R&D proactive action until support formally escalates through proper channels