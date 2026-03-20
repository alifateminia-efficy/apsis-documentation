---
source_file: New feature - Duplex integration.txt
domain: Apsis One - Integrations
topics: [duplex synchronization, bidirectional data sync, profile updates, CRM integrations, generic connector, event-driven architecture, pre-filled forms, API strategy, integration architecture]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Henrik Boye, Speaker 1 (unnamed product/architecture lead), Agneta Lindahl Nevell]
key_components: [Generic Connector, One API, CRM Systems (Dynamics, Tribe, E-deal, Enterprise), Event Tool, Survey functionality, Pre-filled Forms]
session_type: architecture-review
---

## Session Overview

This knowledge transfer session explored a proposed feature called "Duplex Integration" that would enable **bidirectional synchronization of profile data between Apsis One and external CRM systems**. The core tension discussed was whether to implement this as a general platform capability (via One API) or as a feature within the Generic Connector (affecting only CRM integration customers). The team identified significant gaps in customer validation and use case documentation before moving forward, ultimately deciding to gather concrete customer requirements before proceeding with architectural decisions or development work.

---

## Background: Current Data Flow and Limitations

### Historical Data Direction

[Erik Andersson]: The traditional pattern in Apsis has been unidirectional: **all data should come from the CRM system into Apsis**. External systems consume event data from Apsis (clicks, conversion events), but changes made to profiles within Apsis were not pushed back to external systems.

[Lukasz Grabowski]: In the current Apsis Pro integration with CRMs, data flows from CRM to Apsis and then events are synced back (clicks, events), but profile attribute changes made in Apsis do not sync back to the CRM. The exception is **consents—these are updated in the CRM** when they change in Apsis.

### The Trigger for Change

[Erik Andersson]: CRM systems are now requesting the ability to receive **profile changes at the attribute level** from Apsis. This represents a fundamental shift from the historical architecture where CRMs were the system of record for profile data.

---

## Two Architectural Approaches Under Discussion

### Approach 1: General API Solution

**Characteristics:**
- Implement a generic event notification API or mechanism available to **all Apsis customers**, not just those using integrations
- External systems (CRM or other) would register to receive events via One API
- Provides universal availability but requires **customer development effort** to implement listening and data transformation

[Erik Andersson]: If external systems regardless of integration status can listen to events in Apsis using a single API, then the design should support that route. The Generic Connector would consume this same API rather than having custom logic built inside the integration.

**Advantage:** Enables non-integration customers to access this functionality.

**Disadvantage:** Customers must build their own integrations to consume the API and push data back to their systems.

### Approach 2: Generic Connector Implementation

**Characteristics:**
- Build the functionality directly into the **Generic Connector** 
- Requires CRM-side plugin support (already exists in some form)
- Works only for CRMs that support the generic connector

[Erik Andersson]: For Dynamics, this requires a plugin. For newer CRM versions, there is collaboration with SiteShop—they built a generic connector plugin that adds required endpoints. For other CRM systems within the generic connector ecosystem, similar plugin implementations would be needed. Some systems (like Tribe, E-deal, Enterprise 12.1) have this built into core functionality.

**Limitation:** [Erik Andersson] This approach does not work for:
- Old Dynamics
- Old Enterprise versions
- Lime (which uses proprietary integration)

[Speaker 1]: The effort here is split: Apsis must develop the feature and implement it in the generic connector, then partners or Apsis must update plugins for each CRM system to support the new endpoints.

**Advantage:** Easier for end customers—less development work required on their side.

**Disadvantage:** Limited to CRM systems that support the generic connector; higher effort on Apsis and partner side.

---

## The Customer Validation Gap: A Critical Issue

### Initial Lack of Concrete Customer Evidence

[Speaker 1]: **This is the fundamental blocker.** Before proceeding with architecture decisions, we need to answer: Do customers actually want this feature? Are there pilot customers ready to validate it?

The meeting revealed a significant gap: **no specific customer names were provided** as validated users of this functionality, only a vague attribution to "Daniel" and a request that came through.

[Henrik Boye]: "I have no names of any customers that specifically want it. Should I get that before we continue this conversation?"

[Speaker 1]: If there are no pilot customers at this stage, that is a warning sign. Development effort could be substantial, and if no one is waiting for the feature by Q1, it represents wasted resources.

### Historical Precedent: Lack of Demand in Pro

[Speaker 1]: In Apsis Pro, a similar use case exists—customers have subscription forms and CRM integrations. Despite this setup, **customers never requested the ability to push profile attribute changes back to their CRM**. Customers were interested in receiving events (for segmentation triggers) but not in bidirectional attribute synchronization.

The only data that flows back is event-based (clicks, conversions) and consent updates.

> "Customers were using that as a kind of segmentation machine because they get events back to CRM from Pro. But they were never interested in the attributes or the profile attribute updates."

### Agneta's Validation: Real Use Cases Emerge

[Agneta Lindahl Nevell]: When brought into the conversation, Agneta provided substantive validation that the feature is wanted, though still without specific customer names upfront.

**Use Cases Identified:**

1. **Marketing-to-Sales Handoff:** Marketing teams work in Apsis and update customer preferences and behavior signals. Sales teams work in the CRM with opportunities and customer data. When a customer reaches a certain nurture stage or shows purchase intent, that signal (attribute, tag, or segment membership) should push back to the CRM so sales can act.

2. **Segment Membership Sync:** If marketing identifies a behavior segment indicating purchase intent, the CRM needs to know that a contact now matches this segment so sales can prioritize outreach.

3. **NPS and Survey Results:** Survey responses collected in Apsis should synchronize back to the CRM (mentioned as a future need).

4. **Preference Updates:** Marketing preference changes in Apsis should reflect in the CRM so sales understands communication preferences.

5. **Churn Signals:** Marketing behavior data (e.g., declining email engagement) should be available to sales to inform retention strategies.

[Agneta Lindahl Nevell]: "What data can be synchronised back to for instance a CRM is just one example of external systems where customers would expect data to be kind of flooding back."

### Specific Customer Interest: Uniunan

[Henrik Boye]: Uniunan, "one of our biggest customers," has expressed interest in this feature. However, [Erik Andersson] noted that Uniunan uses **Dynamics**, which would **not be supported** by the Generic Connector approach.

This creates a constraint: if a major customer needs this and doesn't use a supported CRM system, the API approach becomes more attractive.

---

## Existing Capabilities and Overlaps

### Event Tool Already Exists

[Erik Andersson]: Event synchronization to CRMs already works. The Event Tool syncs rating data and other event information. Tribe has the capability to receive this but hasn't fully implemented support for it.

### Pre-filled Forms: A Related Feature

[Agneta Lindahl Nevell]: Pre-filled forms are being launched, which allow marketing data to populate form fields. However, data submitted through these forms needs a path back to the CRM or other external systems. This was a catalyst for the duplex synchronization request.

**Current limitation:** The pre-filled forms can send new lead data but not update existing profiles.

### API Segment Checks Already Available

[Speaker 1]: For some use cases Agneta mentioned (like checking if a profile belongs to a segment), **this capability already exists in One API**. Customers can call the API to check segment membership. The question is whether Apsis should proactively push this data or require customers to query it.

---

## Technical Considerations for Implementation

### Plugin Requirements

[Erik Andersson]: Regardless of which approach is chosen, **CRM systems will need to be modified** to accept and process the new data. There are no free options:

- **API Approach:** Customers must build integration code to consume events and push data to their CRM.
- **Generic Connector Approach:** Plugins must be updated by Apsis or partners (SiteShop for some systems) to add endpoints for receiving the new data.

### Scope Beyond Simple Attributes

[Erik Andersson]: The use cases described by Agneta suggest this is not just "push attributes." The requirement includes:
- Pushing behavior patterns
- Pushing fulfillment status
- Pushing segment membership

This may require more sophisticated event schemas than simple attribute updates.

---

## Strategic Considerations

### Reseller Channel and System Coverage

[Agneta Lindahl Nevell]: The company strategy includes using other business units as resellers. If duplex synchronization is a strategic differentiator, it should cover the systems that resellers support—**primarily Tribe, but also E-deal and Enterprise**.

A generic connector solution would be "highly valuable and a good revenue generator" for these standard systems, she noted.

### Resource Scarcity

[Speaker 1]: The company is experiencing constrained development resources. Every project must justify its ROI. Without validated customer demand and clear path to revenue, this feature competes against other initiatives. The risk: ship a sophisticated feature in Q1 and discover no customers are waiting for it.

---

## Decision: Defer Implementation Pending Validation

### Immediate Action Items

[Henrik Boye] and [Agneta Lindahl Nevell] agreed to:

1. **Document concrete use cases** with specific customer scenarios (to be added to Confluence)
2. **Identify customers** interested in the feature, including:
   - CRM integration customers (for Generic Connector path)
   - Customers without integrations (for API path)
   - Potential revenue impact and strategic fit
3. **Identify the CRM systems** those customers use to inform architecture decisions

[Speaker 1]: This information is necessary to decide between Generic Connector and API approaches. Different customer bases and CRM systems point toward different solutions.

### Scope for Immediate Development

[Lukasz Grabowski]: Rather than pursuing the large duplex synchronization project, the team should focus on **smaller, known fixes** and well-defined improvements.

[Erik Andersson]: **Survey support** was identified as a smaller, immediately achievable feature that would deliver value without the architectural complexity of duplex sync.

### Why Pre-filled Forms Matter

[Agneta Lindahl Nevell]: Further development of **pre-filled forms is appreciated**. The team asked for a list of additional capabilities needed. Agneta identified **consent handling** as a priority (though [Erik Andersson] noted he would need to step back from that conversation).

---

## Open Questions and Unresolved Tensions

### API vs. Integration: A Philosophical Difference

[Speaker 1] and [Erik Andersson] held a key tension: Should Apsis build general platform capabilities (APIs) that require customer effort to integrate, or should it build connector-specific implementations that are easier for customers but limited in scope?

**Erik's position:** Scope should not be arbitrarily limited to integration customers. If external systems want to listen to Apsis events, design for that universally.

**The pragmatic counter:** Different customer populations have different capabilities. Some can build integrations; others cannot. The decision depends on which customers we're targeting.

**Resolution:** Cannot be determined without knowing which customers want this and with what CRM systems.

### How Much Is Actually Solved by Existing Features?

Agneta listed use cases, but some may already be partially addressed by existing capabilities:
- Segment membership can be checked via API
- Event data already syncs to some CRMs
- Survey data may be partially available

Before building duplex sync, the team should audit what customers can already do with existing features.

---

## Technical Debt and Historical Context

### Why Consents Are Special

[Lukasz Grabowski]: Consents are the one data type that currently syncs bidirectionally. This is a pragmatic exception: consent changes have legal and compliance implications and must update in all systems. No other attribute type has this characteristic.

### Event-Driven Architecture Already Exists

The event system (for clicks, conversions, etc.) is already in place. Agneta's use cases could potentially be served by extending this event model rather than building entirely new "update" mechanisms.

[Lukasz Grabowski]: "This is made via events, right?"

---

## Key Takeaways

1. **No Architecture Decision Should Be Made Without Customer Validation.** The meeting revealed a pattern of proposing features without concrete evidence of customer demand. This must change.

2. **The Feature Has Merit But Remains Unproven.** Agneta's articulation of use cases is compelling, but without specific customers and CRM systems identified, the business case is incomplete.

3. **Two Viable Paths Exist; Choice Depends on Customer Geography:**
   - **API path:** Serves all customers but requires their development effort. Best if we have non-CRM or non-standard-CRM customers requesting this.
   - **Generic Connector path:** Easier for CRM customers but limited to certain CRM systems. Best if demand is concentrated among Tribe, E-deal, Enterprise users.

4. **Historical Precedent Is a Warning.** Pro had similar capabilities and customers never demanded bidirectional sync. The burden is on this feature to demonstrate it's different.

5. **Pre-filled Forms Are a Prerequisite Dependency.** Duplex sync enables pre-filled forms to update CRMs, but pre-filled forms themselves need further development first.

6. **Smaller Wins Exist.** Survey support and consent handling improvements are achievable in shorter timeframes and should be prioritized while the duplex sync business case is validated.

7. **Partner/Plugin Effort Is Non-Trivial.** Whichever approach is chosen requires CRM vendors or SiteShop to update plugins. This should be factored into timeline and resource planning.

8. **Scope Inflation Risk Is Real.** Agneta's use cases span attributes, segments, behaviors, and events. The implementation must clarify which categories are in scope and design accordingly.

---

## Unresolved Questions

1. **Which specific customers want this, and what are their CRM systems?** (Henrik Boye and Agneta Lindahl Nevell to provide)

2. **What additional pre-filled forms capabilities are needed, and in what priority order?** (Agneta to gather from stakeholders)

3. **Can existing event mechanisms or API capabilities solve some of the stated use cases without new duplex sync infrastructure?**

4. **What is the timeline for use case documentation, and when should the team reconvene to make architecture decisions?** (Tentatively after Henrik and Agneta complete use case documentation; not expected within the following week)

5. **Should survey functionality be prioritized first as a smaller, immediately deliverable feature?** (Implied yes, but not explicitly confirmed)

6. **How does this feature align with the stated strategy of positioning Apsis as a marketing and customer communication platform?** (Agneta mentioned alignment; more clarity needed on specific strategic bets)

---

## Next Steps (Action Items)

- **[Henrik Boye & Agneta Lindahl Nevell]:** Document use cases in Confluence with specific customer examples and CRM systems involved. Populate with technical requirements in collaboration with engineering.
- **[Lukasz Grabowski, Erik Andersson, Michal Rosikiewicz]:** Await use case documentation before scheduling follow-up architecture discussion.
- **[All participants]:** Focus development effort on smaller, validated features (e.g., survey support) rather than duplex synchronization until business case is confirmed.
- **[Agneta Lindahl Nevell]:** Clarify additional pre-filled forms requirements and share with the team.