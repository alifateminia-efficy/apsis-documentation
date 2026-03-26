---
source_file: New feature - Duplex integration.txt
domain: Apsis One Integrations
topics: [Duplex integration, Profile attribute synchronization, CRM bidirectional data flow, Generic connector architecture, API-based event notifications, Feature scope and customer demand validation]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Speaker 1, Henrik Boye, Agneta Lindahl Nevell]
key_components: [Generic connector, One API, Event system, Pre-filled forms, CRM integrations, Tribe, Dynamics, E-deal, Enterprise, Survey functionality]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors"]
---

## Session Overview

This knowledge transfer session discussed a proposed new feature called "Duplex integration"—enabling bidirectional data synchronization between Apsis One and external CRM systems at the profile attribute level. The core challenge debated was architectural: should this feature be built into the generic connector (limiting it to supported CRM systems), exposed via a public API (available to all customers but requiring more integration work), or via integrations-specific approaches? The team emphasized that before development begins, they need concrete customer use cases and pilot customers to validate that the effort will deliver actual business value, rather than building speculative features.

---

## Architectural Debate: API vs. Generic Connector vs. Integration-Specific Approaches

### The Core Problem Statement

[Erik Andersson]: The feature request is driven by CRM systems wanting updates on profile changes at the attribute level in Apsis. Historically, the rule has always been that all data flows from the CRM to Apsis, but now the requirement is bidirectional: changes to profile attributes should sync both directions—from CRM to Apsis and from Apsis back to CRM.

### Two Competing Implementation Strategies

**API-Based (Generic) Approach:**
- Build a public API endpoint or event mechanism that any external system (CRM or otherwise) can subscribe to for profile change notifications
- Available to all Apsis customers, not just those using integrations
- Requires external systems to have their own R&D or hire development resources to implement the integration
- Decouples the feature from any specific connector implementation

[Erik Andersson]: If you design the feature where external systems, regardless of whether it is some integration or not, can listen to events in Apsis using something like one API, then the design should be that route, and then the CRM systems would utilize that instead of us doing something tailor-built inside an integration.

**Generic Connector Approach:**
- Implement support within the generic connector plugin system
- Works for CRM systems that have a generic connector: Tribe, E-deal, Enterprise 12.1, FSC
- Does NOT support legacy systems like old Dynamics, old Enterprise, or Lime
- Still requires CRM vendors or partners to build plugin support in their systems

[Michal Rosikiewicz]: The generic connector would utilize this one API functionality to put it through.

[Erik Andersson]: Absolutely, absolutely.

### Critical Distinction: Scope Limitations

[Erik Andersson]: Only generic connector CRMs would be supported—not old Dynamics, not old Enterprise, not Lime. But for Tribe, FSC Enterprise 12.1, and E-deal, yes.

[Henrik Boye]: What Daniel wanted is that we shouldn't care which CRM wants to integrate with us. They should be able to do that, add data to Apsis, and get data from Apsis.

This highlighted a key tension: Daniel's vision was for CRM-agnostic bidirectional integration, which the API approach supports but the generic connector approach does not.

---

## Comparison to Historical Precedent: Pro Integration Behavior

The team discussed how this differs from the existing Pro integration to understand if there are lessons learned.

[Lukasz Grabowski]: Do you know how this works in Pro integration? Do you have this functionality updating attributes in CRM from Pro?

[Speaker 1]: No, it's just events. I remember we don't have it in Pro. It's CRM to Apsis, and then events are synced back—like clicks—but changes done in Pro are not pushed back to CRM. Except consents, right?

[Erik Andersson]: Yep.

[Lukasz Grabowski]: So consents are updated in CRM. That's how it works.

**Key historical note:** In Pro, only specific data types (consents, clicks, leads) are synced back to CRM. Profile attribute changes have never been pushed back, and apparently customers never requested it.

[Speaker 1]: In Pro, as we discussed a minute ago, we have a very similar setup and we never put the data back to CRM. Nobody asked for that. As far as I remember, there was never a discussion about pushing data back to CRM.

---

## Technical Implementation Details: Generic Connector Architecture

### How Generic Connectors Work

[Erik Andersson]: For Dynamics, it's a plugin. For the new version we have a collaboration with Site Shop, so they have built a generic connector plugin that adds all the required endpoints. They would enhance this for the other CRM systems within FSC. For these systems, it's built into core functionality.

[Speaker 1]: But either way, even if we provided support within the generic connector, we would still require either us or a partner to build in the support for this into the plugin for whatever system. And then that could be offered to the customers.

### What Already Exists in the Event System

[Erik Andersson]: Event tool exists—syncing. We have the rating and everything added now. Also E-deal, for example. I've added support for this if you would want.

[Lukasz Grabowski]: But this is made via events, right?

[Erik Andersson]: Yes, exactly. So in that aspect, we already can push the data. But Tribe has not implemented support for that, so that wouldn't be anything for us to do.

---

## Customer Demand Validation: The Critical Missing Piece

### The Problem: Unvalidated Assumptions

[Speaker 1]: I think we should question whether anyone is actually interested in this feature. Do we have pilot customers? If we don't have any pilot customers, that probably means nobody is interested in that feature right now.

[Henrik Boye]: I have no names of any customers that specifically want it. Should I get that before we continue this conversation?

[Speaker 1]: I think pilot customers would be essential. If at this point we don't have any pilot customers, development will probably take a while. We need to figure out the path—worst case, we have to talk to partners to develop plugins and integrations. And then in one quarter, we'll find out we don't have any pilots because nobody is actually waiting for that. That would be a huge effort and a huge waste of money.

### Risk of Wasting Resources

[Speaker 1]: We have a huge tendency right now to spin up projects, create complex requirements, complex workflows. And if we see ourselves in one quarter with a fully working feature and nobody waiting for it, we could do so many other things during that time.

[Lukasz Grabowski]: This project is not something urgent. The goal was that we asked you that we need to work together with Erik on something. That's why we are here.

[Henrik Boye]: I don't suggest that we start to build it right now. This is more like a discussion on how to do it in the correct way.

---

## Customer Use Cases: Agneta's Input from Sales/Product

### Pre-filled Forms as the Trigger

[Henrik Boye]: We're going to launch pre-filled forms, so anyone with a CRM or any integration that populates options with data should be able to send out the form to fetch more data, and that data somehow needs to go back to those CRMs or other integrations or API. I have no clue if it should be built into the connector or if it should be an API integration, but we need a way of external systems to get those profile updates.

### Specific Use Cases Articulated by Agneta Lindahl Nevell

[Agneta]: We get this question quite a lot. Data synchronization back to external systems—not just CRM, but various systems. Several scenarios include:

1. **Marketing-to-Sales Handoff**: Marketing teams work in Apsis doing marketing activities. Sales and commercial teams work in the CRM updating opportunities. When there's a change in preference or behavior, that information would be valuable to get back to the CRM. For instance, when you push your leads down the funnel or nurture from conversion to retention and back to conversion, sales needs to know about those changes.

2. **Segment Membership Signaling**: If marketing has a great segment for identifying behavior that indicates purchase intent, synchronizing that information back to CRM contacts saying "this contact is now ready for purchase—marketing has done the nurturing and we have the purchase intent—hand over to sales."

3. **Areas of Interest**: From the marketing side, you can allow your base to indicate areas of interest and feed it back to the CRM, enabling smarter decision-making for sales. Where do we have revenue? Where do we start our sales efforts? Who do we invite to events? Who do we reach out to with certain offers?

4. **Churn Indication**: Do we see churn indication in relation to certain behavior in marketing communication?

5. **Survey Data**: With the new survey functionality, customers want NPS data fed back to the CRM as well.

[Agneta]: Quite a lot of use cases for actually getting data from Apsis to a CRM or another external system.

### Identified Customer Interest: Yunan

[Agneta]: The first one that comes to mind is Yunan. Whenever we talk to them, they're asking for more APIs because they lack the functionality of getting information. But that's more in the API than a CRM integration.

### Strategic Considerations

[Agneta]: We should also check the strategy going forward using our other business units as resellers. If that's aligned with our vision and idea on how to utilize them best, then any integration that covers Tribe mainly, but also E-deal and Enterprise, would be highly valuable and a good revenue generator. More standard solutions—for instance, getting event tool data back and from survey—not very customized—if we push that back and get it on campaigns in Tribe, that would create high value. Maybe that's for the generic connector.

[Speaker 1]: We have very scarce resources now and we should be considering where we invest them to get the best return on investment. I would suggest we start with those use cases. This is a very interesting case and apparently there is some kind of interest.

---

## Path Forward: Deferred Implementation, Use Case Validation Priority

### Decision on Development Approach

[Lukasz Grabowski]: For development right now, we should just focus on some smaller fixes and some known things that should work like this. Not the bigger project like this duplex synchronization.

[Erik Andersson]: We can add support for surveys, so that would be fairly simple.

[Lukasz Grabowski]: Yeah, let's start from these kinds of things for development.

### Immediate Next Steps

[Henrik Boye]: Let me and Agneta go through potential use cases and we put them on the Confluence page and then we get back to you.

[Speaker 1]: I think we can help you writing down the use cases. We will not be able to provide the actual metric content, but we can help with that. If you need help with stating that, we can assist.

[Henrik Boye]: Let us do a general description of the use cases, then we send it to you so you can populate it with more technical things.

### Key Outcome

The team decided that before any architectural or implementation decisions are made on the duplex integration feature:

1. **Agneta and Henrik** will gather and document use cases from customers
2. **Agneta and Henrik** will identify specific pilot customers interested in this feature
3. **The team** will evaluate which customers would benefit from API-based vs. generic connector approaches
4. **Development focus** will shift to smaller, validated features in the interim (e.g., survey support in generic connector)
5. **A follow-up meeting** will be scheduled once use cases are documented (timeframe: possibly not next week due to competing work)

---

## Important Caveats and Gotchas

### What the Generic Connector Cannot Support

[Erik Andersson]: You listed a lot of different use cases there, which I don't think the generic connector can support.

This suggests that some of the sophisticated use cases Agneta described (segment membership, behavior-based notifications, churn signals) may require an API-based approach rather than a generic connector plugin.

### Pre-filled Forms Dependency

The entire duplex feature is partially motivated by the upcoming pre-filled forms feature. Any implementation must ensure that data collected via pre-filled forms can be returned to the originating system. However, pre-filled forms are still in development, so the upstream feature is not finalized.

### Confusion About What "Duplex" Means

Early in the meeting, there was confusion about the scope. The feature is being called "Duplex integration," but it's not a single connector—it's a capability (bidirectional attribute synchronization) that could be implemented via multiple technical approaches. The term itself is somewhat misleading.

---

## Historical Context: Why This Matters

### Data Flow Philosophy

Apsis One was historically designed with a "master-detail" data flow philosophy: CRM systems are the system of record for contact data, and Apsis is an orchestration/marketing tool. Profile attribute changes have always flowed CRM → Apsis, never Apsis → CRM. This reflects the separation of concerns: sales/commercial systems own contact mastery; marketing systems enhance and orchestrate on top of that data.

The duplex feature represents a philosophical shift: acknowledging that marketing activities in Apsis (surveys, form submissions, behavior-based segmentation) generate valuable data that should flow back to the CRM to inform sales decisions.

---

## Key Takeaways

1. **Two viable architectural paths exist**: Build via public API (broader reach, more customer work required) or via generic connector (limited to supported CRMs, more Apsis-side effort). Each has trade-offs that should be evaluated against actual customer demand.

2. **Customer demand must be validated before development starts**: Without pilot customers explicitly requesting bidirectional sync, the risk of building an unused feature is high. The team has seen this pattern before in Pro, where it was never requested.

3. **The generic connector alone may not be sufficient**: Some of the use cases described by Agneta (segment membership notifications, behavior signaling, churn detection) may require API-level abstraction rather than plugin-based connectors.

4. **Pre-filled forms are the trigger**: The primary driver for duplex sync is the upcoming pre-filled forms feature. Without pre-filled forms, the business case is weaker.

5. **Yunan and other large customers have expressed interest in more APIs**: Specific customer interest is noted, but their CRM integration strategy (Dynamics) would not work with a generic connector solution, pushing toward an API approach.

6. **Interim development should focus on smaller, validated features**: Rather than start duplex work immediately, the team will focus on smaller improvements like survey support in the generic connector.

7. **Use cases must be documented precisely**: Agneta will own gathering and documenting specific customer use cases, which will then inform the architectural decision.

---

## Unresolved Questions & Action Items

**Action Items:**
- [Henrik & Agneta] Gather specific customer use cases for bidirectional sync and identify pilot customers (target: unknown, possibly not next week)
- [Henrik & Agneta] Document use cases on Confluence page
- [Speaker 1 & Lukasz] Provide technical assistance in refining use case descriptions once Henrik/Agneta provide draft
- [Team] Schedule follow-up meeting once use cases are documented to make architectural decision (API vs. generic connector)
- [Team] Shift immediate development focus to survey support in generic connector as a smaller, validated win

**Unresolved Questions:**
- Will Yunan commit to being a pilot customer if an API-based solution is chosen?
- Are there other large customers besides Yunan interested in this feature?
- What is the technical feasibility of pushing complex behaviors (e.g., segment membership, churn signals) via the generic connector vs. requiring an API?
- How will pre-filled forms integration affect the requirements? (Pre-filled forms feature is still in development)