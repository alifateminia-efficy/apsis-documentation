---
source_file: New feature - Duplex integration.txt
domain: Apsis One Integrations
topics: [Bidirectional data synchronization, Profile updates from Apsis to CRM, Architecture decisions for event notifications, Generic Connector implementation, API-based solutions, Pre-filled forms integration, Customer requirements gathering]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Henrik Boye, Speaker 1, Agneta Lindahl Nevell]
key_components: [Generic Connector, One API, CRM systems (Dynamics, Tribe, Efficy Enterprise 12.1, E-deal), Event notifications, Pre-filled forms, Surveys, Segment synchronization]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.1, Tribe, E-deal (Efficy Corporate), Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This knowledge transfer session explored the proposed "Duplex integration" feature—enabling bidirectional synchronization of profile data from Apsis One back to external CRM systems and integrations. The team debated two fundamental architectural approaches: implementing this via the Generic Connector for supported CRM systems versus building a general API-based solution available to all customers. A critical decision point emerged around customer demand validation: the team recognized that without identified pilot customers and specific use cases, development could become a significant resource drain. By the end, the group agreed to postpone implementation work and instead focus on gathering documented use cases and identifying customer segments that would benefit from this feature.

---

## Architectural Approaches: Generic Connector vs. One API

### The Core Design Question

**Erik Andersson** raised a fundamental architectural concern early in the discussion. The feature request describes external systems needing to receive profile change updates from Apsis when customer attributes are modified. Historically, Apsis has enforced a unidirectional data model: all data originates from the CRM and flows into Apsis.

> "If the main feature is that external systems should be able to get updates for profile changes from Apsis, at least historically we've always had to check: is this something that is limited to integration or all customers? Because if we build this using the integration and the general connector, then of course we can do this absolutely no issue. But then you need to have an integration for you to be able to do this. But if you sit and have like a garage implementation and you want to say I want to have updates or attribute changes from Apsis pushed to me, they would not be able to utilize this."

This distinction matters significantly for market reach. A garage implementation (custom, non-integration-based) would have no access to the feature if it's built only within the integration/connector framework.

### Path 1: Generic Connector Implementation

If implemented via the **Generic Connector**, the feature would:
- Work only with CRM systems that support the Generic Connector
- Require plugin development in supported CRM platforms
- For **Microsoft Dynamics**: already uses a plugin architecture
- For **Tribe, Efficy Enterprise 12.1, E-deal**: Site Shop has built a generic connector plugin that adds required endpoints; enhancements would flow through them for these systems
- Require partners to update plugins to support the new push functionality
- Limit availability to customers using these specific CRM integrations

**[Speaker 1]**: "Even if we provided [support] within the generic connector, then there would be either us or a partner, we would require them to kind of build in the support for this into the plugin for whatever systems."

### Path 2: General API-Based Solution

If implemented as a general mechanism (likely via **One API**):
- Available to all Apsis customers regardless of whether they use an integration
- External systems would register for event notifications through the API
- CRM systems and custom implementations would use this common API rather than integration-specific endpoints
- Requires customer development effort to consume events and push data back to their systems
- More work required from customers but more universally accessible

**Erik Andersson**: "If you design the feature where external systems, regardless if it is some integration or not, can listen to events in Apsis using something in one API, then the design should be that route and then the CRM systems would utilize that instead of us doing something tailor-built inside of an integration."

---

## Current State: What Already Works

### In Apsis One + Pro Integration

The team clarified existing functionality:

- **CRM → Apsis**: Primary direction. All customer profile data flows from CRM to Apsis.
- **Apsis → CRM**: Limited today.
  - **Events synced back**: Clicks from email marketing campaigns are synced back to CRM
  - **Leads created**: New leads generated through forms are sent to CRM
  - **Consents**: Marketing preference and consent changes are synchronized back to CRM
  - **What is NOT synced**: Profile attribute changes made within Apsis (e.g., updating a customer's first name directly in Apsis) are not pushed back to CRM

**[Lukasz Grabowski]**: "Consents are updated in CRM. So this is how it works."

**[Speaker 1]**: "So right now what I see, we have two kind of technical ways of implementing that general solution going through one API or some other mechanism that is not strictly related to integrations."

### Event Tool and Survey Support

**Erik Andersson** noted that some event-driven synchronization already exists:
- Event tool can push data with ratings and engagement metrics
- Support for E-deal already exists for event data
- However, Tribe (one of the CRM platforms) has not yet implemented support for receiving this event data despite availability from Apsis

---

## The Core Feature Request: Bidirectional Profile Attribute Synchronization

### What Customers Need

**Agneta Lindahl Nevell** (representing customer perspectives) outlined multiple use cases where bidirectional sync creates business value:

#### Marketing-to-Sales Handoff
Marketing teams work in Apsis performing nurturing campaigns. Sales teams work in the CRM. When a prospect's status changes—moving from nurture phase to purchase-ready—that information should flow back to the CRM so sales teams can act:

> "When you kind of push your leads down the funnel or when you nurture from conversion to retention and then back to whatever conversion you're expecting. Any of those cases would be valuable to see in the CRM that this change has happened and that there's a new value. It could be in an attribute, it could be a tag. It could also be whether you match a segment or not."

#### Segment Membership Synchronization
If marketing identifies a high-value segment (e.g., "purchase intent detected"), that segment membership should sync back to CRM contacts so sales knows which accounts to prioritize:

> "If marketing has a great segment for identifying behaviour that indicates a purchase intent, synchronizing that information back also to contacts in the CRM saying that this contact is now ready for purchase or it's time for you to reach out."

#### Survey and NPS Data
New pre-filled forms and survey capabilities capture customer feedback in Apsis. This data—including NPS scores and responses—should be available in CRM systems:

> "Now when we have survey where you can do NPS, guess what? They want that information back as well to the CRM."

#### Marketing Preferences and Behavior Tracking
As customers update preferences or exhibit behavior changes in Apsis campaigns (opens, clicks, unsubscribes), these signals are valuable to sales for account-based marketing and churn prevention.

### Customer Examples

**Unionen** (mentioned as a major customer) repeatedly requests "more APIs, more APIs" because they lack the ability to get information flowing back. However, their primary interest appears to be API-based rather than CRM-specific integration.

---

## Critical Uncertainty: Demand Validation

### The Resource Allocation Challenge

A decisive debate emerged about whether development should proceed without confirmed pilot customers. **[Speaker 1]** challenged the team on this point:

> "I think someone those will be the pilot customers for us, right? If we don't have any pilot customers, that would probably mean that nobody is interested in that feature, right? Just from the moment right now."

Later, escalating the concern:

> "If we at this point don't have any pilot customers, development will probably take a while, right? Then we need to figure generic depending on the path. Worst case, we will have to talk to partners to do the develop the plugins and integrations. And then in 1/4 we will find out nobody, we don't have any pilots because nobody is actually waiting for that. That that will be a huge effort and huge kind of waste of money, right?"

### Comparison to Pro (Historical Precedent)

The team noted that in **Apsis Pro**, a similar setup exists with subscription forms and customer engagement tools. Yet customers never demanded profile attribute data be pushed back to CRM:

> "In Pro, as we discussed with just a minute ago, we have very similar setup and we never put the data back to CRM. Nobody asked for that. As far as I remember, there was never a discussion about pushing data back to CRM. Profile data because customers and they also have subscription forms in there. Customers were using that as a kind of ascending machine because they get events back to CRM from Pro. But they were never interested in the attributes or profile attribute subscribers."

This historical context raised questions about whether this is genuinely a market need or an internal assumption.

---

## Next Steps and Action Items

### Immediate Actions

1. **Gather Use Cases (Henrik Boye & Agneta Lindahl Nevell)**
   - Document specific use cases for bidirectional data sync
   - Identify which customer segments would benefit
   - Clarify whether the need is API-based, CRM-integration-based, or both
   - Add use cases to Confluence page

2. **Identify Potential Pilot Customers**
   - Map which customers have expressed interest
   - Include both CRM integration customers AND customers without integrations (who might prefer API-based approach)
   - Validate revenue potential

3. **Technical Team Input**
   - **[Speaker 1]** and **Erik Andersson** offered to help write up use cases and provide technical context
   - Henrik and Agneta to provide general descriptions; technical team will add detail

### Why This Approach

**[Speaker 1]** emphasized resource constraints:

> "We have very scarce resources now and we we should be at least considering where we invest them so we get the the best return on the investment."

The decision-making framework requires both technical and business information:

> "We will have to make some kind of decision not only based on the technical aspects, but also on those customer related aspects. So I would kind of expect Henrik from you to get us pilot customers, but also look into the customers with integrations and without integration because they might be interested in those other customers as well for that feature."

### What This Feature Is NOT (for now)

The team explicitly moved away from this as an immediate development priority. **[Lukasz Grabowski]** clarified:

> "For development, right now we should just focus on some smaller fixes and some known things that OK, this should work like this and not the bigger project like this duplex synchronisation."

Instead, the team identified smaller, more certain wins:

**Erik Andersson**: "We can add support for surveys, so that would be fairly simple."

---

## Pre-Filled Forms: Existing Feature Requiring Attention

While debating duplex sync, **Agneta Lindahl Nevell** emphasized the importance of improving the existing **pre-filled forms** functionality:

> "Any further development of pre-filled forms will be quite appreciated, just mentioning it."

She plans to provide a detailed list of enhancements needed. Initial mention:

> "Consent. First thing, yeah."

This suggests pre-filled forms should receive development focus while the duplex sync business case is being validated.

---

## Technical Considerations for Future Implementation

### Survey Support

**Erik Andersson** noted that basic survey support could be added relatively simply to the generic connector. This could be a good intermediate step while gathering requirements for the larger duplex feature.

### Event-Driven Architecture Observation

The discussion revealed that event-based synchronization (sending events back to CRM) already exists technically via the event tool. The gap is:

1. Not all CRM platforms have implemented receiver support (e.g., Tribe)
2. Profile attribute changes (distinct from events) lack bidirectional sync
3. The scope may be broader than just "attributes"—it includes segment membership, behavior signals, and other derived data

**Erik Andersson**: "I think like listening to you, it feels like the story or the implementation would change quite a bit from what was described on the Confluence page. So I think it would be valuable like if we have those use cases that they exact use cases that they have asked for, then we can put our heads together and see like what what of this is already supported and which use cases is that they are asking for and then how would they be solved?"

### Scope Clarification Needed

The feature request description on Confluence may need revision once use cases are documented. What sounds like a simple "push attributes" feature may actually require:

- Event/behavior streaming
- Segment membership updates
- Dynamic calculated values
- Derived insights (churn indicators, purchase intent signals)

---

## Key Takeaways

1. **Two viable architectures exist**: Generic Connector (limited to supported CRM systems, requires partner plugin work) vs. One API (universal but requires customer development). The choice should be driven by customer demand patterns, not technical preference.

2. **Current state is asymmetrical**: Data flows reliably CRM→Apsis, but only limited event data flows back (clicks, leads, consents). Profile attribute changes don't currently reverse-sync.

3. **Demand validation is critical before development**: Without identified pilot customers and validated use cases, this could become a high-effort, low-ROI project. Historical precedent (Pro) suggests customers may not actually need this.

4. **Multiple customer segments exist**:
   - CRM integration customers (would prefer integrated solution)
   - Non-integrated customers (would prefer API-based approach)
   - Reseller customers (would benefit from generic connector approach for their own resale)

5. **Pre-filled forms is a higher-priority improvement**: This existing feature has clear demand and should be enhanced before investing in duplex sync.

6. **Feature scope may be broader than stated**: Use cases suggest the need includes not just attributes but segment membership, behavior signals, and derived insights—potentially requiring architectural changes beyond simple attribute sync.

7. **Next meeting is conditional**: Follow-up discussion depends on completion of use case documentation and customer identification.

---

## Unresolved Questions and Follow-Up Items

| Item | Owner | Status |
|------|-------|--------|
| Document specific use cases for bidirectional data sync | Henrik Boye, Agneta Lindahl Nevell | Pending |
| Identify pilot customers and their CRM platforms | Henrik Boye, Agneta Lindahl Nevell | Pending |
| Determine if demand is primarily API-based or integration-based | Henrik Boye, Agneta Lindahl Nevell | Pending |
| Get detailed pre-filled forms enhancement requirements | Agneta Lindahl Nevell | Pending |
| Schedule follow-up architecture meeting | Lukasz Grabowski | To be scheduled after use cases documented (possibly not next week due to resource constraints) |
| Clarify exact data types needed (attributes, segments, events, derived signals) | Product team | Pending, depends on use cases |