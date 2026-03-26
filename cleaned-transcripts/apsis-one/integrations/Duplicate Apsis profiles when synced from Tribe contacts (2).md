---
source_file: Duplicate Apsis profiles when synced from Tribe contacts (2).txt
domain: Apsis One Integrations
topics: [Profile deduplication, Tribe-Apsis sync flow, Contact person vs person entities, Merge logic, Async race conditions, Webhook architecture]
speakers: [Baptiste Lapeyre (Tribe), Speaker 1 (Apsis, moderator), Lukasz Grabowski (Apsis), Michal Rosikiewicz (Apsis), Tomasz Kowalski (Apsis), Henrik Boye (Product), "Greg" (mentioned but minimal participation)]
key_components: [Tribe connector, Apsis API, Webhook endpoints, Generic connector, Profile merge mechanism, CRM sync]
session_type: architecture-review
subdomains: [Integration Architecture, Data Synchronization, API Design]
---

## Session Overview

This meeting addressed a critical issue where duplicate profiles are created in Apsis when contacts from Tribe are synced after being associated with multiple companies. The root cause lies in Tribe's data model (where a person must be linked to a company via a contact person entity) combined with Apsis syncing the contact person rather than the base person entity. The team identified that when a person in Tribe gets linked to a new company, a new contact person is created and synced to Apsis as a separate profile, leading to duplicates. The proposed solution involves Tribe sending metadata about the original contact person ID alongside new contact person syncs, allowing Apsis to automatically merge the new profile with the original lead profile in specific business scenarios.

---

## Understanding the Tribe Data Model and Root Cause

### Tribe's Entity Structure

Baptiste explained Tribe's foundational constraint: **a person entity cannot exist without an associated contact person, and a contact person must be linked to a company**. This creates an inherent coupling in the data model.

> In Tribe we have a person that is representing a physical person usually. And then we have a link to a company which is a contact or contact person. In our case it's a contact person and the contact person is what links the person to a company. In Tribe we cannot have a person without having a contact person and we cannot have a contact person that is not linked to a company.

### How Leads Are Initially Created

When a lead is created from an Apsis form submission via the Apsis connector:

1. A **person** entity is created in Tribe
2. A **contact person** entity is created simultaneously, linking that person to a default "Apsis lead" company

Both entities are created in a single operation, but this results in two separate entities in Tribe's system.

### The Duplication Problem

When a user then associates the person with their actual company (rather than the default Apsis lead company):

1. A **new contact person** is created in Tribe, linking the same person to the correct company
2. The original person entity remains unchanged
3. **Only the contact person is synced back to Apsis** (not the person)
4. This new contact person appears as a completely separate profile in Apsis

Baptiste demonstrated this with test data:
> I created one [lead] and then I tried to affect it to a lot of company... I have a person that is named T1.2, but we have 16 contact person and they are linked to different company, although Apsis lead are different company named Apsis lead. So I have 16 person but only 1... 16 contact person but only one person.

### Why Contact Person Is Synced (Not Person)

Initially, the solution seems straightforward: sync the base person instead of the contact person. However, this approach was rejected for business reasons:

**Agneta (product owner at Tribe, referenced by Lukasz and Speaker 1) determined that contact person contains more valuable information than person alone**, specifically:
- Multiple email addresses per contact person (different for each company association)
- Company-in-context data that is essential for the business use case

> The data of an individual in context of a company is more valuable than the data of individual alone. That's why a contact person was chosen to sync down to Apsis.

Therefore, syncing contact person rather than person is the correct business decision, and the solution must accommodate this design choice.

---

## Current Sync Flow and Race Conditions

### How the Webhook/Sync Currently Works

[Michal Rosikiewicz]: When a new contact person is created in Tribe, the flow is:

1. Tribe sends a **webhook notification** to Apsis saying "there is a change on ID X"
2. Apsis does **not** receive the full contact data in the webhook
3. Apsis makes a **separate HTTP request back to Tribe** to fetch the contact details
4. Tribe returns the contact data in the response

This asynchronous request-response pattern creates a critical race condition.

### The Async Problem with Merge

[Baptiste]: They initially attempted to call a merge operation synchronously during the webhook notification, but this fails due to the async nature:

> When we create a new contact person because we linked to a new company, we just notify Apsis that there is some change on something with this ID. Apsis sends a request to get the contact and we provide the information in later, so it's odd and almost impossible for us to call after that the merge on your side because we are not sure [when the data creation is complete].

[Speaker 1]: Confirmed the race condition:
> There is a race condition basically here... So another proposal here, maybe.

The synchronous merge API call that already exists in Apsis cannot be used in this scenario because Tribe cannot reliably know when Apsis has finished creating the profile from the webhook data.

---

## Business Requirements: When Should Merge Occur?

### The Distinction Between Merge and Duplication Cases

A key insight emerged: **not all new contact person creations should result in merges**. There are two distinct business scenarios:

[Henrik Boye]: 
> One person can have two different roles in two different companies and then it should be two profiles in Apsis and then there should not be a merge between those.

[Speaker 1] and [Michal Rosikiewicz] clarified the requirement:
- **First contact person linked to a new company**: This should merge with the original lead profile (the person now has the "real" company association)
- **Subsequent associations**: If the same person is linked to additional companies after the first real association, those should NOT merge — they represent legitimate multiple roles

[Baptiste] refined this further:
> I will check if there is one other contact person that is linked to an Apsis lead company which was created when the lead was synced from Apsis. So in this case if for some reason there is an old contact that has been changed to some company, it will not affect that and if there is another contact that is linked to other any other company then I will not send you the ID of the original contact.

**The implementation rule: Tribe should send the original contact person ID only when:**
1. A new contact person is being created, AND
2. There exists exactly one other contact person for the same person that is linked to the "Apsis lead" company (indicating this is the first real company association)

---

## Proposed Technical Solution

### Overall Architecture

The agreed solution has these steps:

1. When Tribe creates a new contact person for a person (linking them to a real company):
   - Check if there's an existing contact person linked to "Apsis lead" company
   - If yes, include metadata in the sync payload to Apsis
2. Apsis receives the new contact person sync (via existing webhook mechanism)
3. Apsis fetches the contact data from Tribe (existing behavior)
4. **New step**: Apsis receives additional metadata indicating the original contact person ID
5. Apsis creates the new profile first
6. Apsis **then merges** this new profile with the original lead profile (which was created initially)

### Payload Modification

The solution leverages the existing Tribe API endpoint that Apsis calls to fetch contact data. Currently, when Apsis makes a GET request to retrieve contact details, it receives a payload with contact information. 

[Baptiste] proposed:
> When we create a new contact person in Tribe we send you a notification and then you send this request to this URL with all the IDs you need and we send the data in the response to this request. So I think we need to add it here so you can create the new contact with the data we send you. And then if in this payload varies the original contact ID or I don't know how we can call it, then you can merge it after that.

The response payload should include an optional field (e.g., `k_original_lead` or `original_contact_person_id`) containing the GUID of the original contact person:

```
{
  "k_contact": "UUID_of_new_contact",
  "k_original_lead": "UUID_of_original_contact",
  ... other contact fields ...
}
```

[Speaker 1]: 
> We will first create this profile using that CRM ID here K contact which we already do and then we actually look up the other one and merge it to that profile. Simple as that.

### Idempotency and Repeated Merge Attempts

One concern raised: what if Apsis receives the `k_original_lead` field multiple times for the same pair?

[Speaker 1]:
> If we merge the same pair one time, they're already merged, right? ... We cannot merge the same pair. If we merge the same pair one time, they're already merged, right? It's like merge in the Christian way, right? You can't unbind this.

The merge operation is idempotent — merging an already-merged pair is harmless. However, it's preferable to avoid sending duplicate original contact IDs for performance reasons.

[Baptiste]:
> I think I can avoid that. Should be possible. So yeah, I will try.

---

## Alternative Solution: Webhook Payload Approach

During discussion, an alternative emerged: **include the original contact ID directly in the webhook notification** rather than in the subsequent data fetch response.

[Speaker 1]:
> Would you be able to also send us this original contact ID in the webhook when you when you notice that you are creating a contact person that originated from Apsis lead?

[Baptiste]:
> Yeah, I really think so, yeah.

This approach would allow Apsis to merge synchronously (or near-synchronously) without waiting for the data fetch cycle. However, the team decided to move forward with the **data fetch response approach** because:

1. It avoids the async/race condition issues
2. The original solution was already identified as sound
3. Both approaches were acknowledged as viable

---

## Implementation Concerns: Generic Connector Impact

### Challenge: Reusability Across Multiple Integrations

[Michal Rosikiewicz] flagged a critical concern:
> This is part of Tribe connector and we'll have to change it just for this Tribe, but we have to be sure that this won't affect other generic connector using integrations and that might be challenging.

[Lukasz Grabowski] confirmed:
> It is actually challenging because it was built as generic. So this API is for all integrations, so we need to think how to fit it for all.

The API endpoint that receives the contact data is shared across **all integrations** using the generic connector, not just Tribe. Any changes must not break existing integrations.

### Proposed Mitigation

[Speaker 1]:
> There is ways to go about it. I think we have HTTP headers. We have this Jason object here. Will if there is more properties than expected, it will just ignore those properties. Hopefully generic connectors behave this way.

If the response payload includes additional fields like `k_original_lead`, the expectation is that:
- Integrations that understand it (Tribe) will use it
- Integrations that don't recognize it will ignore the extra field (forward compatibility)

If this assumption doesn't hold, alternative approaches exist:
- HTTP header flags (e.g., `X-Merge-Contact-Id`)
- Conditional logic based on integration type
- Feature flags specific to Tribe

---

## Next Steps and Ownership

### Specification and Epic Creation

[Lukasz Grabowski] will own the technical specification:
> I will create documentation for this epic stories, etcetera. But then let's sit together and try to groom it and look at this webhook and see what we can do and how. Then we'll sync again with Baptiste.

[Tomasz Kowalski] noted that flow diagrams would be helpful:
> We can sync with Eric. Maybe about this one if we need to would be good point.

### Product Alignment

[Henrik Boye] committed to aligning with Tribe's product team:
> I will align with Tommy and Agneta, yes, so we don't miss any use case.

Contact point for Tribe product requests: **Tommy** (exact name not fully captured, appears to be a Tribe product owner)

### Implementation Timeline

Timeline is **TBD** pending:
1. Specification completion
2. Grooming and review of the webhook architecture
3. Planning discussion between Henrik (Apsis product) and Lukasz (Apsis engineering)

[Lukasz]:
> I will take care of writing down everything we need and planning and then we should talk Henrik how to and when to plan it. Not sure about the delivery time yet.

### Tribe Implementation

[Baptiste] indicated the Tribe-side changes should be relatively quick:
> I think it would be quite quick on our side to do something like that, whether it's in the notification or in the payload after. I think both can be done on our side.

Next interaction: **Apsis will propose the specification, and Baptiste will confirm feasibility**.

---

## Key Takeaways

1. **Root Cause Identified**: Duplicate profiles occur because Tribe's data model requires a contact person per company association, Apsis syncs contact persons (not persons), and the sync mechanism treats each contact person as a new profile.

2. **Business Decision Confirmed**: Syncing contact person (not person) is correct because contact person contains company-contextual data (e.g., role-specific emails) essential to the business.

3. **Merge Strategy Defined**: Apsis should automatically merge the new profile with the original lead profile only when a person is first linked to a real company (after the default Apsis lead company). Subsequent associations to other companies should remain as separate profiles (legitimate multiple roles).

4. **Metadata Solution Chosen**: Tribe will send an optional `k_original_lead` field (or similar) in the contact data response payload when creating a new contact person under the merge conditions. Apsis will use this to:
   - Create the new profile
   - Look up the original lead profile (identified by the provided ID)
   - Execute an automatic merge

5. **Race Condition Resolved**: By including the merge metadata in the data fetch response (rather than the initial webhook), Apsis can confidently merge after creating the profile, eliminating async uncertainties.

6. **Generic Connector Compatibility**: Implementation must ensure backward compatibility with other integrations using the generic connector. Extra fields in responses should be ignored by systems that don't recognize them.

7. **Ownership Clear**: 
   - **Apsis**: Lukasz (spec), Michal (architecture), Tomasz (flow diagrams)
   - **Tribe**: Baptiste (implementation of metadata in payload)
   - **Product Alignment**: Henrik with Tribe product (Tommy and Agneta)

8. **Implementation Status**: Design complete, specification phase pending. Timeline TBD pending grooming and planning discussions.

---

## Unresolved Questions & Action Items

### Questions for Follow-up

1. **Exact field naming**: What will the optional metadata field be called? (tentatively `k_original_lead`, `original_contact_person_id`, or via HTTP header?)
2. **Webhook payload alternative**: Should the webhook notification also include this metadata, or only the data fetch response?
3. **Flow diagrams**: Would sequence diagrams clarify the async flow for all parties?
4. **Generic connector forward compatibility**: What is the actual behavior of existing integrations when encountering unexpected fields in JSON responses?

### Action Items

| Owner | Action | Status |
|-------|--------|--------|
| Lukasz Grabowski | Create specification for profile merge feature (epic + stories) | Pending |
| Lukasz Grabowski | Coordinate with Henrik on planning timeline | Pending |
| Tomasz Kowalski | Create flow diagrams showing current webhook/sync architecture | Pending |
| Michal Rosikiewicz | Analyze generic connector compatibility and propose safeguards | Pending |
| Baptiste Lapeyre | Implement metadata field in Tribe payload (awaiting spec approval) | Blocked on spec |
| Henrik Boye | Align with Tommy and Agneta on business scenarios and product rules | Pending |
| Apsis team | Present specification proposal to Baptiste for feasibility confirmation | Pending |

---

## Domain-Specific Terminology Clarified

- **Person**: Tribe entity representing a unique individual
- **Contact Person**: Tribe entity linking a person to a specific company; multiple contact persons can represent the same person in different company contexts
- **Apsis Lead Company**: The default/placeholder company created in Tribe when a lead is synced from Apsis (to satisfy Tribe's requirement that contact persons must link to a company)
- **Profile** (in Apsis context): A lead or contact record in Apsis; multiple profiles can refer to the same person
- **Merge** (in Apsis context): Combining two profiles into one; the merge operation is idempotent and irreversible
- **CRM ID**: The unique identifier assigned by Tribe to a contact person; used by Apsis to track and reference contacts