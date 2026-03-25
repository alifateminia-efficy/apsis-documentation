---
source_file: New feature - Duplex integration.txt
domain: Apsis One Integrations
topics: [Duplex synchronization, Profile data bidirectional sync, CRM integration architecture, API-driven event notifications, Pre-filled forms, Integration design patterns]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Henrik Boye, Speaker 1, Agneta Lindahl Nevell]
key_components: [Apsis One, Generic Connector, One API, Microsoft Dynamics, Tribe, Efficy Enterprise, E-deal, Pre-filled forms, Survey integration, Event syncing]
session_type: architecture-review
subdomains: [Architecture, Microsoft Dynamics Integration, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration, Tribe Integration, E-deal Integration, Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This knowledge transfer session explored the architectural design of a proposed **duplex synchronization** feature that would enable bidirectional data flow between Apsis One and external CRM systems. Historically, Apsis has operated under a "single source of truth" model where all data flows CRM → Apsis, but customer demand—particularly around pre-filled forms and marketing data enrichment—has created a need to push profile updates and behavioral data back to CRMs. The team debated two implementation approaches: building the feature into the **Generic Connector** (limiting it to certain CRM platforms) versus creating a **general One API solution** (available to all customers but requiring more development work on their end). A critical gap emerged: the absence of identified pilot customers. The meeting concluded that use case validation with real customers must happen before implementation decisions are made.

---

## Architecture Decision: API-Based vs. Integration-Specific Implementation

### The Core Design Dilemma

**Erik Andersson** articulated the fundamental architectural tension: whether to build duplex synchronization as an integration-specific feature or as a platform-wide capability.

> If the main feature is that external systems should be able to get updates for profile changes from Apsis, at least historically we've always had to check whether this is something limited to integration or for all customers. If we build this using the integration and the generic connector, then of course we can do this absolutely no issue. But then you need to have an integration for you to be able to do this. But if you sit and have like a garage implementation and you want to say I want to have updates or I want to have attribute changes from Apsis pushed to me, they would not be able to utilize this.

**Michal Rosikiewicz** initially proposed narrowing the scope to integrations only, but Erik pushed back, arguing that limiting functionality to integration customers creates an artificial divide.

### Path 1: Generic Connector Approach

Under this model:
- The **Generic Connector** would be enhanced to support outbound event pushes
- Works for: **Tribe**, **Efficy Enterprise 12.0**, **Efficy Enterprise 12.1**, **E-deal**
- Does **NOT** work for: Old Dynamics, Old Enterprise, Lime (non-generic connector CRMs)
- Requires CRM partners to implement plugin support for receiving the data
- Lower technical lift for customers but creates a multi-CRM dependency for full plugin coverage

**Erik noted**: "For Dynamics, yes, it's a plug-in. For the new version we have a collaboration with Site Shop, so they have built a generic connector plug-in that adds all the required endpoints, so they would enhance this for the other CRM systems within FSC."

### Path 2: One API Approach

- Available to **all customers** regardless of CRM platform
- Customers must do their own development work to consume events
- No dependency on CRM-specific plugins
- Higher barrier to entry (requires R&D resources on customer side) but more flexible and future-proof
- Aligns with Daniel's stated vision that "we shouldn't care which CRM wants to integrate with us"

---

## Current State of Data Synchronization

### Apsis Pro vs. Apsis One Integration Model

**Speaker 1** provided historical context from Apsis Pro integrations:

> In Pro, as we discussed just a minute ago, we have very similar setup and we never put the data back to CRM. Nobody asked for that. As far as I remember, there was never a discussion about pushing data back to CRM profile data because customers have subscription forms in there. Customers were using that as a kind of ascending machine because they get events back to CRM from Pro. But they were never interested in the attributes or the profile attribute subscribers—it is always overwritten when it's pulled back from CRM to Pro.

**Current bidirectional flows in existing integrations:**
- ✓ Clicks are synced back to CRM
- ✓ Leads are created in CRM
- ✓ Consents are updated in CRM
- ✗ Profile attribute changes from Apsis are **NOT** currently pushed back
- ✓ Event tool already has syncing support with rating and E-little integration support

**Erik added**: "Event tool exists syncing. We have the rating and everything added now. Also E little for example. I've added support for this if you would want."

This means some infrastructure for event-based data push already exists but has not been fully implemented across all use cases.

---

## Business Case and Customer Demand

### Missing Pilot Customers—A Critical Risk

**Speaker 1** raised a fundamental concern about building a feature without validated customer demand:

> I think someone those will be the pilot customers for us, right? If we don't have any pilot customers, that would probably mean that nobody is interested in that feature, right? If we at this point don't have any pilot customers, development will probably take a while, right? Then we need to figure generic depending on the path. Worst case, we will have to talk to partners to do the develop the plugins and integrations. And then in 1/4 we will find out nobody, we don't have any pilots because nobody is actually waiting for that. That that will be a huge effort and huge kind of waste of money, right?

This framing shifted the meeting significantly. It became clear that **before architecture decisions are made, customer interest must be validated**.

### Agneta Lindahl Nevell's Use Case Articulation

When Agneta joined the call, she provided concrete business scenarios where duplex sync would add value:

**Marketing-to-Sales Handoff Pattern:**
> What data can be synchronised back to, for instance, a CRM? It could be that you're updating your marketing preferences. In the CRM you have sales and commercial teams working, updating data with opportunities and things like that. But the marketing team, they're working in Apsis, doing marketing activities. So whenever there's a change in preference or a change in behavior, that information would of course be valuable also to get back in the CRM, for instance, when you kind of push your leads down the funnel or when you nurture from conversion to retention and then back to whatever conversion you're expecting.

**Specific data elements for sync:**
- Attribute changes (e.g., first name, last name updates)
- Segment membership (e.g., "purchase intent segment matched")
- Behavior indicators (engagement, churn signals)
- Survey/NPS responses
- Marketing preferences and tags

**Revenue-aligned use cases:**
- Identifying which contacts show purchase intent so sales can prioritize outreach
- Segmenting based on behavioral indicators to guide event invitations and targeted offers
- Detecting churn signals in relation to marketing communication patterns

**However**: Agneta acknowledged, "I have no numbers on how many customers or example of customers that really really wants this."

### Named Customer Interest: Uniunan

**Henrik Boye** mentioned **Uniunan** (described as "one of our biggest customers") as interested in this capability, but:
- **Erik clarified**: Uniunan is on **Dynamics**, which uses the old (non-generic) connector and would **NOT** be served by the Generic Connector approach
- Their primary request is "more APIs"—suggesting the One API path might be more appropriate for them
- Agneta indicated their need is broader than CRM integration—they want API-level access to pull data back

---

## The Confluence Page vs. Reality Gap

**Erik raised an important concern** about misalignment between what's documented and what customers actually need:

> I think like listening to you, it feels like the story or the implementation would change quite a bit from what was described on the Confluence page. So I think it would be valuable like if we have those use cases that they exact use cases that they have asked for, then we can put our heads together and see like what what of this is already supported and which use cases is that they are asking for.

This suggests the initial feature spec may not accurately reflect customer demand and the team should rebuild the requirements from customer input rather than assumptions.

---

## Strategic Alignment and Reseller Model

Agneta highlighted that duplex sync decisions should align with **Apsis's broader business strategy** around resellers:

> We also need to check the strategy going forward using our other business units as resellers because if that's you know, if the vision and idea we have on how to utilize them the best way, then of course any integration that covers Tribe mainly but also E-deal and Enterprise would be you know highly valuable and a good revenue generator.

This suggests:
- **Tribe** integration would be particularly valuable for reseller partners
- **E-deal** and **Enterprise** platforms should also be considered
- The choice of Generic Connector vs. One API has downstream revenue implications for partner channels

---

## Pre-Filled Forms: Adjacent Feature with Existing Demand

While discussing duplex sync, **pre-filled forms** emerged as a feature with clearer customer traction. This feature would:
- Allow Apsis forms to be pre-populated with CRM data
- Create a natural need for form submission data to flow back to the CRM
- Work better with the Generic Connector approach (since it's CRM-specific)

**Agneta emphasized**: "Any further development of pre-filled forms will be quite appreciated, just mentioning it."

And specifically requested: **Consent handling in pre-filled forms** as a near-term need.

This suggests pre-filled forms could be a quicker win than full duplex sync if the team wants to deliver customer value in the near term.

---

## Decision Framework: What Needs to Happen Before Architecture Choices

**Speaker 1** synthesized the conversation into a clear decision framework:

1. **Identify pilot customers** with actual business need for duplex sync
2. **Document detailed use cases** for each pilot (not just "we want data back")
3. **Determine which systems/platforms** each customer uses (to decide Generic Connector vs. One API viability)
4. **Assess revenue potential** against development effort
5. **Consider reseller strategy**—which platforms deliver best ROI for partners

> We will not be able to discuss that before we get some feedback from customers interested in that because we need to know whether we go with the generic connector or we go with one API and for that we need to know which customers are interested in that?

---

## Implementation Complexity Breakdown

### Generic Connector Path Complexity

**Effort required:**
1. Implement outbound event capability in Generic Connector framework
2. Define plugin contract for receiving events (what data, in what format)
3. Coordinate with Site Shop for Dynamics implementation
4. Request implementations from other CRM partners for Tribe, E-deal, Enterprise
5. Validate that plugins from multiple vendors work correctly
6. Support multiple parallel plugin implementations

**Advantage**: Once built, customers only need to deploy plugins (less dev work on their end)

**Disadvantage**: Locks feature to Generic Connector CRMs only; delays time-to-market while waiting for partner implementations

### One API Path Complexity

**Effort required:**
1. Design event subscription/webhook model in One API
2. Implement event publishing infrastructure
3. Document API contract and authentication
4. Possibly create SDKs/libraries in common languages
5. Support customers implementing their own integrations

**Advantage**: Available immediately to all customers; doesn't depend on third-party CRM partners

**Disadvantage**: Higher friction for customers without in-house development capability; requires customers to solve integration themselves

---

## Key Concerns and Gotchas

### Resource Scarcity Warning

**Speaker 1** expressed concern about spinning up large projects without customer validation:

> We have huge tendency right now to spin up projects, create complex requirements, complex workflows. And if we see us in 1/4 or something with the fully working feature and nobody waiting for it, that we could do so many other things. We could do so many other things in that during that time.

**Implication**: The team should prioritize validating demand before committing significant engineering resources. Building for hypothetical use cases is a risk in the current resource environment.

### Confluence Documentation Mismatch

The feature page on Confluence may not reflect actual customer needs and should be rebuilt once use cases are documented. Erik's warning suggests the team has been working from an incomplete or incorrect specification.

### Uniunan Platform Constraint

Uniunan (a key customer) is on **old Dynamics** which:
- Does NOT use the Generic Connector
- Cannot be served by a Generic Connector-based solution
- Would require One API approach to benefit

This is a concrete data point that Generic Connector alone may not address key customer demand.

---

## Technical Prerequisites Already in Place

The team identified some existing infrastructure that could be leveraged:

- **Event syncing framework**: Already used for clicks, leads, consents
- **Event tool integration**: Has rating and E-little support already
- **Plugin architecture**: Generic Connector already has plugin model with Site Shop relationship
- **One API**: Already exists for profile queries (segment membership checking mentioned)

This suggests implementation might not require starting from zero—it's more about extending existing event infrastructure.

---

## Immediate Next Steps (As Agreed)

1. **Henrik and Agneta** to document detailed use cases (both customer names and specific data flow scenarios)
2. **Speaker 1** to help populate technical implementation details once business cases are described
3. **Update Confluence** with validated use cases
4. **Schedule follow-up meeting** (timeline: flexible, not urgent per this week)
5. **In the interim**: Team should focus on smaller fixes and known bugs rather than starting duplex sync development

**Henrik stated**: "I don't suggest that we start to build it right now, so this is more like a discussion on how to do it in the correct way."

---

## Pre-Filled Forms—Secondary Feature Priority

Since duplex sync requires more investigation, the team identified **survey integration support** as a quicker-win alternative for development efforts:

**Erik suggested**: "We can add support for surveys, so that would be fairly simple."

This would complement pre-filled forms without requiring the full architectural decision-making that duplex sync needs.

---

## Key Takeaways

1. **Two viable architectures exist**, each with different trade-offs:
   - **Generic Connector**: Easier for customers, harder for us (multi-vendor plugin coordination)
   - **One API**: Harder for customers, easier for us (but limits addressable market to customers with dev resources)

2. **Customer demand is unvalidated**. Without pilot customers and documented use cases, proceeding with implementation is high-risk waste of scarce engineering resources.

3. **Existing customer signals are contradictory**:
   - Agneta sees business value and multiple use cases
   - But cannot name specific customers asking for it
   - Uniunan wants APIs but isn't using Generic Connector anyway
   - Pro customers never asked for this feature historically

4. **The feature spec (Confluence page) is likely wrong** and should be rebuilt from actual customer conversations rather than assumptions.

5. **Resource prioritization matters**: With current team capacity, choosing to build duplex sync (complex, unvalidated) means **not** building other things (like survey integration or pre-filled form enhancements) that may deliver faster ROI.

6. **Reseller strategy is relevant**: The choice between Generic Connector (Tribe-focused) vs. One API (platform-wide) has implications for partner revenue models.

7. **Pre-filled forms are adjacent and more validated**: This feature has clearer customer interest and might be a better near-term focus.

---

## Unresolved Questions and Action Items

### Action Items (Assigned)

| Owner | Task | Timeline |
|-------|------|----------|
| Henrik Boye & Agneta Lindahl Nevell | Document detailed use cases with specific customers and data flows | ~1-2 weeks (not urgent) |
| Speaker 1 (Optional) | Help populate technical implementation details once business cases are provided | Post-use case document |
| Lukasz Grabowski | Schedule follow-up meeting once use cases are documented | TBD |

### Unresolved Questions

1. **Which specific customers are interested in duplex sync, and which CRM platforms are they on?** (Critical for architecture choice)

2. **How much of the requested functionality already exists via One API profile queries vs. requiring new event push capability?** (Could reduce scope)

3. **What is the reseller/partner revenue impact of Generic Connector approach vs. One API approach?** (Strategic input needed)

4. **Does pre-filled forms alone solve enough customer need that duplex sync can wait?** (Prioritization question)

5. **What exactly is on the Confluence page, and how far off is it from actual customer requirements?** (Specification validation)

6. **For customers without in-house dev (who need Generic Connector), are they really willing to wait for multi-vendor plugin coordination?** (Viability of either approach)

---

## Warning for Downstream Team Members

If you're joining this effort:
- **Do not** start implementing based on the Confluence specification yet
- **Do** expect the requirements to change significantly once customer conversations happen
- **Be ready** to argue for scope constraints given resource scarcity
- **Consider** whether pre-filled forms or survey integration might deliver more immediate customer value
- **Remember** that historically, Apsis customers (in Pro) never asked for CRM data push-back, so validate whether this is truly needed or aspirational