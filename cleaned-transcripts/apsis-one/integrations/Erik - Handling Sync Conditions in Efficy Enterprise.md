---
source_file: Erik - Handling Sync Conditions in Efficy Enterprise.txt
domain: Apsis One - Integrations
topics: [Sync Conditions, Full Sync Performance, Enterprise CRM Integration, Data Volume Optimization, API Contract Design, Webhook Filtering, Consent Syncing]
speakers: [Erik Andersson, Raziel Carvajal, Lukasz Grabowski]
key_components: [Efficy Enterprise CRM, Sync Conditions Engine, Full Sync Process, Real-time Webhooks, Consent Records, Get Records Endpoint, Query Parameters, Pagination]
session_type: architecture-review
---

## Session Overview

This session addresses a critical performance problem in the Apsis One integration with Efficy Enterprise CRM: the current sync conditions architecture requires downloading and filtering massive volumes of contact and consent data on the APSIS side, causing full syncs to fail or take 1.5+ days when customers have millions of contacts in their CRM. Erik Andersson presents the problem scope (8 million contacts across 22 installations for a French customer) and proposes moving sync condition filtering upstream to the CRM system itself, similar to an existing solution implemented with Dynamics 365. The discussion centers on whether to implement this as stateless (query parameters per request) or stateful (configuration-based), with implications for API contract design and pagination.

---

## Current Sync Conditions Architecture and Its Limitations

### How Sync Conditions Work Today

**Sync conditions** are rules stored in APSIS that determine which contacts from the CRM get synced into APSIS. They are evaluated on every contact during both full syncs and real-time syncs.

[Erik Andersson]:
> So let's say now that I only want to sync everyone that originates or that has a place in Sweden because I want to do some very tailored send out specifically to Sweden and some marketing campaigns. So you store these conditions and these are stored inside of APSIS and what is happening is that anytime we receive a contact, be it from a full sync or be it from a real time sync, these conditions are evaluated so we compare the data in on the contact and see like does the birthplace equal Sweden? If yes, we will sync this one to APSIS. If not, we will simply discard this contact.

The system was originally designed this way because APSIS integrates with CRM systems over which it has no control, so filtering logic had to be centralized in APSIS itself.

### The Problem: Massive Data Transfer for Minimal Results

This design worked adequately until customers started having 8+ million contacts in their CRM systems—far exceeding initial assumptions.

**Concrete example** [Erik Andersson]:
> Say now that the customer in question, they want to install 22 installations in APSIS and handle different sections, most likely for like organization or group or region. They now then have 8 million contacts in their CRM system. And now when we do a full sync of these, we need to download all of eight millions of these contacts just to be able to evaluate these sync conditions in APSIS. So like ultimately they should sync 100,000 contacts, but of course we will still download the rest of the 7,900,000 and then discard them.

**Consent syncing compounds the problem**:
> On top of this we have the consents for which is the same thing and typically like you have at least one consent for each contact, so that would be approximate 416 million things which we will download from the CRM system for one sync. Now we need to multiply this by 22 if you want to have multiple sections.

### Real-World Impact

[Erik Andersson]:
> I did check on the initial sync for the customer. So we did actually manage to download all of the contacts without the sync failing. When we came to the consent, however, then all hell broke loose because we have an optimization inside of Appsys which will render this useless like it will crash the full sync because memory issues, but even if we remove that optimization, the FSC Enterprise instance in question kind of timed out for 1 1/2 day while this sync was ongoing before it terminated because of a too long time and we saw a gateway time out.

This is not a hypothetical issue—there are already multiple customers affected.

[Raziel Carvajal]:
> Yeah, it's been an issue. I mean even with 1,000,000 even it's been an issue because all this time usually for having access to the information in the CRM, well sometimes the disconnection takes place. Then clients report that the CRM is slow and then when you check the logs and check, OK, there is a sync condition as a sync conversation taking place. OK, so then this is the reason.

---

## Precedent: Dynamics 365 Solution

APSIS has already solved this problem with a different CRM system by moving sync condition evaluation upstream.

[Erik Andersson]:
> What we have actually done already for another CRM, namely this dynamics by site shop, is that while you still configure the sync conditions inside of Appsis here to like keep everything in one place, we are sending the sync conditions over to them. They store these conditions and in some some way they either have a cache ready or generate this on the full sync. One way or another, anytime we ask for a full sync, they will only present us with the contacts that are matching the sync condition.

**Results with this approach**:
> So in the case of this customer now we would only download these 100,000 and the sync would be done in like 5-10 minutes pops. That's like a piece of cake.

[Erik Andersson]:
> Same thing for sync conditions for the web hooks. They went all in and also only sent us web hooks for profiles that are matching the sync conditions.

### Why This Works

By filtering at the source (in the CRM), APSIS avoids:
- Memory overload from processing massive datasets
- 1.5+ day sync durations
- Network bandwidth waste
- Unnecessary database strain on both systems

The trade-off is that the CRM must perform the filtering, which requires either:
1. Pre-caching filtered results
2. Generating filters dynamically when requested
3. Storing configuration and applying it at query time

---

## Proposed Approach for Efficy Enterprise

Two implementation strategies were discussed, with the team gravitating toward the **stateless query parameter approach** as the first step.

### Option 1: Configuration-Based (Stateful) Approach

**How it would work**:
- APSIS sends sync conditions to Enterprise as a configuration setting
- Enterprise stores these conditions
- When APSIS requests a full sync, Enterprise applies the stored conditions and returns only matching contacts

**Advantages**:
- Fewer total requests from APSIS to Enterprise
- Enterprise can optimize caching and query planning

**Disadvantages**:
- Requires Enterprise to maintain state
- More complex for Enterprise API design (goes against stateless API principles)

[Raziel Carvajal]:
> For a moment we have treated all these square, all these the request of this API is stateless. We don't keep any state in the. As soon as we receive the request, we just attend the request and then we we we return the answer without keeping any cash because we might have a concurrency issues later or inconsistencies. So I I really like the idea of having this cache, nevertheless, there I don't really like for a for right now... having to keep state.

### Option 2: Query Parameter Approach (Stateless)

**How it would work**:
- APSIS includes sync conditions as query parameters in each GET request
- Enterprise evaluates conditions on each request without storing state
- APSIS handles pagination as usual (page 0, 1, 2, etc.)

**Advantages**:
- Truly stateless—each request is independent
- Simpler API design philosophy
- No risk of state inconsistencies or concurrency issues
- Aligns with REST principles

**Disadvantages**:
- Requires APSIS to send the conditions with every paginated request
- Enterprise must re-evaluate conditions on each request
- Potentially longer URLs if many conditions exist

[Raziel Carvajal]:
> If the page, if the results are bigger, but if you if Appsys receives a number of results in the request lower than the size of the page, then it's over. So then Appsys shouldn't send another request if we are thinking about sending this consecutive request.

[Erik Andersson]:
> We we will send one more request and that will be empty and then that's when we stop. But that should reduce the amount of requests for you, right? Because like right that you you would still need to process over the same data sets... so but I mean then I think that would be a viable approach at least to investigate because I think that should be fairly simple like you would just need to agree on because we don't send this in the request today. That is of course something we can add.

### Consensus Direction

The team agreed to move forward with the stateless query parameter approach as the initial implementation.

[Raziel Carvajal]:
> Personally speaking, I see this for, you know, to solve the general problem, to give a general solution. I think it makes sense to keep this.

[Erik Andersson]:
> Yeah, no, I mean, I'm I'm of course fine with that. No, no issue.

---

## API Contract Specifications

### Get Records Endpoint Enhancement

**Current behavior**:
- APSIS requests records with pagination (page number, page size)
- Enterprise returns all records matching pagination
- APSIS can optionally specify fields to reduce data volume

**Proposed enhancement**:
- Add sync condition filtering to the GET records request as query parameters
- Enterprise applies conditions and returns only matching records
- Pagination still applies to filtered results

[Erik Andersson]:
> What we would need to introduce here is like a list of sync conditions. We would provide in the like get records request: first name equals this, country equals this and these fields would be the FSC enterprise attributes.

### Sync Conditions Format

**Current limitation**:
> We don't support that today. Everything here is AND. That's a weakness in the sync condition.

[Erik Andersson]:
> We did implement a new version of sync conditions where we do support exactly what you said [OR conditions], but this never came to life because every front end resource disappeared on us and then the reorg happened. So this feature isn't in place for any customer. So right now everything here is AND so it's like birthplace and the address and this and this.

**For the initial implementation**, all conditions will use AND logic:
- birthplace = Sweden AND address = Stockholm
- country = France AND region = Île-de-France

A future enhancement could add support for OR conditions if needed.

### Query Parameter Details

[Erik Andersson]:
> We could still include the page size and the page and if anything is empty then we will stop the syncs like we will always send one more request than we than we technically need to.

**Estimated URL length not a concern**:
[Erik Andersson]:
> I don't know I think we have a limit of 10 sync condition if memory serves so like and typically customer only have like two or three so I I don't think that's gonna be an issue.

**Next step**: APSIS will provide the API specification with:
- Query parameter names
- Format (e.g., `condition[0][field]=firstName&condition[0][operator]=equals&condition[0][value]=Erik`)
- Example requests
- Enterprise reviews and implements

[Raziel Carvajal]:
> Yeah, but this is the but this is frankly Eric and Lucas. I mean you guys because it's usually how we we when you have an API, right, you update the API, you give the number whatever you want, the format whatever you want and you just give an example and we have questions we ask but I don't see why we should really give a stand in the name or just just let us know when the API is written and then we take a look and then we implement.

---

## Consent Syncing Strategy

### The Consent Problem

Consents are separate entities from contacts but tied to them. During a full sync, APSIS currently downloads all consents for all contacts, regardless of whether those contacts match sync conditions.

[Erik Andersson]:
> I have 8 million profiles or contacts in my CRM system and at least one of them have consent and say that they have two consent each. That is still 16,000,000 consent entries that I will today download in the full sync. Because in the full sync we download all contacts and we download all of the consents and that's going to be a problem.

### Consent Filtering via Get Consents Endpoint

The solution mirrors the records approach: add sync condition filtering to the GET consents endpoint.

[Erik Andersson]:
> We will introduce a query parameter where we specify these are the same conditions for the section. We will have the exact same query parameters when we retrieve the consent and then enterprise will just give us the consent for the contact that matches... that matches this data.

**Current limitation with consents**:
> We don't provide the fields there and you can't specify unfortunately consent for any specific contact, but we will do the exact same thing here in the same thing we will introduce.

**Result of applying this to the 8M contact scenario**:
> And then you instead of downloading the 8 million * 22 profiles, we will just do it 100 * 100,000 * 22, which is still a lot, but it's separate, very small syncs instead.

### Webhook Filtering for Consents

[Raziel Carvajal] argued that webhook-based consent sync is less problematic because webhooks only send changed consents in real-time, not all consents:
> The thing is that the consent... the webhook at least in the CRM it triggers when there is a change in the in the consent or a change in the contact and this usually the payload is is is quite small because the number of changes that happen... Let's say after after a certain period of time is very narrow.

**However**, for full syncs, consent filtering via query parameters is essential:
> But if we keep it as you mentioned the other example the other connector did then you keep it in a configuration then it's fine... Naturally we might do the same for consents, no define the the same conditions because this is otherwise it's going to be also confusing for the client that the final users are.

---

## Fields and Attributes Specification

### Available Enterprise Fields for Filtering

[Erik Andersson]:
> We would provide like the enterprise field name and what value it what value it should be.

The sync conditions use **Efficy Enterprise field names**, not APSIS field names. This allows customers to leverage the full power of their CRM data.

[Erik Andersson explains the advantage of letting Enterprise handle filtering]:
> You have access to more data than we have, you can utilize like only sync profiles or contacts where the company of the contact has a revenue bigger than 100,000 or where they only or where they exist at least one time in one of the apps is consent lists or whatever you do, you can do more powerful checks there.

However, the initial approach keeps it simple:
> But I think just for the sake of simplicity, we we already have a way of doing this on Axis side. Like for us this would be like you flick a switch and then we will send you the sync conditions to you.

### Field-Level Optimization

APSIS already has a fields parameter to reduce data volume. This will continue to work alongside sync condition filtering.

[Erik Andersson]:
> We also can provide the fields where we say only give me the data for these fields and that we do to reduce the amount of data.

---

## Unresolved Design Discussions

### Pagination Semantics with Stateless Filtering

A key question remains about how pagination interacts with filtered results in a stateless system.

[Raziel Carvajal raised the concern]:
> If you have 200 contacts in the database and you divide 200 over the number of pages. Then you have the number of pages. Then you need to send this exactly number of requests with the synchronization condition as a parameter, right?

**The resolution**:
- Enterprise does not need to know the total number of filtered results in advance
- APSIS keeps requesting pages (0, 1, 2, ...) with conditions and page size
- When Enterprise returns fewer results than the page size, APSIS knows it has reached the end
- APSIS will send one extra empty request before stopping (harmless overhead)

[Erik Andersson]:
> We will always send one more request than we than we technically need to.

### Potential Caching Alternative for Future Consideration

[Erik Andersson] mentioned a "preflight" optimization used in a high-traffic scenario:
> A like extreme high traffic implementation for Maxo where we expected like 5 million contacts, contact IDs sent to us in like 6 seconds essentially. So then in that case we sent a request to the CRM saying hello we are about to do this sync. Please prepare your data. Then the CRM like they did their SQL queries, they had a cache ready for us.

This was presented as a possible future enhancement if the stateless approach proves too slow, but not a current requirement.

---

## Implementation Urgency and Priority

### Current Customer Impact

This is **actively affecting multiple customers right now**.

[Erik Andersson]:
> It is urgent even... We we we have 3 customers that are suffering from this. That's why I have already hinted this to the product in Apsis. I have hinted it to you. I am hinting hinting it here.

### Practical Context

[Raziel Carvajal] noted that from an operational perspective, the impact is somewhat mitigated if syncs run overnight:
> If they do the synchronisation at night, let's say and it took the whole night, then in the morning they want to have the synchronised data. So in that point of view is not rather urgent because the system works. I mean all the even if it is a big database you receive the results... So if I go to that side, I don't think it's rather urgent for it. It is urgent if they if somehow the clients do that in the middle of the day when they are working and then everything slow down.

However, Erik emphasized the importance of addressing this before he leaves the organization:
> I would highly recommend to do this sooner rather than later, and especially if customers CRMS are already affected like today.

### Next Steps

[Lukasz Grabowski] committed to:
1. Writing an Epic to formalize the requirements
2. Reviewing the recording to ensure accuracy
3. Circulating it for review by Erik
4. Prioritizing implementation if possible before Erik's departure

[Lukasz Grabowski]:
> Eric, let me write an epic and try to describe it as a, you know, exercise for me. Understanding this I have recording so I will try to you know rephrase it and try to information if I miss something and then I will give you to review. OK because you know I could ask you to just if you have time so please prepare this... I I yeah, my time is limited, but I must start doing this because it's important for me to understand more and more this this world and then we'll prioritize it and maybe we'll try to implement this before you. So you're leaving. Not sure if this is possible.

---

## Historical Context: Why Conditions Are Currently in APSIS

[Erik Andersson explained the original architectural decision]:
> This is how it was designed from the start because we were integrating with CRM systems over which we had no control over, so we could not add any logic anywhere else but inside of APSIS. And I mean in all honesty like this has worked quite good up until quite recently.

The design was pragmatic at the time—when customers had smaller contact databases, the overhead was manageable. The Dynamics 365 solution represents a shift in strategy, recognizing that large-scale customers require source-side filtering.

---

## Key Takeaways

1. **Current Problem**: Sync conditions are evaluated in APSIS after downloading all contacts/consents from Enterprise, causing massive data transfer (8M+ contacts, 16M+ consents) that fails or takes 1.5+ days.

2. **Root Cause**: The architecture was designed for small-to-medium CRM databases but doesn't scale to modern enterprise customer sizes (8M+ contacts with multiple installations).

3. **Proven Solution Exists**: The Dynamics 365 connector already filters at the source, reducing a 22-installation, 8M-contact sync from days to minutes (5-10 minutes for 100K contacts).

4. **Proposed Implementation**: Add sync condition filtering to GET Records and GET Consents endpoints via stateless query parameters. Enterprise receives the conditions with each request and returns only matching data.

5. **API Contract TBD**: APSIS will provide detailed API specifications (parameter names, formats, examples) for Enterprise to review and implement. The format should support AND logic with at least 10 conditions.

6. **Pagination Works Stateless**: APSIS sends pages (0, 1, 2, ...) with conditions and page size. When results < page size, APSIS knows pagination is complete.

7. **Consent Strategy**: Apply the same condition filtering to the GET Consents endpoint to avoid downloading all 16M+ consents. Webhooks are less critical since they only send changed consents.

8. **Urgency**: 3+ customers are currently affected. Implementation should occur before Erik leaves the organization. While nightly syncs are tolerable, mid-day syncs impact CRM performance.

9. **Impact Scope**: This affects Efficy Enterprise integration specifically. The Dynamics 365 connector has already solved this, and lessons should be applied here.

10. **Future Enhancement**: Once stateless query parameters are working, consider OR logic support in sync conditions (currently only AND is supported; new logic was designed but never shipped).

---

## Unresolved Questions and Action Items

### For APSIS Team (Erik Andersson):
- [ ] Produce detailed API specification for GET Records with sync condition query parameters
- [ ] Produce detailed API specification for GET Consents with sync condition query parameters
- [ ] Specify parameter names, formats, and examples
- [ ] Document pagination expectations (empty response = end of results)

### For Enterprise Team (Lukasz Grabowski / Raziel Carvajal):
- [ ] Write Epic to formalize requirements (Lukasz to lead; Erik to review)
- [ ] Review APSIS API specifications when provided
- [ ] Implement sync condition filtering in GET Records endpoint
- [ ] Implement sync condition filtering in GET Consents endpoint
- [ ] Test with large datasets (ideally with one of the 3 affected customers)
- [ ] Confirm no URL length issues with typical 2-3 sync conditions (limit of 10)

### TBD / Future Consideration:
- [ ] Prioritize: Implement before or after Erik's departure?
- [ ] Consider whether preflight/caching optimization needed if stateless approach shows performance issues
- [ ] Plan for supporting OR logic in sync conditions once current AND-only implementation is stable