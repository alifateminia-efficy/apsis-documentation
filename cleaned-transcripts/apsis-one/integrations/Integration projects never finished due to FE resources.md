```yaml
source_file: Integration projects never finished due to FE resources.txt
domain: Apsis One Integrations
topics:
  - Full sync report improvements
  - Extended sync conditions
  - Large customer handling (2M+ contacts)
  - Tribe connector pagination issues
  - Tribe lead entity confusion
  - CRM system integration architecture
speakers:
  - Erik Andersson (outgoing domain expert)
  - Lukasz Grabowski (team lead)
  - Tomasz Kowalski (developer)
  - Michal Rosikiewicz (developer)
key_components:
  - Full sync reports
  - Sync conditions
  - Generic connector
  - Tribe connector
  - Microsoft Dynamics (Site Shop)
  - FSC Enterprise
  - Segment builder
  - SQS (Simple Queue Service)
  - Internal broker service
  - Audience module
  - CRM pagination
session_type: knowledge-transfer
subdomains:
  - "Lead creation"
  - "Different Types of Connectors"
```

## Session Overview

Erik Andersson conducts a knowledge transfer session covering three incomplete integration projects that were deprioritized due to front-end resource constraints. The session documents technical debt in the Apsis One Integrations domain, including problems with sync reporting clarity, limited sync condition filtering capabilities, scalability issues with large customers (2M+ contacts), and stability problems with the Tribe connector. The team discusses prioritization of remaining work and handoff plans.

---

## Incomplete Project 1: Improved Full Sync Report

### Current Problem with Sync Statistics

The current full sync report conflates two distinct message types in a single success count, creating confusion for customers and support teams:

> When it says here successful 39,000 customers tend to think that this is 39,000 contacts or profile updates, then they wonder like why? Why are you syncing so many profiles? I only have like 8,000 or 10,000 of them, but these 39,000 they are profile updates plus consent messages.

This ambiguity affects not only customers but also NPS and account managers who interpret the statistics.

### Completed Backend Work

The data model has been separated in the backend to report these metrics independently:

- **Amount of profile messages**: created, successful, failed, skipped
- **Amount of consent messages**: created, successful, failed, skipped

This allows granular visibility but the changes have not been exposed to the frontend UI because:

> Josh left the company and then and the integration stores were always down prioritised in favour of e-mail and audience. So this change is not in place for the front end, but the work is done in in back end.

### Frontend Implementation Gap

The backend separation is complete and partially tested but needs front-end UI work to display the granular metrics. [Erik Andersson]: This is identified as a **low-hanging fruit** for a swift front-end developer—high value for customer support reduction with minimal effort.

### Testing Status

The feature has been tested during backend release but has not received full validation:

> This needs to be more tested because we've, I mean we've not had the real ability to fully test it, but it is working as far as as far as we saw when we tried it for the back end release.

---

## Incomplete Project 2: Extended Sync Conditions

### Current Limitations

The sync conditions feature has two major weaknesses:

**Weakness 1: Implicit AND only**
All conditions use an implicit AND operator:
```
email = "user@example.com" AND mobile = "123456" AND dateOfBirth = "yesterday"
```

Customers cannot create OR conditions or negate criteria.

**Weakness 2: Only equals operator supported**
```
email = "value"  // supported
email != "value"  // not supported
email CONTAINS "value"  // not supported
email NOT_EMPTY  // not supported
```

### Backend Implementation Complete

The backend has been extended to support segment builder logic with AND/OR operators:

> We started supporting the segment builder formats in the sync conditions, but we extended it a bit and added support for and or and that was not so easy to do but it's there in the back end. But as you can see it is not here on the front end because this needs to be changed quite radically.

Prem has tested the backend implementation. The new operators include:
- equals
- not equals
- contains
- starts with
- ends with

### CRM System Evaluation: Split Implementation

The sync conditions must be evaluated either in APSIS or delegated to the CRM system. This has led to a split implementation:

#### Current Approach (Version 1)
Uses an endpoint in the generic connector that sends conditions to the CRM:

```
Endpoint: /sync-conditions
Body:
{
  "entity": "contact",
  "conditions": {
    "firstName": "Elsa",
    "syncToApsis": true
  }
}
```

**Supported by**: All CRM systems (generic connector standard)  
**Limitation**: Only supports implicit AND, only equals

#### New Approach (Version 2)
Tailored endpoint for complex condition structures with AND/OR support:

```
Endpoint: /sync-conditions-v2
Body:
{
  "entity": "contact",
  "conditions": {
    "operator": "AND",
    "operands": [
      {"field": "firstName", "operator": "equals", "value": "Elsa"},
      {"field": "email", "operator": "notEmpty"},
      {
        "operator": "OR",
        "operands": [
          {"field": "source", "operator": "equals", "value": "web"},
          {"field": "source", "operator": "equals", "value": "api"}
        ]
      }
    ]
  }
}
```

**Currently supported by**: Microsoft Dynamics (Site Shop) only  
**Status**: Site Shop is the only CRM that has implemented support for v2 structure

### Critical Dependency: CRM System Support

**[Erik Andersson]:** Before enabling v2 for any account, Site Shop must verify that their implementation handles the new structure:

> Before this is actually enabled for any account like the V2 version of the sync conditions, this needs to be verified with site shop because I don't think they have actually implemented support for the new structure because it never took off inside of inside APSIS.

### Ripple Effects on Enterprise Systems

FSC Enterprise is being brought into the picture because they have massive customer bases (8M+ contacts) and will need sync condition support:

> Me and Walker had discussions with enterprise that they will also need to support sync conditions now that we have like this massive 8 million contacts CRM, so they will need, they will need to be involved in this as well.

### Recommendation for Prioritization

> Don't rush with this feature finished off like all other on the list first because it does have some consequences in the other CRM systems as well.

---

## Incomplete Project 3: Large Customer Scalability (2M+ Contacts)

### The Core Problem

Currently, sync conditions are evaluated in APSIS after all contacts are downloaded from the CRM. For customers with millions of contacts, this creates catastrophic data waste:

> The sync conditions today, they are evaluated inside of APSIS. That means that we need to download all of the contacts, so if they have 8 million but we only want 100,000 of them, we still need to download 8 million of them, which is an insane data waste and time waste for the customer.

### Two-Track Solution Required

**Track 1: Delegated Evaluation (Site Shop approach)**
The CRM system evaluates conditions and paginates only matching results:
- Store conditions as configuration in APSIS
- CRM system evaluates conditions on its side
- CRM generates pages based on filtered results

**Track 2: Stateless API (FSC Enterprise approach)**
Include sync conditions in every request to a stateless API:
- No server-side state in the CRM system
- Pass sync conditions as query parameters or request body
- CRM system filters on-the-fly for each request

> One way is to do as site shop does where they where we like store the conditions as a configuration and they evaluate them or generate the pages based on that. The second approach is as FSC Enterprise is doing like they have a stateless API, so in their case we will need to include the sync conditions when we ask for contacts for them.

### Memory Optimization Within Full Sync

There is a critical memory issue in the full sync process that must be addressed independently:

> In the full sync we are first downloading all of the contacts from the CRM system and we create the profile update messages for them. In the second step of the full sync, we are producing all of the consents and one part of this is that we trigger consent exports from audience for each subscription. But we store the result of all of this in memory and if you have a massive load of profiles then this will just utterly crash the full sync due to memory issues.

#### The Problematic Optimization

An optimization exists that compares incoming consent status with what's already in APSIS. This caching is done in memory, which fails at scale:

> What we are doing with it is that we compare the consent status of the income and consent and how it looks like in APSIS.

#### Proposed Solution: Stream Processing

Remove the memory optimization and stream messages directly to SQS:

> What we want to do after this is we just want to stream everything like take the profile update messages, convert them to the correct format, put them in in SQS and then let the consumer take them as they come. There is no issue with a lot of profile updates being in SQS, there's a lot of problem having them in memory in the producer.

**Benefits of streaming approach**:
- Eliminates memory crashes
- Improves customer transparency: they see the same consent counts across successive syncs only if data hasn't changed (no mysterious count reductions)
- Better observability and troubleshooting

[Lukasz Grabowski]: "From those 3 epics, my vote is that that one is the most important and it is high prior to to plan development."

[Erik Andersson]: "Yes, absolutely... the full sync report... it has been pretty much like that since the release. There is no issue waiting a bit more with it, whereas these the size of the customer is a kind of big problem."

---

## Discovered Issue: Tribe Connector Pagination Duplicates

### The Bug Report

A customer using the Tribe connector reported a mismatch:

- Downloaded contacts from CRM: 272
- Tags in APSIS after sync: 204
- Customer expectation: these numbers should match

### Root Cause Investigation

[Erik Andersson] investigated by:

1. Downloaded the list from Tribe's API directly: confirmed 272 records
2. Checked the last stored sync snapshot: confirmed 204 records from previous sync
3. Re-downloaded using pagination (50 contacts per page): discovered duplicates in paginated responses

> When I downloaded 50 contacts and then 50 and then 50 and then 50 and then 50, then I noticed that some of the example CRM IDs were not included in the total set of the data. And what had happened is that Tribe has given us duplicates on the pagination.

**Scale of duplicates**: Out of 272 paginated results, approximately 68 were duplicates.

> Out of those 272 there were like 68 duplicates. We compared the sets and this was like quite consistent.

### Impact Assessment

This pagination bug affects:
- **Profile list syncs**: currently verified
- **Full sync**: likely affected but not yet confirmed
- **Apsis side**: no issue—the system accepts duplicate IDs without complaint
- **Customer visibility**: not immediately obvious without careful analysis

### Ball in Tribe's Court

There is nothing Apsis can do to work around this at the pagination level:

> There's nothing Apsis can do about this. Of course you can increase the page size, but like that's not the good solution because we don't know how many profiles will be on one in in one list in the CRM system. So we will need to utilize pagination one way or another and then doesn't matter how much we increase page size essentially.

This is a known issue to flag for Tribe support; they must fix their pagination implementation.

---

## Tribe Connector: Virtual Lead Entity Problem

### The Confusion

The Tribe connector configuration in APSIS includes a **lead entity** that doesn't meaningfully exist in Tribe:

> For tribe you can select here on the outbound mappings a subscription you have this lead entity. This lead entity as I described before, this is not like something which really exists inside of tribe, because in tribe for the outbound mappings you have a drop down list which selects like which entity should I create from form submissions inside of Apsis, but the generic connector can't really handle like dynamic entities.

The generic connector architecture requires knowing which entity to download attributes for at configuration time. It cannot handle dynamic entity selection that changes per request.

### Discovery: Tribe Always Responds with Contact

Recently discovered: regardless of what entity type is selected in Tribe's UI dropdown, Tribe always responds to APSIS with entity type = **contact**:

> We did discover quite recently also that no matter what you select in that drop down list inside of tribe, tribe is always responding saying this is a contact. And then they are handling this inside of tribe. And that means that this whole lead entity set up inside of Apsis is worthless and just creates confusion.

### Why This Matters

If APSIS tries to send a form submission with entity type "lead" but Tribe only accepts "contact", the system fails to create the CRM ID relationship. This leads to **duplicate profiles**:

1. Form submission arrives → APSIS creates a profile
2. APSIS sends to Tribe with entity="lead"
3. Tribe responds with entity="contact" or rejects it
4. No CRM ID attached to the new profile
5. Full sync runs → downloads the contact from Tribe
6. Now there are two separate APSIS profiles for the same person

> Or rather we will create duplicates I should say because we will get the form submissions from them. There will be a profile created from the form submission. Then we will try to send this to the CRM system and we will not do any merge. We will not add any CRM ID to it if it is an entity that we don't know how to process because then you will have the form submission which is completely disconnected from the integration because we have no CRM ID and then you do a full sync and then the contact that they created from the form submission will be added to our key space, but that is completely separate from the submitted one from the form and now you have two profiles in UPS that are technically the same ones.

### Awaiting Confirmation from Tribe

[Erik Andersson] posed the question to the Tribe team 3 weeks prior but has not received confirmation:

> I put the question to the tribe team if we like, is this the case? Are you always responding with contacts? Because if so, we can completely remove everything that has with lead inside of a tribe in abscess.

### Configuration to Remove

If confirmed, the following should be removed from APSIS Tribe configuration:

```
Lead entity configuration (currently at: tribe configuration page)
Lead entity in additional entities loading
Lead entity in consent mappings configuration
```

Example configuration artifact:
```
- Lead entity dropdown selection
- Lead entity attributes
- Lead entity mapping rules
```

### Cleanup Work Required

**Essential** (1 day, high priority):
1. Remove lead entity configuration from Tribe connector UI
2. Update mappings to only handle contact entity

**Additional** (optional, audience team):
1. Hide Tribe lead attributes from customer accounts
2. Database cleanup of lead entity entries and key spaces

[Erik Andersson]: "The most important one is just like doing what I did here, like remove the lead entity configuration for Tribe and then I think 90% of the work is done."

### Why This Wasn't Done Already

> Because I want confirmation from tribe that I can actually go ahead and do this because I have previously explained to every consultant why we need to have this now.

The concern is that if Tribe *does* send other entity types in some edge cases that APSIS doesn't know how to handle, removing the lead entity configuration would cause form submissions to fail silently, creating duplicate profiles without APSIS awareness.

### Customer Support Impact

This is the single most frequent customer question about Tribe:

> This is the single most question I've got them regarding tribe like what is the lead in abscess because you don't have lead in tribe.

Explaining virtual entities to customers is not productive:

> And then trying to explain to someone like what a virtual entity is, is not a big or that's not a very fun thing to do.

---

## Debugging Methodology: Using the Internal Broker Service

### Problem Investigation Approach

When investigating the Tribe pagination issue, [Erik Andersson] used the internal broker service to manually test Tribe's API without requiring direct access to customer systems:

> We have this functionality where you can construct a request to the customer's CRM system and route it through our broker service, which will then add the the correct authorization.

### What is the Internal Broker Service?

An endpoint that:
- Accepts requests targeting a customer's CRM system
- Injects the correct authentication credentials (decrypted from database)
- Routes the request to the actual CRM API
- Returns the response to the debugger

This is critical because:

> This collection is dependent on you having direct access to the customer system, which you very seldom do because like we need to they're decrypted from the database, but as we went through the previous Katie session we have this functionality where you can construct a request.

### Practical Example: Testing Pagination

To identify the pagination bug, [Erik Andersson]:

1. Built requests targeting the customer's Tribe API
2. Sent them through the internal broker service
3. Tested sequential pages: page 0, page 1, page 2, page 3, page 4 (50 contacts per page)
4. Combined the results and cross-referenced CRM IDs
5. Identified duplicates across pages

> So I just made the request like page zero, page 12345 and then combined those results and then we could see that they were it was like duplicates galore.

### Key Takeaway

The internal broker service is an indispensable tool for debugging CRM integration issues because:

> It's quite often that we need to make like debug requests to the CRM system. Like how does it look when I make it manually and how? How does it work? How does it look like inside of APSIS? ... Things like that is it's very hard to find fast. But I mean the more, the more you work with this, like the more things you will, the more you will remember and find fast.

---

## Tribe Connector: Architecture and Relationships

### Why Tribe is Unique

Tribe operates differently from other integrated CRM systems within FSC:

> Tribe kind of likes go in their own way with things. They don't want to integrate with other systems, like for example, they utterly refused to integrate with the central customer database in Maxo.

### Isolation and Parallel Systems

- **Maxo**: Central customer database for most FSC products
- **Tribe**: Maintains its own separate customer database
- **Tribe customer service system**: Separate ticketing system (not integrated with Maxo)
- **Tribe marketing module**: In development, designed to reduce Tribe's dependency on external integrations (including APSIS)

> That's why you have like one set of customers in tribe and then you have another set of the customer inside of Maxo. The customer service system is separate for tribe because any customer cases for tribe needs to be registered in their system and not the one in Maxo and now they are working on a another marketing module for tribe so they don't have to integrate with Appsys.

### Why APSIS Must Still Support Tribe

Despite Tribe's isolation strategy, APSIS continues to support the integration because:

1. Existing customers have integrated APSIS with Tribe
2. New customers are sold Tribe and APSIS as a bundle

> But I mean we still have customers that are integrated with Appsys, so we still need to maintain this one. And also there are a lot of customers that have are being sold tribe together with Appsys.

### Stability Assessment

Tribe is the least stable connector in the APSIS portfolio:

> We will see like that that whole connector kind of needs a revamp on both sides I think because now they are getting so many more customers and tribe and it is by far the most not stable connector of them all.

The instability stems partly from architectural decisions (the virtual lead entity setup) and partly from API behavior (pagination duplicates).

---

## Prioritization and Roadmap Planning

### Priority Assessment by Lukasz Grabowski

**Q1 Scope (Closed)**

**Q2 Priority:**

1. **Large customer scalability** (high priority, larger scope)
   - Rationale: Addresses existing customer pain with 8M+ contacts
   - Includes performance optimization and CRM delegation testing
   - Should begin Q2

2. **Extended sync conditions** (medium priority, nice-to-have)
   - Rationale: Good feature but lower priority than scalability
   - Requires CRM system coordination and testing
   - Lower priority relative to customer size issue

3. **Full sync report improvements** (low priority, low effort)
   - Rationale: Has existed as-is since release
   - Quick win for front-end team during slack periods
   - Can be done with free FE capacity

4. **Tribe lead entity removal** (pending Tribe confirmation)
   - Rapid implementation (1 day) if Tribe confirms duplicates are not a concern
   - High customer impact (reduces confusion)
   - Currently blocked on Tribe response
   - High priority **if confirmation is received**

[Lukasz Grabowski]: "If we have free time for the front end developers, they can probably look at that. It's quite easy... And then more issues with drives then better... And try the the sync extend sync condition. I think it's nice to have... it's rather lower prior."

---

## Technical Debt and System Architecture Concerns

### Generic Connector Limitations

The generic connector architecture has inherent limitations that affect multiple CRM systems:

1. **Cannot handle dynamic entities**: Must know entities at configuration time, not at runtime
2. **Version fragmentation**: Now has two versions of sync condition endpoints (v1 with equals only, v2 with complex operators)
3. **Requires CRM system implementation**: Each CRM must independently implement new features

### CRM System Coordination Challenge

Adding features to the generic connector now requires coordination across multiple CRM teams:

- **Site Shop** (Microsoft Dynamics): Has implemented v2 sync conditions
- **Enterprise** (FSC Enterprise): Will need v2 and large customer support
- **Tribe**: Silos itself; slow to respond to feature requirements
- **Others**: Unknown current status

> Before this is actually enabled for any account like the V2 version of the sync conditions, this needs to be verified with site shop because I don't think they have actually implemented support for the new structure because it never took off inside of inside APSIS.

### Historical Context: Why Features Remain Unfinished

Three factors contributed to incomplete projects:

1. **Front-end resource constraints**: "Integration stores were always down prioritised in favour of e-mail and audience"
2. **Departures**: "Josh left the company" (backend developer who worked on improvements)
3. **Architecture misalignment**: Early decisions (like Tribe lead entity) require confirmation from external teams to reverse

---

## Key Takeaways

1. **Three projects were started but not finished** due to front-end resource unavailability:
   - Improved full sync report (backend done, frontend pending)
   - Extended sync conditions with AND/OR operators (backend done, frontend pending, CRM verification needed)
   - Large customer handling (architecture decided, implementation needed on multiple fronts)

2. **Large customer scalability is the highest priority** because it blocks existing customers with 2M-8M contacts. The solution requires delegating sync condition evaluation to CRM systems, plus optimizing APSIS's full sync to use streaming instead of in-memory buffering.

3. **Tribe connector has two critical issues**:
   - Pagination returns duplicates (not Apsis's fault; Tribe must fix)
   - Virtual lead entity confuses customers and creates duplicate profiles if forms submit entities Tribe doesn't handle (awaiting Tribe confirmation before removal)

4. **Sync conditions now exist in two versions** (v1 equals-only, v2 with complex logic) and require CRM system upgrades to enable on each system independently.

5. **Memory optimization in full sync causes crashes** at scale and should be removed in favor of streaming to SQS, improving both reliability and customer transparency.

6. **Internal broker service is critical debugging tool**: allows making requests to customer CRM systems without direct access by injecting authentication credentials through an internal endpoint.

7. **Tribe is the least stable connector** due to its architectural isolation (separate databases, separate ticketing, refusing FSC system integration) and implementation issues (pagination, entity handling).

8. **Generic connector architecture has limitations**: cannot dynamically select entities at runtime, which causes problems with systems like Tribe that support entity selection in their UI.

---

## Unresolved Questions and Action Items

### Blocking Decisions

- **Tribe confirmation** (3+ weeks awaited): Does Tribe always respond with entity="contact" regardless of selection, or can it send other entity types? This determines whether lead entity can be safely removed.

- **Site Shop v2 sync conditions verification**: Has Site Shop actually implemented support for the new complex condition structure, or only appeared to?

- **Enterprise participation in large customer solution**: What is their timeline for supporting delegated sync condition evaluation or stateless API filtering?

### Next Steps (mentioned in session)

- [Lukasz Grabowski]: "Let's try to write down the solution... stories because in the epic is described... Just look at the code and see together and estimate it. So this is something for tomorrow."

- [Erik Andersson]: "I've written a suggestion for it, but I I am not sure it is the best approach, but we can we can discuss it."

- Grooming meeting scheduled tomorrow: will detail approach for sync conditions feature and large customer implementation

### Knowledge Transfer Continuation

[Lukasz Grabowski]: "You will be with with us as a contractor" — Erik will continue as contractor post-departure, available for architecture decisions and questions.