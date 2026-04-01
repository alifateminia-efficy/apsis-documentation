---
source_file: "Duplicate Apsis profiles when synced from Tribe contacts (1).txt"
domain: Apsis One Integrations
topics: [duplicate profile handling, CRM ID management, Tribe-Apsis sync, profile merge, lead qualification workflow, B2B contact model]
speakers: ["Agneta Lindahl Nevell (Product/Domain Expert)", "Lukasz Grabowski (Engineering Lead)", "Speaker 1 (Developer/Architect, unidentified)", "Henrik Boye", "Tomasz Kowalski"]
key_components: [Apsis One, Tribe CRM, Justin (internal service), Apsis Lead, CRM ID / contact relationship ID, profile merge endpoint, Tribe connector, webhook/merge endpoint]
session_type: knowledge-transfer
---

# Session Overview

This session covers a complex data integrity problem: when contacts in the Tribe CRM are qualified (i.e., an "Apsis Lead" is assigned to a real company), a new CRM relationship ID is generated in Tribe, which causes a duplicate profile to be created in Apsis One instead of updating the existing lead profile. Agneta Lindahl Nevell walked through three concrete business scenarios that trigger this duplication. The group analyzed the root cause (Tribe's data model using relationship IDs rather than person IDs as the unique contact identifier), evaluated solution approaches, and converged on a proposal: Tribe should call Apsis One's existing merge/webhook endpoint when a merge is desired, rather than building new logic on the Apsis side. A follow-up technical meeting with the Tribe development team (Baptiste) was scheduled for the next day.

---

## Tribe CRM Data Model: Person ID vs. Contact (Relationship) ID

### Core Concept
In Tribe CRM (a B2B CRM), a **person** (individual human) is distinct from a **contact** (the relationship between a person and a company). The unique identifier used by Apsis One in its CRM key space is the **contact/relationship ID** — a UUID representing the role a person holds at a specific company — not the person's own ID.

- **Person ID**: Identifies the individual (e.g., equivalent to a Social Security number or personal record). Holds person-level attributes (first name, personal data).
- **Contact / Relationship ID**: A UUID identifying the person's role at a specific company (e.g., "CEO at Company A"). Holds role-level attributes. This is what Apsis One uses as the CRM ID.

> "In the B2C world you're the person, but in the B2B world you are your role." — Agneta Lindahl Nevell

### Why Not Use Person ID?
This was raised as a naive/exploratory question. The reason person ID was rejected: if you use the person ID, you can only access attributes stored on the person entity, not the contact/relationship entity. Role-specific data (job title, company association) lives on the relationship, not the person. The decision was made to track contacts (relationships) because that is what the CRM manages and what carries the actionable marketing attributes.

### Multi-Company Consultants: Intentional Duplicates
A person can have multiple contacts (relationship IDs) in Tribe — one per company they are associated with. This is expected and intentional. For example, a consultant doing HR work for Company A and financial work for Company B has **two contact IDs** and should have **two separate Apsis One profiles**, both sharing the same email address.

> "I would be two contacts with the same email address, but I'm actually 2 contacts. I'm the same person, but I'm two contacts." — Agneta Lindahl Nevell

This is considered correct behavior and is by design. Apsis One accommodates this: two separate profiles with different CRM IDs but the same email address can coexist.

---

## The Duplicate Profile Problem: Three Scenarios

### How Apsis Lead Works (Background)
When a contact fills in an Apsis form, a **lead** is automatically created in Tribe CRM under a placeholder entity called **"Apsis Lead"** (a fake/made-up company used as a holding entity). Tribe assigns a CRM ID to this lead and syncs it back to Apsis One, creating a profile keyed by that CRM ID (an email-keyspace contact promoted to a CRM-ID contact).

### Scenario 1: Lead Assigned to Real Company (Same Person, Wrong Initial Company)
1. Contact fills in form → lead created in Tribe under "Apsis Lead" placeholder → CRM ID #1 synced to Apsis One.
2. Tribe user recognizes the contact and assigns them to a real company.
3. Tribe creates a new relationship ID (CRM ID #2) for the person's role at the real company.
4. CRM ID #2 is synced to Apsis One → **duplicate profile created**.
5. **Desired behavior**: The two profiles should be merged into one. History of the lead (form fill, nurturing activities) should be preserved. The new CRM ID should become the primary key.

### Scenario 2: Lead Assigned to Correct Company (Wrong Email or New Contact, Same Company)
1. Lead created from form → Tribe user identifies it as an existing contact or recognizes the email was wrong.
2. Tribe user updates the relationship to an existing real company.
3. Same outcome as Scenario 1: a new CRM ID is issued, a second Apsis One profile is created.
4. **Desired behavior**: Merge into one profile. The contact was just moved to the correct company, not duplicated.

### Scenario 3: Consultant with Multiple Companies (Intentional — Do NOT Merge)
1. A consultant already exists in Tribe as a contact for Company A.
2. They fill in a form → lead created under "Apsis Lead" → CRM ID synced to Apsis One.
3. Tribe user recognizes this is the same consultant and adds a relation to Company B.
4. This results in **three CRM IDs**: original contact at Company A, Apsis Lead entry, and new contact at Company B.
5. **Desired behavior**: Keep separate profiles. The consultant legitimately works for multiple companies and should have distinct profiles per relationship. The "Apsis Lead" CRM ID may need to be merged with the correct company-connected CRM ID, but the multi-company instances should remain separate.

> "Then we want the duplicate scenario because now this consultant filled in that he's also working for this company." — Agneta Lindahl Nevell

### Key Insight: The Decision to Merge Belongs to the Tribe User
The determination of whether profiles should be merged is a **business/user decision made in Tribe**, not something Apsis One can infer automatically. Only the Tribe user knows whether a new contact is a duplicate that should be merged or a legitimate separate relationship.

---

## CRM ID Visibility Problem (Compounding Issue)

The **contact/relationship UUID** (CRM ID used by Apsis One) is **not visible to Tribe customers** in the Tribe UI. Only FSC/internal users can see it. This was confirmed by "Sander."

> "The unique identifier is the system UUID, but the customer cannot see this." — Agneta Lindahl Nevell
> "I asked Sander and he was like, no, they cannot see it. It's not customer-visible data." — Agneta Lindahl Nevell

This creates a practical problem: a Tribe customer who wants to find or reference a specific Apsis One profile immediately after creation cannot use the CRM ID to do so. They would have to search by email address and wait for the sync (could be up to 20 minutes, sometimes longer — noted as 420 minutes on a specific Friday) before the profile appears.

This compounds the duplicate problem because without the unique identifier being accessible, customers cannot easily detect or manually resolve duplicates.

---

## Proposed Solution: Tribe Calls Apsis One Merge Endpoint

### Existing Infrastructure
Apsis One already has a **profile merge endpoint** (described also as a webhook endpoint). This endpoint:
- Accepts key spaces including the CRM key space
- Accepts the IDs of both profiles to be merged

[⚠️ Ambiguous: The merge endpoint is referred to interchangeably as a "merge endpoint," "webhook," and "Justin" functionality. Lukasz referenced a prior conversation with "Eric" about merge functionality in **Justin** (an internal service). The exact endpoint URL, API spec, or service name was not specified in this session.]

### Preferred Solution
**Tribe calls Apsis One's existing merge webhook/endpoint** when a Tribe user performs a qualifying action that should result in a merge. This requires:

1. **Tribe side**: Tribe developers implement logic to detect when a qualifying action (e.g., assigning an Apsis Lead contact to a real company for the first time) should trigger a merge, and call the Apsis One merge endpoint with the two CRM IDs.
2. **Apsis side**: No new work required beyond confirming the endpoint is accessible to Tribe's integration context.

The "if-then-else" logic on Tribe's side:
- If the action is "assign Apsis Lead contact to a real company (first time qualification)" → call merge endpoint
- If the action is "add consultant to a new company (additional relationship)" → do NOT call merge endpoint

> "If they have a scenario, business scenario that is recognised in tribe as booking [...] then they don't call the merge endpoint. If they have a scenario that is recognised in tribe as assigning to a company, then call the merge endpoint as well. What is the problem with this logic?" — Speaker 1

### Alternative Considered: Metadata Field on New Contact (Rejected as Heavier)
Speaker 1 proposed an alternative where Tribe extends the contact entity with a metadata field containing a list of source CRM IDs (Apsis Lead-generated IDs). When Justin/the Tribe connector syncs the new contact, it would detect this list and automatically merge the profiles.

This was considered but rejected as the **preferred** approach because:
- It requires work on the Apsis/Justin connector side
- It would be Tribe-connector-specific logic (not generalizable)
- The merge endpoint approach is simpler and keeps control on the Tribe side

### Race Condition Concern (Partially Resolved)
Speaker 1 raised a potential race condition: if Tribe calls the merge endpoint for a newly created contact before that contact has been synced to Apsis One, the merge endpoint would return a "not found" error.

Lukasz's counter-argument: When Apsis One syncs a lead to Tribe, Tribe **immediately responds with a CRM ID** via an online/synchronous API call. This means the profile in Apsis One is created and the CRM ID is known before Tribe would need to call the merge endpoint. Therefore, by the time Tribe triggers the merge action (user qualification), both profiles should already exist in Apsis One.

[⚠️ This resolution was accepted but not fully stress-tested in the discussion. The exact timing/ordering of the sync calls in edge cases was not fully mapped out.]

---

## Tribe's Reported Obstacle with Merge Approach

Tribe developers had previously been informed about the merge endpoint and expressed uncertainty about how to implement the triggering logic on their side — specifically, they did not know how to reliably identify the connection between the new contact and the original Apsis Lead-generated contact at the point of merge triggering.

> "They in tribe do not know how to deal with it because they don't know the connection." — Lukasz Grabowski

The FSC team's position is that this is a Tribe-side concern and should be solvable with standard if-then-else logic by Tribe's developers. The upcoming meeting with Tribe's developer (Baptiste) is intended to clarify the obstacle and align on the solution.

---

## Marketing/Business Rationale for Preserving History

The reason merge (rather than delete-and-replace) is required: marketers need the full history of a contact's journey — from form fill, through lead nurturing, through qualification — to measure pipeline attribution and ROI.

> "As a marketer, I want to be able to say, hey, look at all these leads I've generated that turned into customers and they generated this much revenue. So yes, we want the history, we just don't want two contacts in Apsis, we want it to be the same." — Agneta Lindahl Nevell

Post-merge, a single Apsis One profile should carry:
- The original form submission activity
- Nurturing activities from the lead phase
- New activities as a qualified contact

---

## Key Takeaways

1. **Root cause of duplicates**: Tribe CRM issues a new relationship/contact UUID whenever a person is associated with a new company. Apsis One uses this UUID as the CRM ID, so any new company assignment creates a new Apsis profile rather than updating the existing one.

2. **The merge decision is always user-initiated in Tribe** — Apsis One cannot and should not try to infer whether profiles should be merged; this is a business-context decision only the Tribe user can make.

3. **Intentional duplicates exist and must be preserved** — Multi-company consultants legitimately have multiple Apsis profiles with the same email. The deduplication logic must be selective, not blanket.

4. **Preferred solution is lowest-Apsis-work**: Tribe calls the existing Apsis One merge endpoint/webhook when a qualifying merge event occurs. No new development on the Apsis side is required if this approach is adopted.

5. **The CRM ID is not customer-visible in Tribe UI** — this is a known issue flagged by the team as problematic, as it limits the ability for Tribe users to directly reference or locate Apsis profiles immediately post-creation.

6. **Justin** (internal service) has merge functionality that is relevant here; there was a prior conversation with "Eric" about this. This should be consulted for implementation details.

7. **Sync latency is variable** — nominally 7 minutes but can reach 20+ minutes and has reached 420 minutes in observed cases. This is relevant for any solution that depends on a profile being present in Apsis One before a merge is triggered.

---

## Unresolved Questions and Action Items

- **[Action — FSC/Speaker 1, Henrik Boye]**: Meeting with Tribe developers (Baptiste) scheduled for the next day to present the merge endpoint proposal and understand what obstacle Tribe is facing in implementing the trigger logic.
- **[Action — Lukasz / team]**: Confirm whether the Apsis One merge endpoint is accessible within the context of the Tribe integration (i.e., is it exposed via the One API or only internally via Justin).
- **[Question — unresolved]**: Can Tribe be asked to stop generating new CRM IDs on company re-assignment, and instead reuse existing ones? This was raised as a question for the next meeting but expected answer is "no" as it was characterized as core Tribe functionality.
- **[Question — unresolved]**: What exactly is the obstacle Tribe developers face in detecting when a merge should be triggered? This is the central question for the next-day meeting.
- **[Ambiguity]**: The exact name, URL, or API spec of the "merge endpoint" / "Justin merge functionality" was not specified. This needs to be confirmed and shared with Tribe developers.
- **[Follow-up consideration]**: The three-CRM-ID scenario (Scenario 3 with consultants) may require partial merge logic (merge Apsis Lead ID with correct company ID, but keep the third). The exact handling of this edge case was not fully resolved.