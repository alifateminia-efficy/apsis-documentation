---
source_file: Duplicate Apsis profiles when synced from Tribe contacts (1).txt
domain: Apsis One Integrations
topics: [Duplicate profile creation, CRM ID mapping, Contact relationships, Person vs Contact entity model, Lead to contact conversion workflow, Integration merge strategies]
speakers: [Agneta Lindahl Nevell, Lukasz Grabowski, Speaker 1 (Grzegorz), Henrik Boye, Tomasz Kowalski]
key_components: [Apsis One, Tribe CRM, Justin (integration layer), Profile merge endpoint, Lead forms, Webhook integration]
session_type: knowledge-transfer
subdomains: ["Duplicate profiles", "Lead creation"]
---

## Session Overview

This KT session addresses a critical bug in the Apsis One and Tribe integration where duplicate contact profiles are unintentionally created in Apsis One when contacts are moved between companies or when lead-qualified prospects are converted to real customers. The team walks through three distinct scenarios where duplication occurs, explores the root cause (the use of relation IDs rather than person IDs), and proposes a solution leveraging Apsis One's existing merge endpoint combined with metadata passed from Tribe during contact sync.

---

## The Duplicate Profile Problem: Three Scenarios

### Scenario 1: Lead Conversion with Company Assignment

When a lead is created through an Apsis form submission in Tribe's system, it's initially assigned to a temporary "Apsis Lead" company entity. A CRM ID is generated and synced back to Apsis One. Later, when the customer validates this lead and assigns it to a real company (moving the contact from "Apsis Lead" to the actual customer company), Tribe automatically creates a new relation and generates a new CRM ID.

**The Problem**: Apsis One receives two separate CRM IDs for the same person:
- The original CRM ID (from the lead created by the form)
- The new CRM ID (from the real company assignment)

This results in two distinct contact profiles in Apsis One, when ideally there should be one unified profile with full history. [Agneta Lindahl Nevell]: "We get duplicates. We don't want that. In that case we want it to be the same."

### Scenario 2: Email Correction or Company Correction

A customer fills out a form with an incorrect email address or realizes they selected the wrong company during the initial form submission. The customer updates the contact record in Tribe, moving it to the correct company. Tribe's automatic behavior creates another new relation ID, triggering the same duplication issue on the Apsis side.

[Agneta Lindahl Nevell]: "We don't want the duplication, we want it to be the same contact" because the user is not creating a new contact—they're correcting metadata on an existing one.

### Scenario 3: Legitimate Multiple Company Relationships (Consultant Use Case)

Some individuals legitimately have relationships with multiple companies. For example, a consultant might work on HR tasks for Company A and financial controller tasks for Company B. In this case, they should actually have multiple distinct profiles in Apsis One, each representing their role at a different company, because communication needs to be segmented by company context.

[Agneta Lindahl Nevell]: "Then we have basically what happens is that we have three CRM IDs for contact. And in those cases, we need to maybe merge the lead CRM ID with the correct company connected, but we need to keep one more instance of the same contact."

The critical distinction: Scenario 3 is intentional and desired; Scenarios 1 and 2 are bugs that should trigger merging.

---

## The Core Data Model: Person vs. Contact vs. Relation ID

### Understanding Tribe's Entity Hierarchy

Tribe CRM uses a three-level entity model:
- **Person**: A unique individual (identified by a person ID), containing universal attributes like name and social security number
- **Contact**: A relationship between a person and a company, identified by a relation ID (a UUID)
- **Company**: An organization in the CRM

A single person can have multiple contacts (one per company where they maintain a business relationship).

[Agneta Lindahl Nevell]: "So you have a person ID and you have a relation ID and the relation ID is the contact and that is what we're using on the Apsis side... a unique identifier for the relationship between the person and the company."

### Why Apsis One Uses Relation IDs, Not Person IDs

When the question was raised—"Why can't we just use person ID for CRM ID on our site?"—the answer revealed the architectural reasoning:

[Agneta Lindahl Nevell]: "If we choose person, then we can't work with the information that is on the contact because then you can only work with a person... My role, I'm a CEO, that's on the relationship, that's on the contact."

In B2B contexts, **you are your role**. The same person holds different roles at different companies. Apsis One needs to sync and work with role-specific data (title, company-specific communication preferences, etc.), which lives on the contact/relation entity, not the person entity. Therefore, the CRM ID must map to the relation ID, not the person ID.

[Agneta Lindahl Nevell]: "In the B2B world you are your role."

### Expected Duplication: The "Schizophrenia" Case

It is expected and correct that the same email address appears in multiple Apsis profiles if that person holds different roles at different companies. [Speaker 1]: "It's called schizophrenia... So this schizophrenia in Apsis 1 manifests in just two separate profiles with different CRM IDs."

This is not a bug—it's a feature. The duplicate profiles are intentionally separated so that campaign targeting and communication can be segmented by company context.

---

## Why Tribe Creates Duplicate Relations (It's Automatic Behavior)

The duplication problem stems from Tribe's automatic behavior when a contact's company is updated. The Tribe system maintains referential integrity by creating new relation IDs whenever a person's relationship to a company changes. 

[Speaker 1]: "It happens automatically. That's the thing... This is a core functionality similar to key spaces to us."

Tribe developers proposed several ideas for handling this, but the core issue remains: changing Tribe's internal architecture to prevent relation ID creation for certain scenarios would require significant product changes and is not feasible. 

[Lukasz Grabowski]: "They have this like main contact and then put all the references to this contact with new IDs. So we shouldn't rather ask them on the next meeting... if they can change this."

---

## The Visibility Problem: CRM IDs Are Not Customer-Visible

A frustrating complexity emerged: the CRM ID (the unique system UUID that identifies a relation) is not visible to Tribe customers. It's internal system metadata.

[Agneta Lindahl Nevell]: "If I want to find this contact, I can't... The unique identifier is the system UUID, but the customer cannot see this... If I create this contact and then I immediately want to find it, then of course I need to use [the CRM ID]... otherwise I have to wait for 20 minutes."

Tribe customers cannot immediately locate a newly synced contact in Apsis One using the CRM ID because:
1. They don't have access to the CRM ID
2. Searching by email alone is slow (up to 20 minutes, sometimes longer on Fridays) because duplicate profiles will appear

[Agneta Lindahl Nevell]: "If all I know is the e-mail address, it will take time and then I will find... I will see duplicate responses because I don't have the unique identifier."

This creates a poor user experience and adds urgency to solving the duplication issue.

---

## Proposed Solutions: Evolution from API Calls to Metadata-Driven Merging

### Initial Proposal: Tribe Calls Apsis Merge Webhook

The first idea was straightforward: Apsis One already has a profile merge endpoint. When Tribe detects that a contact should be merged (e.g., when assigning an Apsis Lead contact to a real company), it could simply call Apsis's merge webhook with both CRM IDs.

[Speaker 1]: "If Tribe is able to call this merge web hook endpoint, then it's easier for us. We don't need to do anything."

**Why This Faces Obstacles on Tribe's Side**:

Tribe developers identified a race condition: if Tribe calls the merge endpoint immediately after creating a new contact, the new contact may not yet be synced to Apsis One. A merge request for a profile that doesn't exist in Apsis yet would fail.

[Speaker 1]: "This new profile in track has not yet been synced... this merge endpoint returns a not found error message."

However, Lukasz noted that Tribe's typical flow is: Apsis sends a lead → Tribe receives it and immediately responds with a CRM ID. The sync back to Apsis happens quickly. The race condition may not be as severe as feared, but it remains a point of discussion.

[Lukasz Grabowski]: "How they organise orchestrate these calls to us because first they should create new contact and then they should call Webhook for merge."

### Preferred Proposal: Metadata-Driven Merging (Less Work on Apsis Side)

The team converged on a cleaner solution: rather than Tribe calling our merge endpoint, we extend the sync protocol to include metadata.

When Tribe syncs a contact to Apsis One, it will include an optional field—a list of related CRM IDs (specifically, the Apsis-lead-generated CRM IDs that should be merged with this contact).

**The Flow**:
1. Lead created from form → synced to Apsis with CRM ID_A
2. Customer assigns lead to real company → new contact created in Tribe with CRM ID_B
3. Tribe includes metadata on the sync: `"merge_with_crm_ids": [ID_A]`
4. Apsis Justin integration layer receives the sync for ID_B and sees the metadata
5. Justin automatically merges ID_B with ID_A, keeping ID_B as the primary key

[Speaker 1]: "When they sync because when you have Apsis lead in tribe and you assign a company to it right, another contact is created right... this new contact also has relation to contact generated from Apsis lead right... they need to either take the ID of that Apsis lead generated contact and put it somewhere in the metadata of that new contact."

**Advantage**: This approach requires zero changes to Apsis One's core merge logic. Tribe simply provides additional metadata that tells us which profiles should be merged. The merging happens via existing code paths. No new APIs or webhooks needed.

[Speaker 1]: "The less work on our side, the better."

**Control remains with Tribe**: If Tribe encounters a legitimate multi-company scenario (like the consultant working for two customers), they simply don't include the merge metadata, and Apsis One will keep both profiles separate.

[Speaker 1]: "It's in control of tribe... when they don't want to merge, they then don't assign this metadata."

---

## Technical Implementation Details

### The Justin Integration Layer

The merge functionality would be implemented in **Justin**, the integration orchestration layer. Justin already handles the syncing of contacts from Tribe to Apsis One.

[Lukasz Grabowski]: "I think we can use Justin for this. At least I can see some conversation with Eric that there is such functionality in Justin."

### Existing Merge Endpoint

Apsis One already has a merge endpoint that:
- Accepts keyspaces including the CRM keyspace
- Takes CRM IDs for both the source and target profiles
- Merges the profiles, preserving history
- Marks one as the primary contact

[Speaker 1]: "I think we already have profile merge end point in in Apsis one API. This accepts key spaces including CRM key space and I think the IDs for both of these profiles in the CRM key space."

The new work would be minimal: intercept the metadata during sync and trigger the merge call programmatically.

---

## History Preservation: A Key Requirement

A critical requirement throughout all scenarios is that history must be preserved. When a lead is converted to a customer contact, the merged profile should retain:
- The form submission event that created the lead
- Any nurture activities that occurred while it was a lead
- The transition point to customer
- All subsequent customer activities

[Agneta Lindahl Nevell]: "We want the history, we just don't want to contact Synapsis, we want it to be the same... as a marketeer, I want to be able to say, hey, look at all these leads I've generated that turned into customers and they generated this much revenue."

This is crucial for marketing attribution and CRM audit trails.

---

## Next Steps and Preparation for Tribe Meeting

The team was preparing for a follow-up meeting with Tribe developers to discuss this solution. The internal Apsis team needed to:

1. **Validate feasibility**: Confirm that Justin can support the metadata-driven merge pattern
2. **Clarify Tribe's constraints**: Understand exactly what obstacles Tribe faces in calling the merge webhook
3. **Propose the metadata solution**: Present the metadata approach as a lighter-weight alternative that puts control in Tribe's hands
4. **Document the use cases**: Ensure Tribe understands which scenarios require merging (Scenarios 1 & 2) and which don't (Scenario 3)

[Lukasz Grabowski]: "Let's be prepared to this one... So we have a proposition for them. I don't know what the problem is, but we will identify... Let's discuss it with them tomorrow."

### Meeting Attendees Decided

- Lukasz Grabowski
- Speaker 1 (Grzegorz)
- Henrik Boye (optional)
- Baptiste (Tribe-side developer)
- Tomasz Kowalski (noted but optional)

**Deliberately excluded**: Agneta Lindahl Nevell, despite her deep knowledge, because she "has this talent to steal a lot of time from the meeting" and the next session would be a technical discussion focused on implementation, not requirements gathering.

[Speaker 1]: "I don't think we need Agneta. As knowledgeable as she is... maybe we don't need her for technical discussion."

---

## Architectural Implications and Design Patterns

### Why This Is More Than a Bug Fix

This issue touches fundamental integration design questions:

1. **Control and Agency**: Should the source system (Tribe) or the target system (Apsis) decide when to merge? The team consensus is that Tribe, as the CRM of record and the system where business logic lives, should control this decision.

2. **Metadata Enrichment**: The solution introduces a pattern where integration metadata (the merge list) travels alongside entity data. This is a useful pattern for other similar scenarios.

3. **Eventual Consistency**: The system must handle timing—Tribe's create + sync + merge flow happens across multiple async operations. The metadata approach reduces coupling by letting Apsis One handle the timing via its own sync queue.

### B2B vs. B2C Data Models

The core reason for using relation IDs instead of person IDs reflects a broader B2B SaaS architecture principle: in B2B systems, identity is contextual. You're not just "a person"—you're "a person in the role of VP of Sales at Company X." This context-dependent identity must be maintained throughout the data model and integration layer.

---

## Caveats and Known Gotchas

1. **Sync Latency**: Apsis One syncs from Tribe have experienced delays ranging from 7 to 420 minutes. The merge must happen after the synced profile is confirmed in Apsis, not before.

2. **Non-Customer-Visible CRM IDs**: Tribe customers cannot see CRM IDs, which creates usability friction. Any solution must account for this asymmetry.

3. **Tribe's Core Functionality**: Creating new relation IDs for relationship changes is core to Tribe's data model and can't be easily disabled without breaking other functionality.

4. **Multi-Company Legitimacy**: The system must distinguish between "should merge" and "should keep separate" at the business logic level, not the technical level. This requires Tribe to encode the business intent in the metadata.

---

## Key Takeaways

1. **The Problem is Real but Solvable**: Duplicate profiles in Apsis One result from Tribe's automatic creation of new relation IDs when company assignments change. This is not a bug in Tribe—it's architectural. Apsis One must handle the merging on the receive side.

2. **Relation ID vs. Person ID is the Right Choice**: Using relation IDs (not person IDs) as CRM IDs is correct for B2B systems where role and company context matter. Don't change this.

3. **Three Scenarios, Two Need Merging**: 
   - Scenarios 1 & 2 (lead qualification, email correction) → merge profiles
   - Scenario 3 (legitimate multi-company role) → keep separate
   - Tribe must encode which scenario applies via metadata

4. **Metadata-Driven Merging is Preferred**: Rather than Tribe calling our merge webhook (which has race condition risks), have Tribe pass a list of CRM IDs to merge as sync metadata. Apsis Justin layer triggers the merge automatically.

5. **Control Should Live in Tribe**: The source system (Tribe) should decide whether to merge. Apsis should execute the merge if metadata is present.

6. **Preserve History**: Any merge must maintain full audit trail and activity history so marketing attribution still works.

7. **CRM ID Visibility is a UX Problem**: That customers can't see the unique identifier that would help them find synced contacts is silly. This should be addressed separately in Tribe product.

---

## Unresolved Questions & Action Items

1. **Does Justin support the metadata pattern?** Need to validate with Eric and Justin documentation that we can intercept custom metadata during contact sync and trigger merges.

2. **What exactly are Tribe's obstacles with the webhook approach?** The race condition theory needs confirmation. Could they buffer the merge call until after sync confirmation?

3. **Can Tribe easily extract and pass the Apsis-lead CRM ID when creating new contacts?** The metadata solution assumes Tribe can track which contact relation IDs originated from Apsis leads and reference them in new contacts.

4. **Should we expose CRM IDs to Tribe customers?** This is a separate product decision for Tribe, but it would improve the experience.

5. **How do we test this with actual Tribe data?** Will need sandbox environment with representative scenarios (lead conversion, email correction, consultant multi-company).

**Next Meeting**: Technical discussion with Tribe developers (Baptiste and team) to propose the metadata solution and work through implementation details.