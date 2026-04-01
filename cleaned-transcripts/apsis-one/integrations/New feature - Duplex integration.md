---
source_file: New feature - Duplex integration.txt
domain: Apsis One Integrations
topics: [duplex synchronization, bidirectional CRM data sync, generic connector, One API, profile attribute updates, pre-filled forms, event syncing, feature scoping, pilot customers]
speakers: [Erik Andersson, Michal Rosikiewicz, Lukasz Grabowski, Henrik Boye, Agneta Lindahl Nevell, "Speaker 1 (Marek)", Tomasz Kowalski]
key_components: [Generic Connector, One API, Apsis One, Tribe CRM, Dynamics CRM, Lime CRM, E-deal, FTC Enterprise, Site Shop plugin, pre-filled forms, surveys, event tool]
session_type: knowledge-transfer
---

## Session Overview

This session was an exploratory discussion about a proposed "duplex" (bidirectional) synchronization feature that would allow profile attribute changes made in Apsis One to be pushed back to external CRM systems. Currently, the data flow is strictly one-directional: CRM → Apsis One. The meeting surfaced a fundamental architectural decision — whether to implement this capability inside the **Generic Connector** (integration-specific) or as a general-purpose feature in **One API** (available to all customers). The group concluded that use cases and pilot customers need to be identified before any implementation path is committed to. The meeting also established that some of the event-syncing groundwork already exists.

---

## Current State: Unidirectional Data Flow and Existing Sync Capabilities

### The Historical Rule: CRM as the Source of Truth

[Erik Andersson]: The established rule has always been that all profile data originates from the CRM system. Changes to profiles flow from CRM → Apsis One, not the other way around. The proposed duplex feature represents a deliberate departure from this rule.

### What Currently Syncs Back to CRM

In the existing Pro/Apsis One integration:

- **Events** (e.g., clicks) are synced back to CRM
- **Consents** are updated back in CRM
- **Profile attributes** (e.g., first name, last name, subscriber attributes) are **not** pushed back to CRM

> [Lukasz Grabowski]: "Same here, except consents, right Eric? So consents are updated in CRM."

> [Speaker 1/Marek]: "In Pro, as we discussed, we have very similar setup and we never put the data back to CRM. Nobody asked for that. There was never a discussion about pushing data back to CRM — profile data. It is always overwritten when it's pulled back from CRM to Pro."

### Existing Event Tool Capability

[Erik Andersson]: The **event tool** already has syncing capability with ratings. E-deal, for example, already has support added for this. However, **Tribe has not implemented support for it yet** — but that implementation would be on Tribe's side, not on the Apsis integrations side.

> "In that aspect, we already can push the data, but Tribe has not implemented support for that, but that would not be anything for us to do."

---

## The Proposed Feature: Duplex Synchronization

### What Is Being Asked For

The feature request, originating from **Daniel** (internal stakeholder), is to allow profile changes made within Apsis One — particularly via forms, surveys, and marketing activities — to be pushed back to external CRM systems or other external platforms.

**Specific use cases raised by Agneta Lindahl Nevell:**

- Updating **marketing preferences** captured in forms, reflected back in CRM
- **Behavioral signals** (e.g., engagement patterns indicating purchase intent) fed back to CRM for sales team action
- **Segment membership** updates — e.g., "this contact now matches our 'purchase intent' segment" — pushed to CRM contacts
- **NPS/survey results** pushed back to CRM
- **Pre-filled form** data edits (profile attributes updated by the user) synced back to CRM
- General marketing-to-sales handoff signals: enabling CRM sales teams to see what the marketing platform knows about a contact

> [Agneta]: "Guess what? They want that information back as well to the CRM. So quite a lot of use cases for actually getting the data from Apsis to a CRM or another external system."

[Erik Andersson] noted that the scope sounds broader than just "push attributes" — it's more like pushing **behaviors, fulfillment states, and marketing signals**, not just raw attribute values. This would change the implementation story significantly from what was described on the Confluence page.

---

## Architectural Decision: Generic Connector vs. One API

### Option A: Generic Connector (Integration-Specific)

The feature would be built into the **Generic Connector**, making it available only to CRM systems that use it.

**CRMs currently supported by the Generic Connector:**
- Tribe
- FTC Enterprise (v12.1+)
- E-deal

**CRMs NOT supported (would not benefit):**
- Old Dynamics
- Old Enterprise
- Lime

**How the Generic Connector works per CRM:**
- **Dynamics**: requires a plugin
- New version (via **Site Shop** collaboration): Site Shop has built a Generic Connector plugin that adds all required endpoints; partners would need to enhance this for new features
- **FTC Enterprise**: built into core functionality

**Tradeoffs:**
- Less development effort for customers since the integration handles the plumbing
- But requires either Apsis or a partner to build plugin support — meaning partner engagement is a prerequisite for rollout
- Ties the feature exclusively to integration customers; non-integration customers cannot use it

### Option B: One API (Platform-Wide)

The feature would be implemented in **One API**, making it available to all Apsis One customers regardless of whether they have a CRM integration.

**Tradeoffs:**
- Available to all customers, not just those with integrations
- Requires customers to have their own R&D/IT capability or hire an integrator to consume the API and connect it to their CRM or tool
- More work for customers but maximum reach

[Erik Andersson]:
> "What he [Daniel] then does is he wants the customers to use One API... he wants customers to be able to add data to Apsis and get data from Apsis regardless of which CRM they use."

### Option C: Hybrid

[Michal Rosikiewicz] raised the possibility that the **Generic Connector could utilize One API functionality** internally — i.e., One API provides the platform capability, and the Generic Connector wraps it for CRM-specific use. Erik confirmed this is absolutely feasible.

### Current State of One API Overlap

[Speaker 1/Marek]: Some of the requested functionality — specifically checking whether a profile belongs to a segment — **already exists in One API**. The distinction is that currently customers pull that data (query-based), rather than having it pushed to them.

---

## Key Concerns and Risks Raised

### Risk: Building Without Validated Demand

[Speaker 1/Marek] raised a strong concern about proceeding without confirmed pilot customers:

> "If we at this point don't have any pilot customers, development will probably take a while, right? Then we need to figure — depending on the path — worst case, we will have to talk to partners to develop the plugins and integrations. And then in a quarter we will find out nobody is actually waiting for that. That will be a huge effort and huge waste of money."

> "We have huge tendency right now to spin up projects, create complex requirements, complex workflows. And if we see ourselves in a quarter with the fully working feature and nobody waiting for it — we could have done so many other things during that time."

### Risk: Feature Scope Creep

[Erik Andersson]: The use cases described by Agneta go well beyond simple attribute pushing. The Confluence page apparently described a narrower feature. The broader scope (behaviors, segment signals, NPS data) would require a substantially different implementation approach.

### Constraint: Tribe Plugin Not Implemented

Even though the event-tool sync capability exists on the Apsis side, **Tribe has not implemented support for receiving that data**. This is a blocker for Tribe-based customers and is outside the Apsis integration team's control.

---

## Decision: What Needs to Happen Before Implementation

The group reached consensus that **no implementation path should be chosen yet**. The following needs to happen first:

1. **Henrik Boye and Agneta Lindahl Nevell** will compile a list of concrete **use cases** on the Confluence page
2. **Pilot customers** must be identified — including both customers with CRM integrations and those without, since the latter group might favor the One API path
3. The use case list + customer profile will be shared with the development team (Lukasz, Michal, Erik), who will:
   - Identify which use cases are already supported
   - Assess remaining gaps
   - Make a more informed recommendation on implementation path

> [Lukasz Grabowski]: "For development, right now we should just focus on some smaller fixes and some known things. OK, this should work like this and not the bigger project like this duplex synchronisation."

[Erik Andersson] proposed that adding **survey support** to the Generic Connector would be a relatively simple near-term addition and a good place to start for the development team.

---

## Customer Context

### Uniunan

**Uniunan** was cited as one of Apsis's biggest customers and one that frequently requests more API capabilities. However, [Erik Andersson] noted that Uniunan is likely using **Dynamics**, which is **not** supported by the Generic Connector. This means if the duplex feature were implemented only via the Generic Connector, Uniunan would not benefit.

> [Agneta]: "The first one that comes to mind is Uniunan, because whenever we talk to them, they're shouting 'more APIs, more APIs' because they lack the functionality of getting information. But that's in the API more than a CRM integration."

### Business Unit / Reseller Strategy

[Agneta] noted that the architectural decision should also consider Apsis's strategy of using other business units as resellers. Any integration covering **Tribe, E-deal, and Enterprise** would be highly valuable as a revenue generator in that context. Standard, non-customized solutions (e.g., event tool getting data from surveys back into Tribe campaigns) were cited as the sweet spot.

---

## Pre-Filled Forms (Adjacent Topic)

Pre-filled forms were referenced multiple times as a related and highly anticipated feature. The duplex sync requirement is partly driven by pre-filled forms: when a user edits their profile data in a form, that updated data needs a path back to the CRM.

[Agneta] specifically mentioned **consent** as a top priority for pre-filled forms enhancement.

> [Agneta]: "Any further development of pre-filled forms will be quite appreciated, just mentioning it."

This discussion was deferred and Erik Andersson excused himself from the meeting when it began to shift toward pre-filled forms specifics.

---

## Key Takeaways

1. **Bidirectional sync is a genuine product need** with multiple validated use cases (marketing preferences, behavioral signals, segment membership, NPS/survey results, form data edits) — but customer demand hasn't been formally quantified yet.

2. **Two primary implementation paths exist**: Generic Connector (integration-specific, lower barrier for CRM customers, requires partner plugin work) vs. One API (platform-wide, higher barrier for customers, no partner dependency). A hybrid is also feasible.

3. **The Generic Connector path only works for Tribe, FTC Enterprise 12.1+, and E-deal** — it explicitly excludes old Dynamics, old Enterprise, and Lime. This is a significant scope limitation.

4. **Some capability already exists**: event tool syncing and ratings sync are already implemented on the Apsis side; Tribe has simply not consumed them yet.

5. **Segment membership queries are already available in One API** — customers can pull this today, though push-based delivery is not yet supported.

6. **No implementation work should begin** until pilot customers are identified and use cases are documented on Confluence.

7. **Near-term development focus** should be on smaller fixes and survey support in the Generic Connector, not the broader duplex synchronization project.

8. **The feature scope is wider than initially defined** on the Confluence page — it's not just "push attributes" but encompasses behavioral signals and marketing intelligence, which changes the implementation story significantly.

---

## Unresolved Questions and Action Items

| # | Action Item | Owner | Notes |
|---|-------------|-------|-------|
| 1 | Compile use cases on Confluence page | Henrik Boye + Agneta Lindahl Nevell | To be done before next follow-up meeting; timeline uncertain (possibly not next week) |
| 2 | Identify pilot customers interested in duplex sync | Henrik Boye + Agneta Lindahl Nevell | Should include both integration customers AND non-integration customers to inform path decision |
| 3 | Follow-up meeting to review use cases and make implementation path decision | Lukasz Grabowski (organizer) | Scheduled after item 1 is complete |
| 4 | Add survey support to Generic Connector | Erik Andersson / dev team | Flagged as relatively simple near-term addition |
| 5 | Clarify Uniunan's CRM setup | — | ⚠️ Ambiguous: Erik suggested Uniunan uses Dynamics (not supported by Generic Connector), but this was not definitively confirmed |
| 6 | Pre-filled forms: consent improvements | — | Deferred; Agneta to align with Henrik on specific requirements |

> ⚠️ **Unresolved architectural question**: Whether to implement duplex sync in the Generic Connector, One API, or a hybrid remains open pending use case documentation and pilot customer identification.