---
source_file: Efficy Enterprise issues.txt
domain: Apsis One - Integrations
topics: [Full Sync Performance, Sync Conditions, Consent Handling, CRM Integration Architecture, Generic Connector, Data Volume Scaling, On-Premise Deployments, Gateway Timeouts]
speakers: [Erik Andersson, Lukasz Grabowski, Tomasz Kowalski, Michal Rosikiewicz]
key_components: [Apsis One, FSC Enterprise, E.deal, Generic Connector, Sync Conditions, Consent Mappings, Full Sync, Audience Export, Integration Report Store, Installations Table]
session_type: debugging-session
---

## Session Overview

This session addresses critical performance and scalability issues identified in the Efficy Enterprise (FSC Enterprise) integration with Apsis One. The core problem: a customer with 8 million contacts (4x larger than any previously tested) is experiencing full sync failures, particularly around consent data retrieval, resulting in sync runs that last over 24 hours with 504 gateway timeouts from the CRM system. The team discusses architectural limitations in how sync conditions are currently implemented and outlines the need for CRM-side filtering to make large-scale integrations viable.

---

## Identifying the Customer and Integration Type

When investigating full sync failures reported in the product help channel, the process involves:

1. **Starting with customer display name** → lookup in back office
2. **Retrieving account ID** (not display name) from back office
3. **Querying the `installations` table in the database** to determine which integration the customer uses

[Erik Andersson]: "The reason we have to go to back office is because this is the like display name of the account, but I am interested in the account ID. The next step would be that we go to the database. In the database the best place to check this for which is also which the integration report store is also using is the installation table because in installations we have like a complete repository of like which integration is installed on which section and account etcetera."

```sql
SELECT * FROM installations WHERE ID = <account_id>
```

For the customer in question, this query revealed an **FSC Enterprise 2 installation**.

### On-Premise vs. Managed Deployment

[Erik Andersson]: "I know that because I've been told it is on premise, but we we have from integration side, we have no way of knowing that. Because like for for us it doesn't make any difference. Like this is an FSC enterprise environment. If this is managed by by FSC or not, we don't know and we don't care."

**Critical constraint for on-premise deployments:** Apsis One requires **public internet access** to the CRM system. VPN tunnels are not supported in the Apsis One environment (unlike in the Pro product). This is a fundamental prerequisite that many customers fail to satisfy before implementing Apsis One.

---

## Full Sync Architecture and Data Volume Challenge

### The Core Problem: 8 Million Contacts + Consent Data

This customer has:
- **8 million total contacts** in FSC Enterprise
- **1.2 million contacts** that match sync conditions across multiple sections
- **Expected output: ~100,000 contacts per section** after filtering
- **Each contact typically has 1+ consent records** (some customers have mapped 20 consent lists)

**The critical design flaw:** Sync conditions are evaluated **on the Apsis One side**, not on the CRM side. This means:

[Erik Andersson]: "This check this comparison here. This still happens on Apsis side. We still have to download all the data and then filter it on our side. So if you now look at this number again. You have 8 million contacts in the CRM system that is presented to us when we do the full sync. Out of these we will only sync 1.2 million, but that means that the full sync will still get to handle 8 million profile downloads, even though in the end we will only actually sync like if I understand this correctly, like one section here will have 100,000."

**Data processing scale:**
- 8 million contact records downloaded
- 8 million consent entries downloaded (in addition to contacts)
- Total entries to process and filter: **~16 million**

### Pagination and Gateway Timeouts

The full sync uses pagination to handle large datasets:

For this customer: **16,000 pages of profiles** were fetched during the contact phase. This worked, albeit slowly, because the CRM system could handle contact pagination.

**However, the consent retrieval is failing.** When attempting to fetch 500 consent entries at a time:
- **Gateway timeout (504) from the CRM system**
- Single request for 500 entries: **22 seconds to complete**
- Calculation: 8 million ÷ 500 × 22 seconds = **astronomically long sync time**

[Erik Andersson]: "When I tried to do this request on my own, it took 22 seconds to download 500 entries. So now picture 8 million / 500 times 22 seconds and we are up in not so pleasant numbers."

The sync that started at 4 PM was still running with timeouts at 11 PM (7+ hours) and continued beyond the observed timeline.

---

## Current Sync Condition Implementation

### How Sync Conditions Work Today

Sync conditions are configured in Apsis One back office and restrict which records should be synchronized. Example:

> "I only want to sync contacts where the e-mail matches eric@apsis-fake.com"

When this condition is set:
1. Apsis One receives the configuration
2. During full sync, **all 8 million contacts are still downloaded from the CRM**
3. Filtering happens **on Apsis One side** after download
4. Only matching records are synchronized to the subscription

### The Better Approach: CRM-Side Filtering (Proven Model)

**For Microsoft Dynamics/Site Shop**, a better pattern was already implemented:

[Erik Andersson]: "For Microsoft Dynamics for Site Shop, we developed a a new feature because Site Shop only wanted to sync contacts that fulfilled the sync conditions. But in order for site shop to know which contacts they should sync, they of course would need to be aware of the sync conditions. So we added a feature where when you configure it here inside of Apsys we make a request to site shop where we say you should only give us contacts where the e-mail field matches this value on site shop. They are then doing this filtering when we make a request. So instead of them giving us like 1,000,000 contacts that we need to do filtering on, they only give us one contact, thus making the full sync go incredibly, incredibly fast."

**Result:** Instead of downloading 1 million contacts and filtering to 1 matching contact, Site Shop returns only the 1 contact that matches the condition.

### Why This Must Be Done on CRM Side

For customers of this scale, downloading all data then filtering is **not feasible**:

[Erik Andersson]: "If we are to support customers of this size, the same thing has to be done inside of FSC Enterprise and E deal because it is not going to be feasible for us to sit and download 8 million entries, throw away 7,900,000 of them and only utilize these ones and then doing the same thing for consents."

**What needs to happen:**
1. Apsis One configures sync conditions in back office
2. Apsis One sends the sync condition to the CRM system via an API endpoint
3. The CRM system **enforces the filtering** on its side
4. When full sync requests data, the CRM returns **only filtered records**
5. When the CRM sends webhook updates, it only sends records matching the conditions

---

## Technical Implementation: Generic Connector Sync Condition Support

### FSC Enterprise Uses Generic Connector

FSC Enterprise 12.1 uses the **generic connector**, which already has support for sync condition transmission built in.

[Erik Andersson]: "Because the FSC Enterprise 12.1 is using the generic connector. It is as easy I hear saying. Connector supports sync condition true. This is the change that would need to be done in APSIS for this problem to be solved, but of course it won't work unless they have an endpoint to support it."

### Required Endpoint Structure

The generic connector sends sync conditions using the V1 sync condition endpoint. The structure includes:

```
V1 sync conditions per section:

- entity: contact (or case, etc.)
- send_delete_request_if_no_longer_matching: true/false (configurable in back office)
- conditions:
  - field: <CRM_field_name> (e.g., "first_name")
  - operator: equals/contains/greater_than/etc.
  - value: <value> (e.g., "Elsa")
  - apply_to_entity: contact/case/etc.
```

[Erik Andersson]: "The sync conditions here like we have the V1 sync condition and then we register it per section. And them here we would say like which entity is it for? Should they send us a delete request if they are no longer matching because this can be toggled inside of apps one in back office. And then what? What are the actual SIM conditions?"

**Example:** Sync only contacts where `first_name = "Elsa"` and `sync_to_apsis = true`

Consent filtering is also applicable:

[Erik Andersson]: "This is of course also applicable for consents. You should only present us consents that belong to a contact which fulfill the SIM conditions."

### The Effort Is Primarily on CRM Side

[Lukasz Grabowski]: "So you said that, yeah, we can, we can move filtering on the CRM side, yeah, but still we have here sync conditions, right?"

[Erik Andersson]: "So you you configure it here. And when you have configured here, we are making a request to the CRM system."

The configuration remains in Apsis One, but **implementation is on the CRM side**:
- Expose the new endpoint to accept sync conditions
- Implement filtering logic for full sync requests
- Implement filtering logic for webhook payloads
- Potentially maintain a cached list or SQL-based filtering

[Erik Andersson]: "I fully acknowledge that handling this in an efficient way is not trivial because you have a lot of like a lot of pagination updates or cache updates or however you you handle this because you can't really do this in real time because like you need to have some kind of ready cache that you uh you post to the um."

---

## Consent Data Handling and Memory Optimization

### Current Optimization: Existing Consent Comparison

The full sync includes an optimization to reduce unnecessary updates:

1. **Export all subscriptions from Apsis One** that have consent mappings
2. **Keep the export in memory**
3. **Compare incoming CRM consent data** against the in-memory export
4. **Only send updates** if consent status differs
5. **Skip unchanged consent** entries

[Erik Andersson]: "So as one step in the full sync is that we are doing a export from audience for all of the subscriptions that there is a mapping for. During the full sync we are doing a export to check what is the existing consent status inside of axis one. We keep that export in memory and then do a comparison."

### The Memory Problem at Scale

For the 8 million contact customer, this optimization will **crash due to out-of-memory errors**:

[Erik Andersson]: "This is working today based on the amount of profiles of course that the customer has because now let's say that they would have these 7-8 million profile entries in audience and they all have multiple. They all have multiple consent entries if we were to load that whole export into memory and do that comparison. I am fairly sure the sync will crash because of memory issues."

**Proposed solution:** Remove the optimization for large customers.

[Erik Andersson]: "So if we should support customers of this size, it is probably an optimization we should remove and just handle anything that the CRM system gives us because that is streamed in in contrast to this export comparison like as soon as we have downloaded it from the CRM system we put it into SQS and send it to the consumer to handle like we don't keep all of the downloaded messages in memory, but we handle the whole audience export in memory."

**Why this is safe:**
- CRM data is **streamed in** and immediately sent to SQS
- Not held in memory
- Audience export is **currently held entirely in memory** and compared
- Removing the comparison means: send everything from CRM → SQS → consumer handles

**For this specific customer:** This optimization isn't even breaking the sync because consent data never arrives due to gateway timeouts, so the memory issue hasn't manifested yet.

---

## The Core Root Cause: CRM System Stress

### Why Timeouts Occur

The CRM system (FSC Enterprise) has a **proxy server** in front of its business logic. When requesting consent data:

1. Apsis One requests 500 consent entries
2. **Pagination on CRM side is inefficient** (suspected implementation issue)
3. Request takes 22 seconds
4. After extended time, gateway times out (504 error)
5. HTML error response returned (indicating proxy timeout, not application error)

[Erik Andersson]: "I happen to know that FC has a proxy server in front of their business logic because you can see here that we are actually getting an HTML request back with the with the 504, so all the contact downloads have worked. The consent download is for sure not working as it should."

### Current Status for This Customer

- ✅ **Contact downloads:** Working (though taking 1.5+ hours for 8 million records)
- ❌ **Consent downloads:** Failing consistently with 504 timeouts
- ✅ **Contacts visible in Apsis One:** Yes (contacts were successfully downloaded and synced)
- ❌ **Consent data visible in Apsis One:** No (timeouts prevent consent retrieval)

---

## Documentation and Limitations

### Platform Limitations Not Currently Documented

[Lukasz Grabowski]: "Do we have somewhere written this limitations? Because there is a place in somewhere in Docs or GitHub where we keep limitations of our platform. So I think this should be documented there and then people they cannot expect from from us to to handle for example this amount of."

**Key limitation to document:** Previously tested full sync with 2 million contacts (largest customer at that time). Now supporting customers with 8 million+ contacts requires architectural changes.

---

## Path Forward and Escalation Process

### Step 1: Expectation Setting with Product

[Erik Andersson]: "Step one, I'm going to talk with PSI can include you if you want to be part of that and I think this this is the important part. So we like above all like we set expectations so this doesn't get out of hand like we can say like we've tested this previously with 2 million it is working but now moving forward we need to have this fix because as we can see here like the CRM system is not feeling well because of it."

### Step 2: Product Team Engagement

Product (PS) needs to understand:
- **Current state:** Testing has been limited to 2 million contacts
- **New reality:** Customers with 8 million+ contacts now being onboarded
- **Impact:** CRM system is failing under this load
- **Both teams affected:** Apsis One infrastructure AND customer's CRM infrastructure

[Lukasz Grabowski]: "I think they should talk to product, they should talk to product and highlight, highlight the issue and then product should decide, OK, so let's have all together meeting and look what we can do and then we need to agree on solution, right?"

### Step 3: Multi-Organization Effort

This is fundamentally a **cross-organization coordination problem:**

[Erik Andersson]: "You will find that this is essentially always going to be the biggest issue that is like cross organization efforts because of course everyone always wants to protect their own time and wants to commit to as little as possible."

**Persuasion strategy:** Frame as a **business and stability issue**, not just a technical request.

[Erik Andersson]: "But but usually when you can present it in this way as we have like we are wasting enormous amount of computing processing time like we can press on this being a like money issue. And above all, like a stability issue because we can see here like the CRM is not feeling good from this, so we can show them like this is going to benefit you a lot because we are gonna put less stress on your on your system."

**Benefits to FSC Enterprise team:**
- Less stress on their infrastructure
- Fewer timeouts and failures
- Full syncs reduced from 1.5+ days to ~20 minutes for 100k records

**Benefits to Apsis One:**
- Lower infrastructure costs
- Better customer experience
- Fewer support escalations

### Step 4: Support Process Discipline

**Critical:** Do not circumvent the official support channel process.

[Erik Andersson]: "I will not reply to internal messages because people are still trying to circumvent everything and coming directly to me and this is right now also particularly bad because then you don't get to see what I am doing. So I am steering everything right now to the help channel, regardless of where it comes from."

[Erik Andersson]: "We want to have a reliable flow for support cases. If this means us being annoying, sure, let's be annoying and then people can see that it some new support process needs to be decided on."

---

## Additional Optimization Opportunities

### FSC Enterprise Consent Pagination

Beyond the sync condition filtering (primary fix), FSC Enterprise may be able to implement **quick pagination optimizations:**

[Erik Andersson]: "From what we can see like the contact downloads like it's going fine there there are seemingly no issues there like of course it is taking like over 1 1/2 hour, but with 8 million profiles, like I don't think that's an issue. But of course as soon as we start downloading consents, then all hell breaks loose. So there it might be that they can do some simple optimizations."

These could include:
- Caching optimizations
- Query optimization
- Connection pooling improvements

[Erik Andersson]: "So we can still maybe have not a perfect really well working solution, but maybe at least we can prevent the sinks from taking two days if they just optimize their pagination or caching or whatever."

However, **pagination optimizations are secondary** to implementing sync condition filtering at the CRM level.

---

## Design Philosophy: Where Configuration vs. Enforcement Lives

### The Evolution of Integration Configuration

In the early Maxo development, there was a push to move **all** configuration to the CRM system:
- Field mappings in CRM
- Sync conditions in CRM
- Subscription mappings in CRM

[Erik Andersson]: "In when Maxo was being developed, then like everyone wanted like the integration page in in Apsis to essentially disappear. They wanted to set up the field mappings inside of Maxo, they wanted to set up the sync conditions inside of Maxo, etcetera."

### Current Compromise

No one wanted to prioritize building the UI in the CRM system, so the current design is a **middle ground:**

[Erik Andersson]: "But no one wanted to actually spend the time of implementing the UI etcetera in the CRM system because like no one wanted to prioritize like anything. So we had to like settle for a middle ground where it was being enforced inside the CRM, but we still kept the configuration inside of inside of APSIS."

**Today's reality:**
- ✅ **Configuration:** Apsis One (field mappings, subscription mappings, sync conditions)
- ✅ **Enforcement:** CRM system (where applicable)

### The Ideal Long-Term Approach

[Erik Andersson]: "In in the best of worlds you would have wanted to set up everything in one place."

CRM-side configuration is actually preferable for sync conditions because CRM systems have access to **related entity data** that Apsis One doesn't:

[Erik Andersson]: "There is a point in actually setting up sync conditions inside the CRM system instead, because there you have access to a plethora of data that we don't have access to inside of APSIS. Let's say that you would only want to sync contacts from the CRM system which belong to a company where where the revenue is higher than 500,000 crowns a year. This is nothing we can do in abscess because we don't have access to like organizational data."

### Common Requests Unsupported in Current Design

[Erik Andersson]: "You might also want to say I only want to sync contacts to APSIS that have at least one consent, and that's a very common request that we've had. You cannot do the synapsis today either, but you could very easily set this up in the CRM system because then you have direct access to it like you can make very complex and nice sync conditions, but because no one wanted to like spend time on implementing it like we're here, here's where we are."

---

## Key Takeaways

1. **Scale Problem is Real:** The 8 million contact customer is 4x larger than any previously tested, exposing architectural limitations in sync condition filtering and consent data retrieval.

2. **Root Cause:** Sync conditions are evaluated **on Apsis One side** (after downloading all data), not on the CRM side. This forces Apsis One to download, process, and filter enormous datasets.

3. **Proven Solution:** Microsoft Dynamics/Site Shop already uses **CRM-side sync condition filtering**, making syncs dramatically faster. This same approach must be implemented in FSC Enterprise.

4. **Dual Impact:** This isn't just an Apsis One problem. The CRM system itself is failing (504 timeouts) under the stress of providing 8 million records + consents. Fixing this benefits both systems.

5. **Secondary Concern:** The in-memory consent comparison optimization will cause memory outage crashes at scale and should be removed.

6. **Documentation Gap:** Platform limitations (tested up to 2M contacts, not suitable for 8M+) must be documented.

7. **Cross-Org Effort:** Success requires product team engagement, customer (FSC Enterprise) commitment, and clear communication of business/stability benefits.

8. **Process Discipline:** All issues must flow through proper support channels, not direct escalations, to maintain visibility and accountability.

9. **Configuration vs. Enforcement:** While Apsis One will continue to host configuration UI, enforcement (filtering) must happen on CRM side for scalability.

10. **Competitive Risk:** Without this fix, Apsis One will not be viable for large enterprise deployments with millions of contacts.

---

## Unresolved Questions and Action Items

### Open Questions

1. **What customizations can FSC Enterprise implement for quick pagination optimization?** (Secondary benefit even if sync condition filtering is also done)
2. **How complex is it for FSC Enterprise to add the sync condition filtering logic?** (Depends on their architecture)
3. **Do E.deal and other enterprise CRMs have the same limitations?**

### Action Items

| Owner | Task | Priority |
|-------|------|----------|
| Erik Andersson | Schedule meeting with Opri (Product/PS) to present the problem and strategy | High |
| Erik Andersson | Highlight consent pagination issues in FSC Enterprise channel to request quick optimizations | Medium |
| Lukasz Grabowski (optional) | Participate in initial product team discussion | High |
| Product Team | Escalate sync condition filtering requirement to FSC Enterprise | High |
| Documentation Owner | Document platform limitations: tested up to 2M contacts, larger deployments require CRM-side filtering | Medium |

### Ticket Status

- ⏳ **Current ticket:** In submariner, marked for product team action
- **Recommendation:** Do not proactively resolve; await product team escalation and customer feedback
- **Reason:** This requires CRM-side changes that only customer/product can drive