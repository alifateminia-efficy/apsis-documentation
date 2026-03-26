---
source_file: "Duplicate Apsis profiles when synced from Tribe contacts (1).txt"
domain: Apsis One Integrations
topics: ["duplicate profiles", "CRM contact identity model", "lead qualification flow", "profile merge endpoint", "Tribe/CRM integration", "CRM ID visibility", "race conditions in merge orchestration"]
speakers: ["Agneta Lindahl Nevell (domain/product expert)", "Lukasz Grabowski", "Henrik Boye", "Tomasz Kowalski", "Speaker 1 (unidentified, likely technical lead)"]
key_components: ["Apsis One", "Tribe CRM", "Justin (internal service)", "CRM ID / contact UUID", "person ID", "relation ID", "profile merge endpoint", "webhook merge endpoint"]
session_type: knowledge-transfer
subdomains: ["Duplicate profiles", "Lead creation"]
---

## Session Overview

This session investigates a bug in which syncing Tribe CRM contacts to Apsis One results in duplicate profiles for the same individual. Agneta Lindahl Nevell walks through three concrete scenarios in which duplicates arise, rooted in the Tribe CRM data model where a single person can have multiple **relation IDs** (one per company relationship). The group discusses why Apsis One uses the relation/contact ID rather than the person ID, why this causes duplicates in certain lead qualification flows, and what the correct fix looks like. The agreed-upon direction is for Tribe to call Apsis One's existing **profile merge endpoint** (available via Justin) when it determines two profiles should be consolidated, rather than building merge logic on the Apsis side.

---

## Tribe CRM Data Model: Person ID vs. Relation/Contact ID

### Core Identity Concept in Tribe

In Tribe CRM, a **person** can hold multiple **roles** relative to multiple companies. The data model distinguishes:

- **Person ID**: the unique identifier for the individual human (analogous to a Social Security number). Stores personal-level data: first name, personal attributes, etc.
- **Relation ID / Contact ID**: a UUID representing the *relationship* between a person and a specific company. Stores role-specific data: job title (e.g. "CEO"), company affiliation, etc.

> "In the B2B world, you are your role." — Agneta Lindahl Nevell

A person with roles at two different companies will have **two contact/relation IDs**, while having one person ID.

### Why Apsis One Uses the Relation/Contact ID

An early question was raised: why not use the **person ID** as the CRM ID in Apsis One, which would naturally avoid duplicates for the same individual?

**Answer:** Using the person ID would lose access to relationship-level data. Apsis One cannot navigate the company–person hierarchy using person ID alone. Role-specific attributes (e.g. job title, company affiliation) live on the contact/relation entity, not the person entity. A deliberate decision was made to use the **contact/relation UUID** as the CRM ID in Apsis One.

**Consequence:** A consultant working for two companies legitimately produces **two Apsis profiles** with different CRM IDs but the same email address. This is *expected and correct behavior* for that scenario.

---

## Three Duplicate-Profile Scenarios

### Scenario 1: Lead Qualified and Moved to a Real Company

1. An Apsis One form submission creates a lead in Tribe CRM, attached to the fake placeholder company **"Apsis Lead"**.
2. Tribe syncs the CRM ID back to Apsis One; Apsis merges this into the email-keyed profile → the profile now has a CRM ID.
3. A Tribe user qualifies the lead and reassigns the contact to a **real company**. Tribe creates a **new relation ID** for the new company relationship.
4. This new relation/contact is synced to Apsis One, creating a **second profile**.
5. **Desired outcome:** The two profiles should be merged into one, preserving the history of the original lead (form fill, nurturing activities).

### Scenario 2: Wrong Email or Existing Contact Reassigned

1. A lead is created via form with incorrect email, or the contact is identified as someone already known.
2. The Tribe user updates the relation to an existing company.
3. Same result: two CRM IDs in Tribe → two profiles in Apsis One.
4. **Desired outcome:** Merge into one profile.

### Scenario 3: Legitimate Multi-Company Contact (No Merge Wanted)

1. An existing contact (e.g. a consultant) is added to a *second* company, producing a third CRM ID.
2. **Desired outcome:** Keep as separate profiles — the person genuinely represents two different roles for two different companies. *Do not merge.*

> "We don't want the duplication in scenarios 1 and 2. In scenario 3, we want the duplicate because this consultant filled in that they're also working for this company." — Agneta Lindahl Nevell

This means **the merge/no-merge decision cannot be made automatically by Apsis One** — only the Tribe user has the business context to know which scenario applies.

---

## CRM ID Visibility Problem

A significant complication: the **contact/relation UUID is not visible to Tribe customers** in the CRM UI. It is an internal system field.

> "I asked Sander and he was like, no, they cannot see it. It's not customer-visible data. And it's the unique key that we use." — Agneta Lindahl Nevell

**Impact:** Customers cannot easily look up or cross-reference profiles using the CRM ID. If only the email address is known, finding the correct profile in Apsis One can take time (sync lag of up to 20 minutes or more in some cases), and searching by email may return duplicates.

**⚠️ Ambiguity:** It is unclear whether Tribe *developers* (as opposed to end customers) have programmatic access to the contact UUID when constructing integration payloads. This was not fully resolved.

---

## Proposed Solution: Tribe Calls the Apsis One Merge Endpoint

### Existing Capability

Apsis One already has a **profile merge endpoint** (accessible via **Justin**, the internal integration layer). This endpoint:
- Accepts key spaces including the CRM key space
- Takes two CRM IDs and merges the corresponding profiles
- Was apparently discussed previously with Erik (referenced in a prior conversation)

> "I think we can use Justin for this. I can see some conversation with Eric that there is such functionality in Justin." — Lukasz Grabowski

### Proposed Flow

1. Tribe user qualifies a lead → Tribe creates a new contact/relation entity with a new CRM ID.
2. Tribe's own logic determines whether this is a "merge" scenario (Scenario 1/2) or a "new relationship" scenario (Scenario 3).
3. If merge is appropriate: **Tribe calls the Apsis One/Justin webhook merge endpoint**, passing both the new CRM ID and the original Apsis Lead–generated CRM ID.
4. Apsis One merges the two profiles, using the new CRM ID as the primary key, while preserving all historical data from the lead profile.
5. If merge is not appropriate (Scenario 3): Tribe simply does not call the merge endpoint. Two profiles remain in Apsis One.

### Why Merge Logic Should Live in Tribe, Not Apsis

- Apsis One has no knowledge of company associations or the business context of why a new CRM ID was created.
- The merge/no-merge decision is entirely determined by Tribe-side business logic (e.g., "was the original contact created from Apsis Lead company?").
- Building this logic in Apsis One would be fragile and would require Apsis to understand Tribe's internal model.

### Alternative Considered and Rejected: Metadata List on New Contact

Speaker 1 proposed an alternative: when Tribe syncs a newly qualified contact, it could include a **custom metadata field** on the contact containing a list of Apsis-Lead-generated CRM IDs to be merged. Justin would detect this field during sync and trigger the merge automatically.

**Reason rejected (or deprioritized):** This would require changes on the Apsis/Justin side and would only apply to the Tribe connector, making it a connector-specific hack rather than a general solution. The simpler approach — Tribe calls the merge webhook — requires no new work on Apsis's side.

> "The less work on our side, the better." — Speaker 1 / Lukasz Grabowski

---

## Race Condition Concern

Speaker 1 raised a potential **race condition**:

> "If I have this Apsis Lead profile and I create a new profile in Tribe, and I know I want them merged, so I call our merge endpoint — but this new profile in Tribe has not yet been synced to Apsis. The merge endpoint returns a 'not found' error."

**Lukasz's response:** This is likely not a significant issue in practice, because when Apsis sends a lead to Tribe, Tribe **immediately responds with the CRM ID** (synchronous API call). The profile is already registered in the Apsis system by the time Tribe could act on it. The merge call would come after, and the profile should already exist.

**⚠️ This was not fully validated** — it was acknowledged as a known concern but considered low risk based on the current API interaction pattern.

---

## Open Questions for Tribe Developer Meeting (Scheduled for the Next Day)

The group agreed to bring a concrete proposal to a **technical meeting with Tribe developers** the following day. Key questions to resolve with Tribe:

1. **Why haven't they implemented the merge webhook call yet?** They apparently have the idea but don't know how to execute it. What is the actual obstacle on their side?
2. **Can Tribe reliably distinguish** between Scenario 1/2 (merge desired) and Scenario 3 (separate profiles desired) using their existing business logic? Proposed heuristic from Agneta: if origin is "Apsis Lead" and a real company is being assigned *for the first time*, auto-merge; otherwise don't.
3. **Can Tribe call the Apsis One/Justin merge webhook?** Is this endpoint exposed and accessible in the context of the Tribe integration?

**Attendees confirmed for the meeting:** Speaker 1, Henrik Boye, Baptiste (Tribe developer). Tomasz optional. Agneta not invited to keep the meeting focused and technical.

---

## Key Takeaways

1. **Duplicate profiles in this context are sometimes correct** (multi-company consultant) and sometimes a bug (lead qualified to a real company). The distinction is a business-logic decision that only Tribe can make.

2. **Root cause:** Tribe CRM creates a new relation/contact UUID every time a person is associated with a company. Apsis One uses this relation UUID as the CRM ID, so each new relation = a new Apsis profile.

3. **The reason Apsis uses relation ID (not person ID)** is that role/company-specific attributes live on the relation entity, not the person entity. This was a deliberate architectural decision.

4. **The fix should be Tribe-driven:** Tribe calls the existing Apsis One merge endpoint (via Justin) when it determines two profiles represent the same logical contact. No new Apsis-side logic needed.

5. **The CRM ID is invisible to Tribe customers**, which complicates any customer-facing merge UX and makes direct profile lookup by CRM ID impossible for end users.

6. **A merge endpoint already exists in Justin** — this is not a new feature request, it is a matter of Tribe integrating against an existing capability.

---

## Unresolved Questions / Action Items

- [ ] **Confirm with Tribe developers** what obstacle is preventing them from calling the merge webhook, and validate the proposed solution in tomorrow's technical meeting.
- [ ] **Validate race condition risk** more carefully: confirm that by the time Tribe would call the merge endpoint, the new contact is guaranteed to already exist in Apsis One.
- [ ] **Clarify Tribe developer access to contact UUID** — can they programmatically read the internal contact/relation UUID to pass to the merge endpoint?
- [ ] **Define the merge trigger heuristic** precisely: Agneta suggested "if origin = Apsis Lead AND this is the first real company assignment → auto-merge." This needs to be agreed upon and documented.
- [ ] **Confirm Justin merge endpoint contract** — what exact parameters does it accept (key space, CRM ID format), and is it accessible from Tribe's integration layer?