---
source_file: New feature - Duplex integration.txt
domain: Apsis One Integrations
topics: [duplex synchronization, bidirectional CRM sync, profile attribute updates, generic connector, One API, pre-filled forms, use case validation, feature scoping]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Henrik Boye, Agneta Lindahl Nevell, Speaker 1 (Marek - unconfirmed last name), Tomasz Kowalski]
key_components: [Generic Connector, One API, Apsis One, Tribe, Dynamics, E-deal, FSC Enterprise, pre-filled forms, surveys, Confluence]
session_type: architecture-review
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

This session was a scoping and architecture discussion around a proposed new "duplex" (bidirectional) synchronization feature: allowing profile attribute changes made in Apsis One to be pushed back to external CRM systems. Currently, the data flow is unidirectional — from CRM to Apsis — with the exception of events (clicks) and consents. The group debated two implementation paths: building the feature into the **Generic Connector** versus exposing it through the **One API** for all customers. The session surfaced a key concern about the lack of validated pilot customers before committing to development, and concluded with an action item for Henrik and Agneta to document concrete use cases on Confluence before the next meeting. The group also agreed that smaller, incremental integration work (e.g., survey support) should proceed in parallel.

---

## Background: Current State of Data Flow Between Apsis and CRMs

### What is Currently Supported

Today, the data flow between CRM systems and Apsis One is largely **unidirectional**: data originates in the CRM and is pushed to Apsis. The following items are synced back from Apsis to CRM:

- **Events** (e.g., email clicks) are synced back to CRM
- **Consents** are updated in CRM from Apsis

What is **not** currently supported:
- Profile attribute changes made in Apsis (e.g., via form submissions) being pushed back to CRM

[Erik Andersson]: The established rule has always been that all data should come from the CRM system. The proposed feature would break from that convention by enabling Apsis-to-CRM attribute updates.

### Historical Parallel: Apsis Pro

In Apsis Pro, a similar unidirectional architecture exists: CRM data flows into Pro, and only events are synced back. Profile attribute changes made in Pro are **not** pushed back to the CRM and, historically, no customers have asked for this. Data pulled from CRM to Pro always overwrites any local changes.

> "In Pro, as we discussed, we have a very similar setup and we never put the data back to CRM. Nobody asked for that." — Speaker 1 (Marek)

---

## The Proposed "Duplex" Feature: Bidirectional Sync

### Feature Description

The proposed feature would allow external systems (CRMs or others) to receive updates when profile attributes are changed in Apsis One — for example, when a contact updates their marketing preferences via a pre-filled form, or when behavioral signals indicate purchase intent.

**Motivating use cases described by Agneta Lindahl Nevell:**
- Marketing preference updates that should be visible to sales teams in CRM
- Behavioral signals (e.g., purchase intent indicated by segment membership) surfaced back to CRM to trigger sales handoff
- NPS/survey response data pushed back to CRM
- Segment membership status synced back to CRM contacts (⚠️ partially already available via One API)
- Form submissions with updated areas of interest feeding back to CRM for smarter sales targeting

[Erik Andersson]: Some of this already exists. Event tool syncing with ratings is supported, and E-deal has support added for this. However, **Tribe has not implemented support for receiving this data**, and that would not be Apsis's responsibility to develop.

> "It doesn't sound just like push attributes. It sounds more like push behavior and marketing fulfillment or something — not just attributes." — Erik Andersson

---

## Architecture Decision: Generic Connector vs. One API

### Option A: Build into the Generic Connector

**How it would work:** The duplex sync functionality would be implemented within the Generic Connector framework. CRM partners/plugins would need to add endpoints to receive the pushed data.

**Which systems this covers (important limitation):**
The Generic Connector is only available for specific CRM systems. It does **not** cover:
- Old Dynamics (legacy)
- Old Enterprise
- Lime

It **does** cover:
- Tribe
- FSC Enterprise 12.1
- E-deal

For Dynamics, a plugin is required. For the newer version, Site Shop has built a Generic Connector plugin that adds all required endpoints. For FSC, the functionality is built into core.

**Pros:**
- Less development work required from CRM partners/customers compared to a fully custom API integration
- Standardized implementation path for supported CRMs

**Cons:**
- Excludes legacy CRM integrations (old Dynamics, Lime, old Enterprise)
- Still requires plugin updates from partners (e.g., Tribe would need to implement support)
- Ties the feature to integration customers only — non-integration customers cannot use it

### Option B: Build into One API (Platform-Level)

**How it would work:** The feature would be exposed via One API, available to all Apsis One customers regardless of whether they have a CRM integration. CRM systems would then use their existing One API key to register for and consume these events/updates.

**Pros:**
- Available to all customers, not just integration customers
- Aligns with Daniel's stated vision: any CRM should be able to add data to Apsis and get data from Apsis, regardless of which CRM it is
- Generic Connector could still leverage this One API functionality internally

**Cons:**
- Higher implementation burden on the customer side — requires R&D or IT capacity, or hiring an external company to build the integration
- More development effort on the Apsis side before any CRM can benefit

[Henrik Boye]: Daniel's intent was specifically that the solution should not be CRM-specific — any external system should be able to interact with Apsis.

> "What Daniel wanted is that we shouldn't care which CRM wants to integrate with us. They should be able to add data to Apsis and get data from Apsis." — Henrik Boye

[Erik Andersson]: If built via One API, the CRM systems should then utilize that, rather than having something tailor-built inside a specific integration. The Generic Connector could also utilize this One API functionality internally.

---

## Key Concern: Lack of Validated Demand and Pilot Customers

Speaker 1 (Marek) raised a strong process concern: the team risks investing significant development effort — including potential partner plugin development — into a feature with no confirmed pilot customers.

> "If we at this point don't have any pilot customers, development will probably take a while, then we need to figure out generic depending on the path. Worst case, we will have to talk to partners to develop the plugins and integrations. And then in a quarter we will find out nobody is actually waiting for that. That will be a huge effort and huge waste of money." — Speaker 1 (Marek)

**Known potential customer interest:**
- **Uniun** (described as one of the biggest customers) — has expressed demand for more API access; they want to get information out of Apsis. However, Uniun is likely on Dynamics, which is **not** covered by the Generic Connector path.
- Agneta confirmed that the question "what data can be synchronized back to CRM?" is received frequently in customer conversations.

**Decision made:** Before committing to an implementation path, Henrik and Agneta will compile a list of:
1. Concrete use cases (documented on Confluence)
2. Named pilot customers who have expressed interest
3. Which integrations/CRM systems those customers are using (to inform the Generic Connector vs. One API decision)

---

## Existing Partial Support Worth Noting

[Erik Andersson] flagged that some of the described use cases are already partially supported and should be checked against the use case list before scoping new work:

- **Event tool syncing** (including ratings) already exists in the Generic Connector; E-deal has support added
- **Segment membership queries** are already available via One API — customers can query whether a profile matches a segment, though this is pull-based (customer queries Apsis) rather than push-based (Apsis notifies customer)
- **Survey support** could be added to the Generic Connector relatively simply and was suggested as a near-term incremental task

---

## Implementation Approach for Near-Term Development

Given the unresolved questions about pilot customers and direction, the group agreed the team should **not** begin building the duplex feature immediately. Instead:

- Focus on **smaller, known fixes** and incremental improvements within the integration scope
- **Survey support** in the Generic Connector was identified as a feasible near-term addition
- The duplex synchronization project should wait for the use case and customer validation from Henrik and Agneta

[Lukasz Grabowski]: The purpose of this session was not to start implementation, but to understand the problem space and design direction with Erik's input on correct architectural approach.

---

## Pre-Filled Forms — Adjacent Topic

Agneta briefly highlighted strong customer interest in continued development of **pre-filled forms**, specifically mentioning consent handling as a priority area. Erik stepped out of the conversation at that point, indicating this was outside his scope for the session.

---

## Key Takeaways

1. **Current architecture is unidirectional** (CRM → Apsis), with the only reverse flows being events (clicks) and consents. No profile attribute push-back exists today.

2. **Two valid implementation paths exist**: Generic Connector (integration-customers only, covers Tribe/FSC/E-deal but not legacy Dynamics/Lime) vs. One API (platform-wide, higher customer implementation burden). The Generic Connector could also be built on top of a One API implementation.

3. **Daniel's stated vision favors the One API path** — any CRM should be able to integrate regardless of which system it is.

4. **The feature scope is broader than "push attributes"** — use cases include behavioral signals, segment membership, NPS/survey results, and marketing preference changes, some of which have different technical requirements.

5. **Some functionality already exists**: event syncing via the Generic Connector (E-deal has it; Tribe has not implemented it), and segment queries via One API.

6. **No confirmed pilot customers yet** — this is a blocking concern before committing development resources to either path.

7. **Near-term development recommendation**: survey support in Generic Connector is a small, tractable addition that can proceed now.

---

## Unresolved Questions and Action Items

| Item | Owner | Notes |
|---|---|---|
| Document concrete use cases for duplex sync on Confluence | Henrik Boye + Agneta Lindahl Nevell | Includes behavioral sync, NPS, preferences, segment membership |
| Identify named pilot customers and which CRM/integration they use | Henrik Boye + Agneta Lindahl Nevell | Critical for deciding Generic Connector vs. One API path |
| Check which use cases are already covered by existing functionality | Erik Andersson + team | Some event sync and One API segment queries already exist |
| Schedule follow-up meeting once use cases are documented | Lukasz Grabowski | Henrik noted it may not happen the following week |
| Confirm whether Uniun is on Dynamics (which would rule out Generic Connector path for them) | ⚠️ Unresolved | Erik suggested Uniun is likely on Dynamics and would not be covered by Generic Connector |
| Add survey support to Generic Connector | Development team | Flagged as a small, near-term deliverable independent of the larger duplex decision |