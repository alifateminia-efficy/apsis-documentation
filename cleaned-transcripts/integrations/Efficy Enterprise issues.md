---
source_file: Efficy Enterprise issues.txt
domain: Integrations
topics: [Full Sync Performance, Sync Conditions, Consent Handling, Large Dataset Management, CRM Integration Architecture, FSC Enterprise Scaling, Generic Connector Capabilities]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [FSC Enterprise, Efficy CRM, APSIS (audience management platform), Generic Connector, Sync Conditions, Consent Mappings, Full Sync Process, Database Installations Table]
session_type: debugging-session
---

## Session Overview

This session covers critical performance issues encountered during full sync operations with a large-scale FSC Enterprise customer (8 million contacts with multiple consent entries). The team discusses the architectural limitations of the current integration design, particularly around how sync conditions are currently applied post-download rather than pre-filtering at the source. Key topics include identifying the root cause of gateway timeouts during consent retrieval, understanding the scaling limitations of the current system, and proposing a solution that requires moving filtering logic from APSIS to the CRM system itself.

---

## Customer Investigation and Account Identification

### Locating Customer Integration Details

[Erik Andersson]: When investigating customer issues reported in the product help channel, the first step is to identify which integration a customer uses. This requires navigating to the back office using the account name, but the key is that **we need the account ID, not the display name**.

The most reliable source for checking which integration is installed is the **installations table in the database**:

```sql
SELECT * FROM installations 
WHERE ID = [account_id]
```

This table maintains a complete repository of which integrations are installed on which sections and accounts. In this case, the customer was found to have an **FSC Enterprise 2 installation**.

### On-Premise vs. Cloud Deployments

[Erik Andersson]: From the integrations team's perspective, there is **no technical way to determine if a customer's deployment is on-premise or cloud-hosted**. This distinction only comes from customer communication or documentation. Whether FSC manages the environment or the customer does is irrelevant to the integration layer.

**Important caveat**: On-premise installations typically surface specific recurring issues:
- **IP whitelisting and access requirements**: Many customers say "we have this on-premise" but then refuse to allow public internet access
- **VPN tunnel requirements**: APSIS does not support VPN tunnels for on-premise environments (unlike the Pro product, which has dedicated support). The current requirement is: **environments must have public internet access for APSIS integrations to function**

[Tomasz Kowalski asked about squid proxy rules needing updates]: [Erik Andersson]: IP whitelisting is handled at installation time and is separate from VPN tunnel requirements. The squid proxy is not involved in VPN tunnel discussions.

[Michal Rosikiewicz]: On-premise deployments do significantly complicate support and troubleshooting.

---

## Root Cause Analysis: Full Sync Performance Degradation with Large Datasets

### Load Testing History and New Scale

[Erik Andersson]: Previous load tests for full sync operations were conducted with **2 million contacts**, which at the time represented the largest customer dataset. Full syncs would take time but ultimately succeeded. The test benchmark was established because no customer approached that volume.

**Current situation**: A new customer has been onboarded with **8 million contacts** — **4 times larger than any previously tested scenario**. This represents a fundamental scaling challenge for the integration platform.

### The Sync Condition Architecture Problem

Sync conditions allow filtering which contacts should be synchronized. For example, a condition might be: "only sync contacts where email matches eric@appsis-fake.com."

**Critical architectural limitation**: Even when sync conditions are applied, the filtering **still happens on APSIS's side**, not the source system's side.

What this means:
- The full sync must download **all 8 million contacts** from the CRM system
- These are filtered locally in APSIS
- Only the filtered subset (e.g., 100,000 contacts) is actually synchronized
- **All 8 million are still transferred and processed**

[Erik Andersson]: Here's a concrete example: this customer has 8 million contacts, of which 1.2 million are synchronizable across multiple sections, with max 100,000 contacts per section. However, each full sync still handles downloading all 8 million profiles even though only 1.2 million will be used.

### Consent Data Multiplication Problem

Each contact typically has at least one consent entry (sometimes 2, with extreme cases of 20+ consent lists mapped). For this customer:
- 8 million contact entries to download
- 8 million consent entries to download (approximately 1:1 ratio minimum)
- **Total data volume becomes humongous**

[Erik Andersson]: The impact is not limited to volume but to the actual performance characteristics observed.

---

## Consent Retrieval Failure: Gateway Timeouts and Performance Degradation

### Observed Behavior

Contact pagination worked without crashes. The customer's full sync fetched **16,000 pages of profiles** without critical failures — it took a long time, but succeeded.

**Consent retrieval hit a hard wall**:
- After requesting just **500 consent entries**, the CRM system returned a **504 Gateway Timeout** error
- When manually testing the same request, a single batch of 500 consent entries took **22 seconds** to download

[Erik Andersson]: Do the math: 8 million contacts ÷ 500 (per request) × 22 seconds per request = an unacceptable timeframe. The sync started around 4 AM and by 8 AM, 11 AM, and beyond, it was still timing out.

### Root Cause: CRM System Bottleneck

The CRM system (FSC Enterprise) has a **proxy server in front of its business logic**. We know this because:
- Contact downloads succeeded, returning paginated data correctly
- Consent downloads failed with HTML 504 responses (not API responses)
- The proxy/gateway is timing out, likely due to inefficient pagination implementation on the CRM side

[Erik Andersson]: > "They are taking an incredibly long time to give a reply, and I happen to know that FSC has a proxy server in front of their business logic because you can see here that we are actually getting an HTML request back with the 504."

**Critical understanding**: APSIS cannot solve this problem unilaterally. The CRM system must optimize its pagination or caching mechanisms to handle these requests faster.

---

## Current Consent Filtering Optimization and Future Risk

### Existing Optimization: Audience Export Comparison

The full sync includes an optimization step:
1. **Export from APSIS** all subscriptions that have consent mappings
2. **Keep this export in memory** 
3. **Compare** incoming consent from the CRM: does it differ from what's already in APSIS?
4. **If different**: create a consent update message
5. **If same**: skip (no update needed)

This optimization works today because of the dataset sizes historically handled.

### The Memory Scaling Problem

[Erik Andersson]: If a customer has 8 million contacts in APSIS with multiple consent entries each, and we load the entire export into memory to perform this comparison, **the sync will very likely crash due to memory issues**.

This is because:
- Contact/consent data **streamed from the CRM is not kept in memory** — it goes directly to SQS for the consumer to handle
- The **audience export is loaded entirely into memory** — creating an asymmetry
- 8 million entries + consent data + comparison logic = potential OutOfMemory exception

[Erik Andersson]: For this particular customer, this isn't the breaking issue right now because "the existing consent matrix is empty because we have no consents in APSIS for this." The contact download reached APSIS, but consent data never arrived due to the gateway timeouts.

**Going forward**, if consent data does arrive and the customer runs another full sync with existing data in APSIS, this optimization could fail.

### Proposed Solution: Remove the Optimization

[Erik Andersson]: We should **remove this optimization and process all incoming CRM data as it streams in**, rather than holding it in memory. This trades a minor performance optimization (skipping redundant updates) for stability at scale.

[Lukasz Grabowski asked if removing the filter is a temporary solution]: [Erik Andersson]: Removing the filter won't solve the current problem (we're not even receiving the consent data). But it will **prevent future crashes when over 8 million consent entries are in APSIS**. The current breaking issue is the CRM's timeout; removing this optimization ensures we don't hit memory limits once that's fixed.

---

## Solution Architecture: Pushing Sync Condition Filtering to the CRM

### Current Design vs. Proposed Design

**Today** (current state):
- Sync conditions are configured in APSIS back office
- APSIS downloads all data
- APSIS filters locally
- Only filtered data is synced

**Proposed solution**:
- Sync conditions are configured in APSIS back office (UI stays the same)
- APSIS **sends the sync conditions to the CRM** via an API call
- The CRM **applies filtering before returning data**
- APSIS receives only the data it needs

### Precedent: Dynamics + Site Shop Implementation

[Erik Andersson]: For **Microsoft Dynamics and Site Shop**, a new feature was developed specifically because Site Shop wanted to sync only contacts matching sync conditions.

The solution:
- Site Shop needed to be aware of the sync conditions to know which contacts to return
- An endpoint was created where APSIS can send conditions to Site Shop
- Site Shop filters at query time
- Instead of returning 1 million contacts for APSIS to filter to 1 contact, Site Shop returns only that 1 contact
- **Result**: Full sync becomes incredibly fast

This exact pattern must be implemented for **FSC Enterprise and E.Deals**.

### API Endpoint Requirements

[Erik Andersson]: The CRM system must expose a new endpoint for sync condition updates. The structure is based on the **V1 sync condition schema**:

```
Endpoint: /sync-conditions (or similar)

Payload includes:
- entity: which entity this applies to (e.g., "contact")
- sendDeleteRequest: boolean - should the CRM send delete webhooks for contacts no longer matching?
- conditions: array of condition rules

Each condition rule:
- crmFieldName: the field name in the CRM (e.g., "first_name")
- operator: equality or similar
- value: the value to match
```

Example: "First name equals Elsa AND sync_to_apsis equals true"

**Additional requirement**: Consent filtering must also respect sync conditions. The CRM should only send consents for contacts that match the active sync conditions.

### Technical Complexity on CRM Side

[Erik Andersson]: Implementing this efficiently is non-trivial for the CRM:

- Pagination + cache updates for every condition change
- Can't be done entirely in real-time (would require constant recalculation)
- Needs some form of ready cache or SQL statement incorporation
- May involve significant query optimization

> "I fully acknowledge that handling this in an efficient way is not trivial because you have a lot of like pagination updates or cache updates or however you handle this because you can't really do this in real time."

But this work lives entirely on the CRM's side. **APSIS's implementation effort is minimal** — we already have the infrastructure to send sync conditions (proven with the generic connector and Site Shop).

---

## Generic Connector Capabilities

### Existing Support for Sync Condition Transmission

[Lukasz Grabowski asked if the generic connector already supports sending sync conditions]: [Erik Andersson]: Yes. The **Dynamics by Site Shop connector is built on the generic connector**, and it already implements sync condition transmission.

[Lukasz Grabowski verified]: The generic connector has this capability. Looking at the configuration:

```
Connector supports sync condition: true
```

This is exactly what FSC Enterprise 12.1 would need to use, since it's also built on the generic connector.

### Configuration vs. Enforcement Distinction

[Lukasz Grabowski asked]: If we enable this property in the generic connector, we'd still see the filter UI in APSIS front-end, but the filter wouldn't be used on APSIS's side. How does that work?

[Erik Andersson]: The configuration remains in APSIS for user-facing purposes, but the **enforcement moves to the CRM side**:

1. User configures sync conditions in APSIS UI (unchanged)
2. APSIS makes an API request to the CRM with these conditions
3. The CRM **enforces** the filtering on its side
4. Data returned to APSIS is already filtered

This is a **middle-ground solution** that emerged from earlier Maxo development discussions.

### Historical Context: Maxo Vision vs. Current Reality

[Erik Andersson]: When Maxo was being developed, the product vision was to move **all integration configuration out of APSIS** — field mappings, sync conditions, everything should be configured in Maxo (the CRM's configuration tool).

However, **no one wanted to spend the time to implement the UI in the CRM system**. So a compromise was reached:
- Configuration remains in APSIS (where users expect it)
- Enforcement happens in the CRM (for efficiency)

[Erik Andersson]: > "We had to settle for a middle ground where it was being enforced inside the CRM, but we still kept the configuration inside of APSIS."

In a perfect scenario, everything would be configured in one place. But given prioritization constraints, this hybrid model is the current state.

### Ideally: Configuration Should Be in the CRM

[Erik Andersson]: There's actually a strong argument for **why sync conditions should be configured in the CRM system itself**:

In APSIS, we have access only to fields mapped to subscriptions. In the CRM, you have access to **all related entity data**:

**Examples of conditions APSIS can't support today:**
- "Only sync contacts from companies with revenue > 500,000 crowns/year" (requires company data)
- "Only sync contacts with at least one consent" (requires direct consent table access)
- Complex joins across organizational hierarchies

> "You have access to a plethora of data that we don't have access to inside of APSIS. Let's say that you would only want to sync contacts from the CRM system which belong to a company where the revenue is higher than 500,000 crowns a year. This is nothing we can do in APSIS because we don't have access to organizational data, et cetera, because that's related entity data."

**But for now**, the hybrid model (configure in APSIS, enforce in CRM) is the practical solution.

---

## Why This Matters: Business and Stability Impact

### Cost and Performance Metrics

[Erik Andersson]: A full sync that takes 1.5 days instead of 15 minutes directly impacts:

- **Computing time**: Massive waste on APSIS infrastructure
- **Data transfer**: Excessive bandwidth between APSIS and CRM
- **Customer satisfaction**: Syncs taking 36 hours vs. 15 minutes
- **System stress**: Puts severe load on the CRM's infrastructure (as evidenced by the 504 timeouts)

For this customer with 100,000 contacts to sync: "100,000 contacts is like nothing for us" when not buried under 8 million irrelevant downloads.

### The CRM System's Perspective

[Erik Andersson]: > "It is not only a big issue for us, it is right now a bigger issue for the CRM system because they are giving us timeouts. So of course if they just give us what they have to, we will be done so much faster and like the sync will take less time, the customer will be happier and it will cost APSIS less money because it's less computing time and less data traffic."

This is a compelling business case: **fixing this benefits everyone** — APSIS saves compute, the CRM saves processing, the customer gets faster syncs.

### Scaling Trend Warning

[Erik Andersson]: This was not urgent before because the largest customers were manageable. But now:

> "It has not been so needed up until now because while we are wasting process time, like we can still download a million contacts and discard a lot like that. That's not a super big issue, but if we now have 3 customers that have over like 7 million contacts and consents like with this trend, that's not gonna go well for us."

---

## Next Steps and Change Management

### Step 1: Communicate with Product (via PS/Opry)

[Erik Andersson]: The first action is to present this issue to the **product team** (via PS, who owns customer relationships). This conversation must include:

1. **Setting expectations**: We've tested up to 2 million; now we need new capabilities for 8 million+
2. **Demonstrating the problem**: Show the 504 timeouts and the 1.5+ day sync duration
3. **Framing as multi-stakeholder benefit**:
   - APSIS: Lower compute costs, faster syncs
   - CRM (FSC Enterprise): Reduced load on their system
   - Customer: Syncs complete in reasonable time

[Lukasz Grabowski]: The broader challenge is cross-organizational alignment:

> "The bigger effort is on the communication and CRM side and of course how to persuade them that OK guys, you need to adjust your system. This is the most difficult thing right to talk to them and say hey we cannot do this on our site you need to do it otherwise we will fail."

### Step 2: Structured Escalation

[Lukasz Grabowski]: The proper flow should be:
1. R&D (Erik) talks to PS about the technical issue and requirements
2. PS escalates to their product/stakeholder meeting
3. **All parties** (APSIS, FSC, customer) meet to agree on solution
4. Prioritization and implementation planning

[Erik Andersson]: This is correct. Even if R&D wanted to be proactive, the heavy lifting must come from the CRM side, and that requires product-level commitment.

### Step 3: Support Process Discipline

[Erik Andersson]: A critical meta-issue: there are currently multiple routes for issues to reach R&D:
- Product help channel (proper route)
- Internal chat messages (people circumvent the system)
- Direct Slack/email to individuals

> "I've said now like I will not reply to internal messages because people are still trying to circumvent everything and coming directly to me. I am steering everything right now to the help channel, regardless of where it comes from."

[Erik Andersson]: This may seem harsh, but it's necessary:

> "We want to have a reliable flow for support cases. If this means us being annoying, sure, let's be annoying and then people can see that some new support process needs to be decided on."

**Recommendation**: Let support process failures surface (lack of response on product help channel → visibility of support scaling issue) rather than allowing R&D to be pulled in multiple directions.

[Lukasz Grabowski]: Agreed. A single unified flow prevents abuse of alternative routes.

---

## Parallel Short-Term Improvements

### FSC Enterprise Console Optimization

[Erik Andersson]: While the big solution (sync condition filtering) requires CRM implementation, there may be **quick wins on FSC's side**:

- Contact pagination is working — fetching 16,000 pages took time but succeeded
- Consent pagination is the bottleneck (500 entries per request, 22 seconds each)
- FSC may be able to do "simple optimizations" to pagination or caching

[Erik Andersson]: If FSC can optimize consent retrieval even moderately, syncs might complete in a few hours instead of 1.5+ days. Not ideal, but not catastrophic.

> "We can still maybe have not a perfect really well working solution, but maybe at least we can prevent the syncs from taking two days if they just optimize their pagination or caching or whatever."

**However**: The full solution requires the sync condition filtering feature.

---

## Documentation and Communication

### Platform Limitations Documentation

[Lukasz Grabowski raised]: There should be a documented list of **platform limitations** (max contact volume, max consent entries, etc.) in the docs or GitHub. This prevents customers from encountering surprise failures.

[Erik Andersson]: Currently, no such documentation exists. This should be added and referenced during customer onboarding.

### Ticket Status

There is an existing ticket in the issue tracking system (submariner). The team's position: **do not proactively work on this before hearing back from the product team**. Support should react, and if they don't, that highlights a support scaling issue that needs organizational attention.

---

## Key Takeaways

1. **Scale is the primary issue**: 8 million contacts + consent entries represents 4x the largest previously tested volume. Current architecture doesn't gracefully handle this.

2. **Architecture problem, not APSIS bug**: The integration must download all data to the CRM and then filter locally. This worked for smaller datasets but fails at scale. Consent timeouts are a CRM infrastructure issue, not a code defect.

3. **Sync conditions must move to the CRM**: The generic connector already has the infrastructure to send conditions to the CRM. FSC Enterprise needs to expose an endpoint to receive them and apply server-side filtering. This pattern already works for Site Shop.

4. **Immediate wins are limited**: Removing the audience export comparison optimization prevents future memory issues but doesn't solve current timeouts. FSC optimizing pagination is a band-aid.

5. **This is a product-level decision**: While R&D's effort would be minimal (enable an existing feature), the real work and prioritization decision belongs to the product team negotiating with the customer and FSC.

6. **Process discipline is necessary**: Issues must flow through proper channels (product help → PS → product meeting) rather than ad-hoc internal messages. This prevents R&D from being pulled in multiple directions and surfaces process scaling issues.

7. **Multi-stakeholder benefit**: Moving to server-side filtering benefits APSIS (less compute), FSC (less load), and customers (faster syncs). This is a strong business case for prioritization.

---

## Unresolved Questions and Action Items

### Action Items

1. **[Erik Andersson]**: Talk with PS/Opry to present the technical findings and prepare for escalation to product leadership
2. **[Erik & Lukasz]**: Consider including Lukasz in the PS conversation to provide integration architecture context
3. **[Erik Andersson]**: Highlight the consent pagination issue in the FSC Enterprise channel to see if quick optimizations are possible
4. **[Team]**: Document platform limitations (max contacts, max consent entries tested) in public docs
5. **[Product team - dependent on PS escalation]**: Schedule meeting with APSIS, FSC, and customer to agree on sync condition filtering implementation

### Unresolved Questions

1. **Does FSC have the capability to implement server-side sync condition filtering?** Unknown until PS discusses with FSC product team.

2. **What's the actual pagination implementation in FSC's consent endpoint?** Not investigated in detail; FSC team would need to review.

3. **Are there other CRM systems approaching similar scale?** Not mentioned, but this should inform broader platform planning.

4. **Should the audience export comparison be removed proactively, or only if/when large customers hit memory limits?** Deferred pending PS escalation outcome.

5. **Is there a maximum practical contact volume for APSIS given compute and infrastructure constraints?** Should be calculated and documented.