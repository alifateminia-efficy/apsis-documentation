---
source_file: Efficy Enterprise issues.txt
domain: Apsis One Integrations
topics: [Full sync performance degradation, Sync conditions and filtering, Consent data handling, Large dataset processing, On-premise vs cloud deployment, Gateway timeouts, Memory optimization]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Apsis One, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Generic Connector, Microsoft Dynamics (legacy), Dynamics Site Shop, Sync Conditions, Field Mappings, Consent Mappings, Full Sync, Pagination]
session_type: debugging-session
subdomains: [Architecture, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration]
---

## Session Overview

This knowledge transfer session addresses critical performance issues with Efficy Enterprise full sync operations, specifically triggered by three affected customers with very large contact databases (7-8 million contacts). The core issue stems from architectural limitations where Apsis One must download and filter all data locally rather than pushing filtering logic to the CRM system. The discussion covers the gap between what was previously load-tested (2 million contacts) and current production requirements, explores workarounds and long-term architectural solutions, and defines the steps needed to escalate this to product leadership.

---

## Understanding Customer Integration Types and Discovery Process

### Identifying Integration Type for a Customer

When a customer reports sync issues, the first step is identifying which integration they're using:

1. **Start with customer name** in the support ticket or issue report
2. **Look up in back office** to find the account display name (this is human-readable but not sufficient)
3. **Query the database** using the account ID (not the display name) to get precise information

The correct database query uses the **installations table**, which serves as the "complete repository of which integration is installed on which section and account":

```
SELECT * FROM installations WHERE ID = [customer_account_id]
```

[Erik Andersson]: This returns the actual integration type installed, such as "FSC Enterprise 2" (formerly Efficy).

### On-Premise vs Cloud Deployment

An important caveat: **the Apsis One integration layer has no built-in way to determine if an Efficy installation is on-premise or cloud-managed**. This information must come from external communication with the customer or account team.

However, on-premise deployments create predictable problems:

- **IP whitelisting and network access requirements** are more common
- **Public internet access is mandatory** for Apsis One to function—we do not support VPN tunnels for Efficy/FSC integrations (unlike some other products)
- Customers often say "we have this on-premise and want to use Apsis" but then refuse to expose their system to public internet access, creating an impossible situation

[Erik Andersson]: > "We need to have public access to the environments. If you are to use Apsis, we don't support VPN tunnel. If they want to use on-premise, we need to be able to call it from the Internet for now period. Otherwise you can't use Apsis."

---

## The Scale Problem: From 2 Million to 8 Million Contacts

### Previous Load Testing Limits

Apsis One was previously load-tested with **2 million contacts**. At that time, this represented the largest customer base, and full syncs completed successfully (albeit slowly).

**Current situation**: One newly onboarded customer has **8 million contacts**—4x the tested volume—plus **1.2 million of these should be synchronized** across multiple sections (with ~100,000 contacts per section depending on sync conditions).

This massive discrepancy between tested and actual production loads has exposed fundamental architectural limitations.

---

## Architectural Issue: Sync Conditions Currently Evaluated on Apsis Side

### Current Design: Download All, Filter Locally

Sync conditions allow users to specify which contacts should be synced. For example:

```
Sync only contacts where: email matches "eric@apsis-fake.com"
```

However, the current architecture evaluates this filter **on Apsis One's side**, not the CRM's side:

1. Full sync requests all 8 million contacts from Efficy Enterprise
2. Apsis One downloads all 8 million profiles (this works via pagination with ~16,000 pages)
3. Apsis One then filters locally, keeping only ~1.2 million that match the sync condition
4. ~6.8 million contacts are discarded

[Erik Andersson]: > "This check, this comparison here, still happens on Apsis side. We still have to download all the data and then filter it on our side."

The rationale for this design: **All integration configuration was intentionally centralized in Apsis One** (field mappings, sync conditions, consent mappings) rather than requiring CRM-specific UI configuration.

### The Data Volume Explosion with Consents

The problem multiplies when consents are added:

- 8 million contact entries downloaded
- Each contact typically has at least 1 consent entry (observed: customers with up to 20 consent mappings)
- Realistic scenario: 8 million contact entries + 8 million consent entries = **16 million total records processed**

[Erik Andersson]: > "Usually like each contact has at least one consent. They can quite easily have two consent. I have seen customers that have 20 consent lists mapped, but say for the sake of it that each one has one consent. That means that you have 8 million consent entries add on top of this 8 million contact entries."

---

## The Consent Timeout Problem: Gateway Failures

### Observed Failures

When the system attempts to fetch consent data in batches of 500 entries:

- **Gateway 504 timeout** errors from Efficy Enterprise
- Manual test: downloading 500 consent entries took **22 seconds**
- Math: (8 million / 500) × 22 seconds = **approximately 320,000 seconds** (~89 hours of processing time)

A real-world sync that started at 4 PM was still failing at 11 PM+ due to consent timeouts.

[Erik Andersson]: > "As soon as we requested like 500 consent and consent entries, we were smacked with a gateway timeout from the CRM system and when I tried to do this request on my own, it took 22 seconds to download 500 entries."

### Root Cause Analysis

The 504 responses are HTML gateway timeouts, not JSON errors, indicating:

- Efficy has a proxy server in front of its business logic
- Pagination implementation in Efficy is extremely slow for consent data
- The proxy hits its timeout limit before Efficy can deliver results

[Erik Andersson]: "I happen to know that FC has a proxy server in front of their business logic because you can see here that we are actually getting an HTML request back with the 504."

### Why Contacts Work But Consents Don't

Contact pagination appears to be optimized (16,000 pages downloaded successfully), but consent pagination is not. This suggests:

- Different endpoint implementation or resource allocation for consent data
- Possible indexing or query optimization issues in Efficy's consent storage
- The proxy may have different timeout settings per endpoint

---

## Solution Approach 1: Push Filtering to the CRM Side

### Precedent: Microsoft Dynamics and Site Shop

For **Microsoft Dynamics with Site Shop** (which uses the generic connector), Apsis One implemented a different approach:

1. User configures sync conditions in Apsis One UI
2. Before making a full sync request, Apsis One sends the sync condition to the CRM system
3. The CRM system internally applies the filter
4. When Apsis One requests contacts, the CRM returns only filtered results (1 contact instead of 1,000,000)

Result: **Full sync completes in minutes instead of hours/days**.

[Erik Andersson]: > "When you configure it here inside of Apsis, we make a request to Site Shop where we say you should only give us contacts where the email field matches this value. On Site Shop, they are then doing this filtering when we make a request. So instead of them giving us like 1,000,000 contacts that we need to do filtering on, they only give us one contact, thus making the full sync go incredibly, incredibly fast."

### Efficy Enterprise 12.1 Support

Efficy Enterprise 12.1 uses the generic connector, the same connector as Dynamics Site Shop.

In the generic connector configuration:

```
Connector supports sync condition: true
```

This property can be enabled for Efficy Enterprise 12.1, but **it will only work if Efficy exposes a new API endpoint** to receive and enforce sync conditions.

### Required CRM-Side Changes

Efficy Enterprise must implement:

1. **New API endpoint** to receive sync condition configuration:
   - V1 endpoint structure similar to Dynamics implementation
   - Include entity type (contact, case, etc.)
   - Include delete request handling (whether to send delete webhooks for contacts no longer matching)
   - Include the actual sync condition clauses (field name in CRM, operator, value)

2. **Filtering logic** to apply conditions:
   - During full sync requests: return only matching contacts
   - During webhook requests: only send webhooks for contacts that match conditions
   - Must handle pagination efficiently

3. **Consent filtering**:
   - Only return consent data for contacts that match sync conditions
   - Critical for avoiding 16 million record downloads

[Erik Andersson]: > "The sync conditions here, we have the V1 sync condition and then we register it per section. And them here we would say like which entity is it for? Should they send us a delete request if they are no longer matching because this can be toggled inside of Apsis One in back office. And then what? What are the actual sync conditions? So here the fields here. This is the CRM name for the field."

### Implementation Complexity Acknowledgment

[Erik Andersson]: > "I fully acknowledge that handling this in an efficient way is not trivial because you have a lot of like a lot of pagination updates or cache updates or however you handle this because you can't really do this in real time because like you need to have some kind of ready cache that you post to us or if you can somehow incorporate this into an SQL statement."

Possible Efficy-side implementations:
- SQL WHERE clauses incorporated at the query level
- Cache-based filtering with periodic updates
- Scheduled sync condition evaluation

---

## Solution Approach 2: Optimize Consent Export Memory Handling

### Current Optimization: In-Memory Comparison

During a full sync, Apsis One performs an optimization step:

1. **Export existing subscriptions** from Apsis One for all mapped consent types
2. **Load entire export into memory**
3. **For each incoming consent from CRM**: compare against in-memory export
4. **Only send consent updates** if values differ
5. **Skip unchanged consents** to reduce message volume

This works fine for customers with millions of contacts **if those contacts are not in Apsis One yet**, because the export would be empty or small.

### The Memory Risk with Large Existing Datasets

However, **if the customer runs another full sync and Apsis One already has 7-8 million profiles with consents**:

1. Export would contain 7-8 million consent records
2. Loading all into memory before comparison could cause **out-of-memory crashes**

[Erik Andersson]: > "If we were to load that whole export into memory and do that comparison, I am fairly sure the sync will crash because of memory issues."

### Proposed Fix: Stream Processing Instead

Remove the in-memory comparison optimization for large syncs:

- CRM consent data is already streamed in (received in chunks)
- Send each consent update to the message queue (SQS) as soon as it arrives
- Let the consumer service handle duplicates and updates
- Trade off: slightly more message queue traffic, but avoids memory exhaustion

[Erik Andersson]: > "If we should support customers of this size, it is probably an optimization we should remove and just handle anything that the CRM system gives us because that is streamed in in contrast to this export comparison."

### Immediate vs Long-Term Relevance

For the current 8 million contact customer: **This optimization isn't the breaking issue right now** because consents aren't being received (they're timing out). But once Efficy fixes its pagination, this becomes critical.

---

## Documentation and Platform Limitations

### Missing Limits Documentation

Apsis One lacks documented **platform limitations** for data volume and sync scope:

[Lukasz Grabowski]: > "There is a place somewhere in Docs or GitHub where we keep limitations of our platform. So I think this should be documented there and then people they cannot expect from us to handle for example this amount."

Recommendations:
- Document tested limits (2 million contacts, N consents per contact)
- Document current architectural constraints
- Provide guidance on when to implement filtering
- Update as scale limits change

---

## Management and Escalation Strategy

### Setting Expectations: The Critical First Step

This issue cannot be solved quickly because it requires **cross-organization coordination**:

- Efficy Enterprise engineering must prioritize endpoint development
- API design must be agreed upon
- Implementation must be tested
- Apsis One must coordinate on timeline

[Erik Andersson]: > "Step one, I'm going to talk with Opri [Product]. We like above all like we set expectations so this doesn't get out of hand. We can say like we've tested this previously with 2 million it is working but now moving forward we need to have this fix because as we can see here like the CRM system is not feeling well because of it."

### Recommended Escalation Path

1. **Erik to Product (Opri)** with technical findings
2. **Product to Efficy** with business/technical case
3. **Joint meeting** to agree on solution timeline
4. **Efficy implementation** with Apsis One validation

### Business Case Elements

When escalating, emphasize:

- **Cost impact**: Running 8 million + 8 million consent syncs wastes computing resources, data transfer, and time
- **Stability impact**: Current implementation stresses Efficy's system, causing 504 timeouts and potentially degrading their entire platform
- **Customer impact**: Syncs taking 1.5+ days instead of 15-20 minutes result in poor customer satisfaction
- **Industry trend**: If this customer has 8M contacts, others likely will too

[Erik Andersson]: > "We can press on this being a money issue. And above all, like a stability issue because we can see here like the CRM is not feeling good from this, so we can show them like this is going to benefit you a lot because we are gonna put less stress on your system."

### Why Speed of Implementation Matters

[Lukasz Grabowski]: > "The bigger effort is on the communication and CRM side and of course how to persuade them that OK guys, you need to adjust your system. To fix it. So this is the most difficult thing right to talk to them and say hey we cannot do this on our site you need to do it otherwise we will fail and then they can say OK so then you cannot fulfil our needs. So maybe we should leave you."

---

## Support Process Discipline

### Enforcing the Support Channel Flow

This incident revealed a problematic pattern:

- Account managers posting directly to product help channel
- Developers expected to monitor multiple channels (Slack DMs, product help, emails, internal chats)
- No coordination or prioritization mechanism
- Risk of issues being missed or duplicated

[Erik Andersson]: > "This is a dangerous thing. So like I I would still really suggest like if you are to look into it, only do things if it comes from socket and support to our help channel or otherwise like essentially ignore it because we can't sit and have stories from the product help channels, stories from internal chat, stories from emails."

### Recommendation: Unified Intake

[Erik Andersson]: > "I am steering everything right now to the help channel, regardless of where it comes from. And if this means that I will upset people, sure, let them be upset, but we need to have one unified flows for it."

Failing to enforce this will:
- Create precedent for bypassing support process
- Lead to missed critical issues
- Overload individual developers
- Make visibility impossible for management

---

## Immediate and Short-Term Actions

### What Apsis One Can Do Now (Limited Impact)

1. **Document the finding** that Efficy's consent pagination has severe performance issues
2. **Check for quick optimizations** in Efficy's proxy/caching configuration (though we have no control here)
3. **Remove the in-memory consent comparison optimization** to prevent future crashes (future-proofing)
4. **Highlight the issue in the enterprise channel** to encourage Efficy to investigate their pagination

[Erik Andersson]: > "I'm going to highlight this issue in the enterprise channel because from what we can see like the contact downloads like it's going fine there. But of course as soon as we start downloading consents, then all hell breaks loose. So there it might be that they can do some simple optimizations."

### What Requires Efficy Enterprise Changes (Required for Real Fix)

1. Implement V1 sync condition API endpoint
2. Add filtering logic to full sync and webhook operations
3. Optimize consent pagination (if possible within their architecture)
4. Test with 8+ million contact datasets

---

## Unresolved Questions and Action Items

### Action Items

- [ ] **Erik** to schedule call with Opri (Product) to review findings and discuss escalation approach
- [ ] **Erik** to post technical findings to enterprise integration channel (internal Slack/Teams)
- [ ] **Product team** to arrange meeting with Efficy Enterprise engineering
- [ ] **Apsis One development** to remove in-memory consent export optimization (pending scheduling)
- [ ] **Documentation team** to add platform scalability limits to Docs/GitHub

### Unresolved Technical Questions

- **Exact cause of Efficy consent pagination slowness**: Is it indexing, query complexity, or proxy timeout configuration?
- **Feasibility of quick Efficy-side optimizations**: Can they achieve acceptable performance without architectural changes?
- **Timeline for Efficy API implementation**: How long would V1 sync condition endpoint take to develop?
- **Customer willingness to wait**: Will the 8M contact customer accept a multi-month solution timeline?

---

## Key Takeaways

1. **Scale changes everything**: The architecture works at 1-2 million contacts but breaks at 8 million. This is now a business-critical issue because customers at this scale exist.

2. **Filtering at the source is essential**: Pushing sync condition filtering to Efficy would reduce data transfer by ~87% (8M → 1.2M contacts) and prevent the cascade of downstream problems. The technical implementation is non-trivial but necessary.

3. **Consent handling is the bottleneck**: Contact downloads work; consent downloads fail. This is Efficy-specific and requires their investigation.

4. **Support process discipline prevents chaos**: Multiple communication channels for issues create missed problems and unpredictable workload. Enforce a single entry point.

5. **Cross-org coordination is the real challenge**: The solution is technically straightforward but requires alignment between three organizations with competing priorities. Managing expectations early is critical.

6. **Document scalability limits**: Customers and sales need to know what volumes are supported and what architectural changes are needed for larger datasets.