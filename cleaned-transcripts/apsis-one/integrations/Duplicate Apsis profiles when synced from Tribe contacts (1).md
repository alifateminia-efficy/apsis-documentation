---
source_file: Duplicate Apsis profiles when synced from Tribe contacts (1).txt
domain: Apsis One Integrations
topics: [Contact Deduplication, CRM ID Management, Lead-to-Contact Conversion, Tribe Integration, Profile Merging, Relationship Hierarchy]
speakers: [Agneta Lindahl Nevell, Lukasz Grabowski, Speaker 1 (likely Grzegorz Kozub or technical lead), Henrik Boye, Tomasz Kowalski]
key_components: [Apsis One API, Tribe CRM, Profile Merge Endpoint, Contact Entity, Person Entity, CRM ID, Relationship ID (UUID), Apsis Lead Company, Justin (integration/automation tool)]
session_type: knowledge-transfer, architecture-review, debugging-session
subdomains: [Contact Management & Deduplication, Integration Architecture & Data Flows]
---

## Session Overview

This session discusses a critical issue in the Apsis One–Tribe CRM integration: duplicate profile creation in Apsis when contacts are synced from Tribe. The core problem stems from Tribe's data model where a **person** can have multiple **relationships** (contact roles) across different companies, each with a unique **relation ID**. When Tribe users assign a lead (initially created as an "Apsis lead") to a real company, or move an existing contact to a new company, duplicate CRM IDs are created in Apsis. The team explores three primary scenarios, proposes a merge-based solution leveraging existing Apsis APIs, and identifies key implementation challenges around metadata transfer and orchestration of merge calls.

---

## Tribe CRM Data Model and the Root of Duplication

### Person, Relationship, and Contact Identities

Tribe's data model distinguishes between three core concepts:

- **Person ID**: A unique identifier for an individual in the system. The person entity holds attributes like name and contact information common across all roles.
- **Relation ID (UUID)**: A unique identifier for a specific relationship between a person and a company. This is what Tribe calls a **contact**.
- **Company**: An organizational entity to which a person can be related in multiple roles (e.g., employee, lead, consultant).

[Agneta Lindahl Nevell]: In Tribe, you have a person with a person ID, and that person can have different roles across different companies. Each role is a separate relationship, which gets its own UUID. This relation ID is what we use as the contact on the Apsis side—it's the identifier of the contact, the role the person has when connected to a company.

The critical insight: **Apsis uses the relation ID (contact identifier) as its CRM ID**, not the person ID. This design choice was deliberate because it preserves the B2B hierarchy—in Apsis, a person's role (e.g., "CEO") is attached to their relationship with a company, not to the person record itself.

[Agneta Lindahl Nevell]: In the B2B world, you are your role. A decision was made to work with the contact (relation) because that's what the CRM manages—the contact, not the persons. If we switched to using the person ID, we'd lose access to the hierarchy and relationship-specific attributes.

### Why Person ID Cannot Be Used

[Speaker 1]: Why can't we just use person ID for CRM ID on our side?

[Agneta Lindahl Nevell]: If we chose person ID, we can only fetch information on the person entity, not on the contact entity. We don't have the hierarchy, the relationship hierarchy. My role as a CEO is on the relationship (contact), not on me as a person.

This design has important consequences: **multiple contacts in Apsis can share the same email address because they represent the same person in different professional roles**.

---

## The Three Duplication Scenarios

### Scenario 1: Lead Creation → Company Assignment (Apsis-Generated Lead)

A user fills out an Apsis form, creating a lead that is initially assigned to a fictional "Apsis lead" company in Tribe. Later, the user validates the lead and assigns it to a real company.

**What happens:**
1. Lead arrives from Apsis form → Tribe creates a contact under "Apsis lead" company → Tribe sends back a relation ID (CRM ID).
2. Apsis receives this CRM ID and stores it as the contact's CRM ID.
3. User assigns the contact to the real company → Tribe creates a **new relation ID** for this person-company pair.
4. Tribe syncs this new contact → Apsis receives a new CRM ID.

**Result:** Apsis now has two profiles for the same email address, both with different CRM IDs—one from the lead under "Apsis lead," one from the contact under the real company.

[Agneta Lindahl Nevell]: We get duplicates. We don't want that. In that case we want it to be the same. We moved it to the correct company. We didn't create a new one, but we moved it.

**Desired outcome:** Merge the two profiles, using the new CRM ID as the primary key, but preserve the history showing that this contact was originally a lead.

### Scenario 2: Lead → Contact with Wrong Email (Within Same Company Context)

A lead is created but the user realizes the email was incorrect or it's a new contact on the same company. The user updates the relation, correcting the email but keeping the same relation ID.

**What happens:**
- On Tribe's side, the relation might be updated, but details are unclear in the discussion.
- The same duplication manifests because the original lead contact and the corrected contact appear as separate entities.

**Desired outcome:** Merge them as the same contact.

### Scenario 3: Consultant/Multi-Company Relationship (Legitimate Duplication)

A consultant works for multiple companies and has legitimate, distinct contacts in Tribe for each company relationship. The consultant's email is the same across all contacts (e.g., a corporate email used for multiple client engagements).

[Agneta Lindahl Nevell]: Say that I am a consultant and I do HR consultancy work for company A, but I do financial controller tasks for Company B. Then I would be two contacts with the same email address, but I'm actually two contacts. I'm the same person, but I'm two contacts.

**What happens:**
- Apsis correctly receives two profiles, one for each company relationship, both with the same email address.

**Desired outcome:** Keep both profiles separate—do NOT merge them. This is the expected and correct behavior.

---

## The "Schizophrenia" Problem: Multiple Profiles with Same Email

The term "schizophrenia" emerged to describe a contact with multiple legitimate personas in Apsis:

[Speaker 1]: So this schizophrenia in Apsis 1 manifests in just two separate profiles with different CRM IDs and similar other attributes, right?

[Agneta Lindahl Nevell]: Yes, multiple personalities, yeah.

This is **expected and acceptable** for true multi-company relationships (Scenario 3) but **undesirable** when it arises from lead qualification or contact reassignment (Scenarios 1 and 2).

**Key constraint:** Apsis One has no awareness of company context in its profiles. When syncing, Apsis doesn't know which company a contact belongs to—it only knows CRM ID and attributes. This makes server-side deduplication logic at the Apsis layer problematic.

[Speaker 1]: We don't have anything about the company in Apsis one, right? When we sync, we don't know that the profile is from such and such company.

---

## CRM ID Visibility and the Hidden Identifier Problem

A critical operational issue: **The CRM ID (relation UUID) is not visible to Tribe customers**, even though it is the only guaranteed unique identifier for a contact.

[Agneta Lindahl Nevell]: The unique identifier is the system UUID, but the customer cannot see this. So if I want to find this contact, I can't. It's not visible to customers. I can see it because I'm an FSC person, but if I was a real customer, I wouldn't see this.

**Implications:**
- Customers cannot search for or reference profiles using the CRM ID.
- Searching by email may return results after a long delay (5–20 minutes, sometimes 420 minutes on Friday) due to sync lag.
- This creates a UX problem and makes manual merging difficult.

[Agneta Lindahl Nevell]: If you don't know the unique identifier, you can't find it immediately. It takes time. So if all I know is the email address, it will take time and then I will find duplicate responses because I don't have the unique identifier.

---

## Proposed Solution: Merge-Driven Deduplication via Tribe

The team converges on a **Tribe-initiated merge approach** rather than Apsis-side logic.

### Core Principle

The decision of **whether to merge or keep separate** should remain with the Tribe user. Tribe has the business logic and context to know when Scenarios 1/2 (merge) versus Scenario 3 (keep separate) apply. Apsis should provide the capability; Tribe should provide the orchestration.

[Speaker 1]: The solution would be something around: Apsis has APIs, endpoints, passive connections, and when these scenarios are performed in Tribe by the Tribe user, they select whether to merge or not. I think that's the solution. I don't think the solution would be to have any kind of logic on our side on Apsis One side to decide whatever to do with those kind of profiles.

### The Merge Endpoint

Apsis One already has a **profile merge endpoint** in its API that accepts:
- Keyspaces (including CRM keyspace)
- Two profile IDs in the CRM keyspace to merge

[Speaker 1]: I think we already have profile merge endpoint in Apsis one API. This accepts keyspaces including CRM keyspace and the IDs for both of these profiles in the CRM keyspace.

Additionally, there is existing **merge webhook functionality** (possibly via Justin, an internal integration/automation tool):

[Lukasz Grabowski]: We can use Justin for this. At least I can see some conversation with Eric that there is such functionality in Justin to just give... I would like to merge this contact during some samples, somewhere, so this exists.

### Two Implementation Pathways Discussed

#### Path A: Tribe Calls Apsis Merge Endpoint Directly

Tribe, when it identifies a merge scenario, directly calls the Apsis One merge endpoint with:
- Both CRM IDs to merge
- Specification of which CRM ID becomes the primary key

**Advantage:** Minimal work on the Apsis side; pure API call.
**Challenge:** Tribe needs to know about this capability and implement the logic.

#### Path B: Metadata-Driven Merge (Preferred by Apsis Team)

Tribe stores metadata on contacts indicating related/predecessor contact IDs. When Apsis syncs a contact, it fetches this metadata and:
1. Receives the new contact and any metadata indicating CRM IDs to merge with.
2. Creates the new profile in Apsis.
3. Automatically triggers merge of the new profile with the profiles identified in the metadata.

[Speaker 1]: If we sync a profile and it has this list [of IDs to merge], we merge those profiles—the profile we're syncing and the profiles identified by this list, because those will be the source profiles, the lead profiles.

**How Tribe would implement it:**
- When a lead (relation ID: `abc-123`) is assigned to a company, Tribe creates a new contact (relation ID: `xyz-789`).
- Tribe stores `abc-123` in the new contact's metadata (a new field or hidden field in Tribe).
- When syncing `xyz-789` to Apsis, Tribe includes this metadata.
- Apsis receives both IDs and performs the merge.

**Advantage:** Apsis doesn't need to call external APIs; Tribe handles the orchestration.
**Drawback:** Requires work on both Tribe's side (to add and maintain metadata) and Apsis's side (to fetch and use this metadata during sync).

[Speaker 1]: This new contact also has relation to contact generated from Apsis lead, right? So when they do that, when they create this contact, they need to either take the ID of that Apsis lead generated contact and put it somewhere in the metadata of that new contact, or when we sync that new contact, we need to be able to fetch that ID.

### Control and Flexibility

Both pathways preserve **Tribe's control**: if Tribe doesn't include a merge signal, Apsis treats the profiles as separate.

[Speaker 1]: This is fine. This is great because then Tribe, when they don't want to merge, they then don't assign. They don't assign this metadata then that's it. It's in control of Tribe.

---

## Race Condition: Timing of Profile Creation and Merge Calls

A subtle timing issue emerged:

When Tribe creates a new contact and wants to merge it with the predecessor, there's a race condition:
1. Tribe calls merge endpoint with two CRM IDs: old and new.
2. But the new profile hasn't been synced to Apsis yet.
3. Merge endpoint returns **not found** error for the new profile.

[Speaker 1]: If I have this Apsis lead profile and I create a new profile in Tribe and now in this context I know I want them to be merged, so what I do is I call our merge endpoint with these two IDs. However, this new profile in Tribe has not yet been synced. So this endpoint returns a sad not found error message.

**Mitigation:**

Tribe should create the contact first (which triggers immediate sync via online API call), then call the merge endpoint. Apsis will return the new CRM ID in the sync response, so the new ID is known immediately.

[Lukasz Grabowski]: When we send them this lead, they immediately assign CRM ID and respond to us with the ID.

[Speaker 1]: So they create a new profile in our system immediately by an online API call. OK, so then they can just call the merge endpoint after that.

---

## Tribe's Perspective and Challenges

Agneta represents the Tribe stakeholder perspective and emphasizes that Tribe developers have **identified this issue but struggled to implement a solution**.

[Agneta Lindahl Nevell]: The developers on the Tribe side have reached out a couple of times with ideas how to solve this. I was pushing for... (various ideas, but Tribe lacks clear direction on how to implement merge triggers).

### One Tribe Proposal (Rejected by Apsis)

Tribe suggested: if a contact's origin is "Apsis lead" and you update the relationship to an existing real company, merge automatically. Otherwise, don't merge.

[Agneta Lindahl Nevell]: One idea being that if the company made-up company is Apsis lead and we update the relationship to an existing real company, then it should merge. But if the origin is not Apsis lead, then they shouldn't merge.

**Apsis team's concern:** This doesn't align with Scenario 3 (legitimate multi-company). A consultant's Apsis lead origin doesn't mean their later company assignments should auto-merge.

### Tribe's Constraint

Tribe treats the creation of new relation IDs as core functionality akin to keyspaces. Changing Tribe's behavior to avoid creating duplicate relation IDs would be a fundamental architectural change, which is not feasible.

[Speaker 1]: I believe this is to them a core functionality similar to key spaces to us. So changing that for them would not be an easy fix.

---

## Outstanding Questions and Action Items

### Questions to Address with Tribe (in follow-up developer meeting)

1. **How will Tribe identify and communicate merge scenarios to Apsis?**
   - Via direct API calls to Apsis merge endpoint?
   - Via metadata on contacts that Apsis fetches?
   - Another mechanism?

2. **Can Tribe make the CRM ID visible to end users**, or at least provide a way for customers to reference contacts by their relation ID?

3. **What is the exact technical obstacle preventing Tribe from calling the merge endpoint?**
   - Is it an integration/authentication issue?
   - A logical/orchestration issue in knowing when to merge?
   - Lack of awareness of the capability?

[Lukasz Grabowski]: I don't know what's the obstacle on their side. They have great team, great product. But we should ask them on the next meeting because we'd rather...

### Follow-Up Meeting Scheduled

A technical discussion with Tribe developers is scheduled for the next day. Attendees from Apsis will include:
- **Lukasz Grabowski** (organizer, primary technical contact)
- **Speaker 1** / **Grzegorz Kozub** (likely architect or tech lead)
- **Henrik Boye** (optional but available)
- **Baptiste** (developer)
- **Tomasz Kowalski** (optional)
- **NOT Agneta Lindahl Nevell** (despite her deep knowledge; the team notes she tends to extend meetings and a technical-developer-only discussion is preferred)

[Speaker 1]: Yes, I think so. I don't think we need Agneta. As knowledgeable as she is, she has this talent to steal a lot of time from the meeting.

---

## Key Takeaways

1. **Root Cause:** Tribe's relation ID model, combined with lead-to-company assignment workflows, creates duplicate relation IDs in Tribe, which manifest as duplicate CRM IDs in Apsis.

2. **Core Data Model:**
   - Person ID (unique to individual, rarely used on Apsis side)
   - Relation ID (unique to person-company pair, used as Apsis CRM ID)
   - Apsis One has **no company awareness**; it only sees CRM IDs

3. **Three Scenarios Identified:**
   - Scenario 1 & 2: Lead → real company assignment. **Should merge.**
   - Scenario 3: Consultant multi-company. **Should NOT merge.** Legitimate duplicate emails.

4. **Proposed Solution:**
   - Tribe to drive merge decisions, not Apsis.
   - Use existing Apsis merge endpoint or metadata-driven approach.
   - Tribe to include merge signal (either via API call or metadata) when it identifies merge scenarios.
   - Maintains Tribe's control over business logic.

5. **Implementation Preference:**
   - **Minimize Apsis-side work**: Tribe should call merge endpoint if possible.
   - **Fallback to metadata approach** if Tribe has technical constraints.
   - Both require Tribe-side implementation effort.

6. **CRM ID Visibility Gap:**
   - CRM ID is not visible to Tribe customers; only visible to FSC (Apsis staff).
   - This is a UX problem that Tribe should address independently, but it complicates manual merge scenarios.

7. **Race Condition Handled:**
   - Tribe must ensure new profile is created (synced to Apsis) before calling merge endpoint.
   - Apsis returns new CRM ID immediately, so this should be feasible.

8. **Next Steps:**
   - Developer meeting with Tribe to discuss feasibility of merge endpoint approach.
   - Clarify Tribe's technical constraints and integration capabilities.
   - Finalize solution and effort estimates.

---

## Unresolved Questions

- **Does Tribe have the capability to call the Apsis merge endpoint**, or would they need Apsis to implement a webhook/callback mechanism?
- **Can Tribe feasibly add and maintain metadata on contacts** to communicate merge intent to Apsis?
- **Will Tribe make CRM IDs customer-visible**, or accept the UX limitation?
- **What is the exact failure mode Tribe encountered** when they attempted to implement merge automation on their side?

---

## Historical Context & Rationale

This duplication problem is not a bug but a consequence of deliberate design choices:

- **Tribe chose to use relation IDs** because B2B semantics require person-company role context, not just person identity.
- **Apsis chose to receive and store CRM IDs** without company awareness because Apsis One is not primarily a B2B-hierarchical system.
- **Apsis lead company** was introduced as a workaround for B2B leads that don't yet have a company assignment, but this created the duplication scenario when those leads are later qualified.

The proposed merge-driven solution respects both systems' architectural constraints while giving Tribe (the source of truth for B2B hierarchy) the control to decide when duplication is acceptable.