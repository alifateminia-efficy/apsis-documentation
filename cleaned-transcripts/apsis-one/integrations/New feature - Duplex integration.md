---
source_file: New feature - Duplex integration.txt
domain: Apsis One Integrations
topics: [Bidirectional Profile Synchronization, Data Flow Direction, API-driven vs Integration-specific Solutions, Generic Connector Implementation, Pre-filled Forms, CRM Data Sync Requirements]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Speaker 1 (Architecture/Product Lead), Henrik Boye (Product), Agneta Lindahl Nevell (Customer Success)]
key_components: [Generic Connector, One API, CRM Systems, Event Synchronization, Pre-filled Forms, Survey Integration, Tribe, E-deal, Efficy Enterprise variants]
session_type: architecture-review
subdomains: [Architecture, Generic Connector, Inbound Flow, Outbound Flow, Microsoft Dynamics, Efficy Enterprise 12.1, Tribe, E-deal (Efficy Corporate)]
---

## Session Overview

This architecture review discussed a new feature request for **duplex (bidirectional) synchronization** of profile attribute changes between Apsis One and external CRM systems. Historically, data flow was unidirectional: CRM→Apsis only. The discussion centered on two competing architectural approaches: building the feature via the **generic connector** (CRM-integration-specific) versus via a **public One API** (available to all customers). The team identified that without concrete customer use cases and pilot customers, the effort-to-value ratio was unclear, and deferred detailed design until customer requirements could be validated.

---

## Core Feature Request: Bidirectional Profile Synchronization

### Current State: Unidirectional Data Flow

The existing integration model has been strictly unidirectional:

> "The main rule has always been all data should come from the CRM system" [Erik Andersson]

Currently supported downstream from Apsis→CRM:
- **Events**: clicks, conversions, behavioral data
- **Leads**: new lead creation via forms
- **Consents**: explicit consent changes are synced back

**Not currently synced back from Apsis→CRM**:
- Profile attribute changes (first name, last name, custom attributes)
- Segment membership changes
- Marketing preference updates
- Survey responses (NPS, etc.)

### Why Change Is Being Requested

The driving use case comes from **pre-filled forms** functionality. When customers use forms in Apsis to collect or update profile data, they need that modified data pushed back to their CRM systems without overwriting existing CRM-sourced data. This enables workflows where marketing and sales teams work with updated information from both systems.

---

## Two Architectural Approaches: Detailed Comparison

### Approach 1: Generic Connector Implementation

**What it means**: Build the bidirectional sync capability into the **generic connector** (the standard plugin framework for CRM integrations).

**Supported CRM systems** (via generic connector):
- Tribe
- Efficy Enterprise 12.1
- E-deal (Efficy Corporate)

**NOT supported** (no generic connector):
- Old Dynamics (legacy)
- Old Enterprise (legacy)
- Lime CRM

**Implementation requirements**:
- We (Apsis) implement the feature in the generic connector codebase
- CRM vendors or partners must then build/update their connector plugins to handle the new endpoints
- For example, Tribe integration with Site Shop would need to add support for receiving and processing the outbound data

**Customer burden**: Lower—integration partner typically handles the setup

**Limitation**: Feature is only available to customers already using supported CRM integrations.

[Erik Andersson]: "Because if we build this using the integration and the general connector, then of course we can do this absolutely no issue. But then you need to have an integration for you to be able to do this. But if you sit and have like a garage implementation and you want to say I want to have updates... they would not be able to utilize this."

### Approach 2: Public One API Solution

**What it means**: Build a standardized **public API endpoint** that any external system (CRM or custom) can call to receive or query profile updates. This is not integration-specific.

**Supported systems**: Any system that can make HTTP requests to Apsis's public API

**Implementation requirements**:
- We (Apsis) build and maintain the API endpoint
- Each external system (CRM, custom solution, third-party tool) must develop their own integration logic to consume the API
- This follows the principle of "CRM systems should utilize the one API key that we have created for them to register for the functionality"

**Customer burden**: Higher—customers may need internal R&D or external development partners to build the integration

**Advantage**: Available to all Apsis customers regardless of which CRM (or no CRM) they use

**Synergy**: The generic connector *itself* could consume this One API, meaning both approaches aren't mutually exclusive—the generic connector becomes a consumer of the standardized API.

> "Absolutely, absolutely" [Erik Andersson, agreeing that generic connector could utilize the One API functionality]

---

## Customer Demand & Business Case Analysis

### The Critical Unknown: Do Customers Actually Want This?

A significant tension emerged during discussion around **whether we have validated customer demand**:

**Speaker 1 (Architecture Lead)** raised repeated concerns:
- In **Pro** (predecessor platform), they had similar subscription form capabilities with event sync back to CRM, but customers never asked for profile attribute sync back
- "Nobody asked for that. As far as I remember, there was never, never a discussion about pushing data back to CRM."
- Without pilot customers, this could be a major effort with zero uptake: "If we don't have any pilot customers, that would probably mean that nobody is interested in that feature"
- Risk of wasted effort: "In 1/4 we will find out nobody, we don't have any pilots because nobody is actually waiting for that. That will be a huge effort and huge kind of waste of money."

**Henrik Boye (Product)** acknowledged the risk but indicated interest coming from business stakeholders:
- Request originated from Daniel (internal stakeholder)
- Mentioned Uniunan as a potential interested customer
- Initially had "no clue" of specific use cases but acknowledged the workflow makes sense

### Agneta Lindahl Nevell's (Customer Success) Use Cases

Once Agneta joined the discussion, she articulated compelling business use cases that justified the feature:

**Key Use Case 1: Marketing-to-Sales Handoff**
> "In the CRM you have sales and commercial teams working updating data with opportunities and things like that. But the marketing team, they're working in apps, doing marketing activities. So whenever there's a change in preference or a change in behavior, that information would of course be valuable also to get back in the CRM"

**Key Use Case 2: Purchase Intent Signaling**
> "If marketing has a great segment for identifying behaviour that indicates a purchase intent. Synchronizing that information back also to contacts in the CRM saying that this contact is now ready for for purchase or it's time for you to reach out."

**Key Use Case 3: Behavior Correlation & Churn Detection**
- Marketing changes, behavioral attributes
- NPS survey responses
- All need to flow back to CRM so sales understands context

**Key Use Case 4: Strategic Revenue Impact**
Agneta emphasized this feature supports Apsis's strategic positioning:
> "This aligns with the strategy decision... where we want to put apps in the future as the marketing and customer communication platform enabling smarter marketing across brands"

**Key Use Case 5: Cross-Business Unit Reseller Strategy**
- Feature would be valuable through reseller channels (Tribe, Enterprise, E-deal)
- Standard solutions (non-customized event/survey data) pushed through generic connector would create "high value" and "revenue generator"

**Specific Customer Mentioned**: Uniunan
- "Quite a lot actually" of requests for data sync
- Primary request is **API-driven** rather than CRM-specific
- Currently using old **Dynamics** (which has no generic connector)

---

## Key Technical Details & Existing Capabilities

### Event Tool Already Supports Some Sync

[Erik Andersson] clarified that some capabilities already exist:
- Event tool can push events back to CRM systems
- Rating data can be synced
- "We have the rating and everything added now. Also E little for example. I've added support for this if you would want."

**However**, Tribe has **not implemented support** for consuming these events on their side, so it's a missing CRM-side feature, not an Apsis limitation.

### Pre-filled Forms: Separate But Related Feature

Pre-filled forms capability (partially implemented) is the catalyst for this feature request, but development of that feature should continue separately. Agneta requested:
- Further enhancement of pre-filled forms functionality itself
- **Consent** handling in forms (top priority)

[Agneta]: "Consent. Consent. First thing, yeah."

---

## Decision: Defer Detailed Design Pending Requirements

### Action Items & Next Steps

**Henrik Boye & Agneta Lindahl Nevell** will:
1. **Document concrete use cases** - Write detailed scenarios with specific customer names/types
2. **Identify pilot customers** - Determine which customers would be willing to test/validate the feature
3. **Clarify integration scope** - Determine whether customers need:
   - API-driven solution (available to all)
   - Generic connector solution (CRM integrations only)
   - Combination of both
4. **Populate Confluence page** with findings

**Timeline**: Henrik indicated this may not happen next week due to other priorities.

> "Let me and Agneta go through potential use cases and we put them on the confluence page and then we get back to you." [Henrik Boye]

### Why Detailed Design Is Blocked

[Speaker 1]: "We will not be able to discuss that... before we get some feedback from customers interested in that because we need to know whether we go with the generic connector or we go with one API and for that we need to know which customers are interested in that."

The architectural choice between API vs. generic connector has significant downstream implications:
- **Generic connector**: Faster for CRM customers, requires partner plugin updates, limits reach
- **One API**: Slower for customers to adopt, available to all, requires standardization effort

This decision cannot be made on technical merit alone—it requires customer and revenue context.

---

## Interim Development Focus: Smaller, De-risked Work

Rather than pursuing the ambitious duplex sync feature immediately, the team agreed to focus on smaller improvements with known value:

**Potential interim work**:
- **Survey sync support** - Making survey/NPS responses available for sync (relatively simple capability)
- **Pre-filled forms enhancements** - Continued development of form-based data collection
- **Known bug fixes and small feature completions** in the integration scheme

[Lukasz Grabowski]: "For development, right now we should just focus on some smaller fixes and some known things that OK, this should work like this and not the bigger project like this duplex duplex synchronisation."

[Erik Andersson] offered: "We can add support for surveys, so that would be fairly simple."

---

## Strategic & Organizational Context

### Apsis's Positioning

The feature request aligns with stated strategic direction:
- Apsis positioning itself as "marketing and customer communication platform enabling smarter marketing across brands"
- Deepened CRM integration supports this vision
- Cross-business unit reseller strategy (Tribe, Enterprise, E-deal) benefits from better CRM sync

### Resource Constraints & ROI Discipline

[Speaker 1] emphasized organizational discipline around feature investment:
> "We have very scarce resources now and we we should be at least considering where we invest them so we get the the best return on the investment"

This is not just about feature prioritization—it's about avoiding the anti-pattern of building complex features for hypothetical customers when there's no validation.

---

## Unresolved Questions & Ambiguities

### What Exactly Is "Duplex Integration"?

The term "duplex integration" was used throughout but never formally defined. The discussion suggests it means:
- Bidirectional attribute synchronization at the profile level
- With ability to filter/control what attributes sync back (not just mirror everything)
- Respecting existing data ownership rules (don't overwrite CRM-sourced attributes from Apsis changes)

However, some of Agneta's use cases (segment membership, behavior detection) suggest it might be broader than attribute sync alone—potentially including **computed properties** (segment membership, scoring, etc.) rather than just raw attributes.

[Erik Andersson] noted this ambiguity:
> "I think it would be valuable like if we have those use cases... then we can put our heads together and see... because it doesn't sound just like push attributes. It's more sounds like push, like a behavior and marketing or fulfillment or something, not not just attributes."

### API Capabilities: Check vs. Push

[Speaker 1] raised an important distinction:
- Apsis **One API already supports checking** if a profile belongs to a segment: "Some of those features that you mentioned like whether a profile belongs to a segment, we already have that in the in the the one API, so they could use that already."
- Question is whether customers need **us pushing** this data vs. them **pulling** it via API calls

This distinction affects design: is the feature about event-driven push, polling capability, or both?

### Tribe's Event Implementation Status

Erik mentioned Tribe *could* receive events but hasn't implemented consumer-side support. It's unclear whether:
- This is a planned feature on Tribe's roadmap
- We need to work with Tribe to add it
- We should build a different integration mechanism if Tribe won't consume events

---

## Key Takeaways

1. **Feature Request Is Strategically Sound But Unvalidated**
   - Makes sense from marketing-sales integration perspective
   - Aligns with Apsis's platform strategy
   - But lacks concrete customer commitment and pilot customers

2. **Two Viable Architectural Paths Exist, Each with Tradeoffs**
   - **Generic Connector**: Easier for CRM customers, limited reach, partner dependency
   - **One API**: Universal reach, harder for customers to adopt, standardization burden

3. **Architecture Decision Is Blocked Until Customer Requirements Are Captured**
   - Need specific use cases, not just abstract scenarios
   - Need pilot customer commitment
   - Need visibility into which customer segments (CRM vs. non-CRM, API-native vs. traditional) are interested

4. **Existing Capabilities Are Underutilized**
   - Event sync already works technically (Tribe doesn't implement consumer side)
   - One API already supports segment membership checking
   - Some survey/rating data can already be synced

5. **Resource Allocation Should Follow Customer Validation**
   - Small interim work (surveys, pre-filled form enhancements) is lower-risk and has known value
   - Large architectural effort should wait for pilot customers and clear use case prioritization

6. **Further Discussion Scheduled**
   - Henrik and Agneta will compile use case documentation and Confluence updates
   - Follow-up meeting TBD (not next week)
   - Architecture team (Erik, Lukasz, Michal) will review requirements once documented

---

## Next Steps & Action Items

| Owner | Action | Status |
|-------|--------|--------|
| Henrik Boye, Agneta Lindahl Nevell | Document concrete use cases with customer names/scenarios | Pending |
| Henrik Boye, Agneta Lindahl Nevell | Identify pilot customers willing to validate feature | Pending |
| Henrik Boye, Agneta Lindahl Nevell | Populate/update Confluence page with findings | Pending |
| Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz | Review requirements and architectural decision once use cases documented | Blocked - awaiting requirements |
| Development Team | Focus interim effort on surveys support and pre-filled form enhancements | Approved |