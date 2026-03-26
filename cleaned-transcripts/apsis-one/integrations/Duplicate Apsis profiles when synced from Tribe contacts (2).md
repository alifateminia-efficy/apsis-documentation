---
source_file: Duplicate Apsis profiles when synced from Tribe contacts (2).txt
domain: Apsis One Integrations
topics: ["Duplicate profile creation from Tribe syncs", "Contact person vs Person entity modeling", "Merge strategy for duplicate profiles", "Webhook async race conditions", "Generic connector architecture considerations"]
speakers: ["Baptiste Lapeyre (Tribe)", "Speaker 1 (Apsis, likely Greg or product lead)", "Lukasz Grabowski (Apsis)", "Michal Rosikiewicz (Apsis)", "Tomasz Kowalski (Apsis)", "Henrik Boye (Product)", "Henrik Boye (Product manager)"]
key_components: ["Tribe CRM", "Apsis One", "Generic connector", "Webhooks", "Profile merge APIs", "Contact person entity", "Person entity", "Apsis Lead company (fake account)"]
session_type: architecture-review
subdomains: ["Duplicate profiles"]
---

## Session Overview

This KT session addresses a critical bug where duplicate profiles are created in Apsis One when contacts are synced from Tribe and subsequently associated with multiple companies. The core issue stems from Tribe's data model requiring both a Person entity and a Contact Person entity (the latter linked to a company), while Apsis One currently syncs the Contact Person instead of the Person. When a single person is linked to multiple companies in Tribe, each company association creates a new Contact Person, resulting in multiple duplicate profiles in Apsis. The team discussed the root causes, evaluated potential solutions considering async race conditions, and settled on a hybrid approach: Tribe will send profile merge metadata in the API payload during contact creation, allowing Apsis to intelligently merge duplicate profiles when appropriate while respecting business scenarios where duplicates should remain separate.

---

## Tribe Data Model and Root Cause Analysis

### Contact Person vs Person Entity Architecture

[Baptiste Lapeyre]: The core structural issue is in how Tribe models contacts. Tribe has two entity types that interact in a constrained way:

- **Person**: Represents a physical individual
- **Contact Person**: Links a person to a specific company

In Tribe's system, a person cannot exist without at least one contact person, and a contact person must always be linked to a company. This is a hard requirement in the system design.

When Apsis One sends a lead form submission to Tribe via the connector:
1. Both a Person and a Contact Person are created simultaneously
2. The Contact Person links the person to a synthetic "Apsis Lead" company (a fake account created to receive the lead)

### The Duplication Scenario

[Baptiste Lapeyre]: The duplication occurs when a user reassigns the person to a different company:

1. Original sync from Apsis → creates Person T1.2 + Contact Person linked to "Apsis Lead" company
2. User in Tribe associates Person T1.2 with Company A → creates new Contact Person linked to Company A
3. User associates Person T1.2 with Company B → creates another Contact Person linked to Company B
4. And so on for all 16 companies in the test case

The result: one Person, but 16 Contact Person entities, each synced back to Apsis as a separate profile.

[Baptiste Lapeyre]: In my testing, I created one lead (D1.2) and then associated it with multiple companies. The result was one person named T1.2 but 16 contact persons linked to different companies. Currently, Apsis syncs the contact person, not the person itself.

### Why Not Sync Person Instead?

The team discussed switching the sync source from Contact Person to Person, but rejected this approach:

[Speaker 1]: The decision to sync contact person instead of person was driven by business requirements. The data of an individual in the context of a company is more valuable than the data of an individual alone. Contact Person carries contextual data that Person entity lacks.

[Lukasz Grabowski]: Apsis lacks some information at the person level entity. Contact Person entities have additional data fields.

[Baptiste Lapeyre]: There are multiple email fields, for example. Some are stored on Person, others on Contact Person. We can't change this without losing valuable context.

**Decision**: Maintain the current Contact Person sync approach. This means the solution must handle merging at the Apsis side, not by changing Tribe's fundamental architecture.

---

## The Async Race Condition Problem

### Webhook Flow and Timing Issues

[Baptiste Lapeyre]: When Tribe creates a new contact person, here's what happens:

1. Tribe calls the Apsis webhook to notify of a change
2. Apsis triggers a request back to Tribe to fetch the full contact data
3. Tribe responds with the data, including the new CRM ID

The challenge: the webhook notification is asynchronous. Apsis receives the notification and creates a profile, but the timing is not synchronous.

[Lukasz Grabowski]: So you call our webhook and then Apsis sends a request to get the contact and receives the information later.

[Baptiste Lapeyre]: Exactly. It's async and almost impossible for us to call the merge operation on your side after that, because we are not sure when you have finished creating the profile.

[Speaker 1]: There is a race condition here. When we originally discussed this yesterday, the simplest solution—a synchronous merge call from Tribe after profile creation—won't work due to this async nature.

**Consensus**: The synchronous merge approach is off the table because of unpredictable timing between the webhook notification and profile creation completion.

---

## Business Logic: When Should Profiles Merge?

### Scenario Analysis

[Speaker 1]: There are multiple business scenarios. In some cases, when a new contact person is created in Tribe, it should result in a profile that is merged to the original lead. In other cases, the new contact person should sync as a completely separate profile.

[Henrik Boye]: One person can have two different roles in two different companies. In that case, there should be two profiles in Apsis and they should NOT be merged.

[Baptiste Lapeyre]: The first time a contact is linked to a new company, we want to merge. But if it's linked to multiple companies beyond that, those should remain as separate profiles because the person has different roles.

### Decision Logic

[Speaker 1]: I think we've reached consensus: Tribe should send the merge metadata only when creating the first contact person for a company that is not the "Apsis Lead" company.

[Baptiste Lapeyre]: I will send the original contact ID only when:
1. This is the first duplicate contact person being created
2. It's linked to an Apsis Lead company (the synthetic company created when the lead was synced from Apsis)

If for some reason an old contact was changed to another company, or if it's linked to any other company beyond the first new link, I won't send the merge ID.

**Business Rule**: Merge profiles only for the first company association after the initial Apsis Lead company. This preserves the ability to have multiple non-merged profiles for different company roles while preventing unintended duplication from the initial lead sync.

---

## Proposed Technical Solution

### Two Viable Approaches

The team identified two potential solutions:

#### Option 1: Include Metadata in Webhook Notification

[Baptiste Lapeyre]: When Tribe calls the webhook to notify of a new contact person, it could include the original contact ID in the same notification. Apsis could extract this ID and initiate the merge as part of the profile creation flow.

**Drawback**: Still subject to async timing issues if merge happens via separate API call.

#### Option 2: Include Metadata in Fetch Response Payload (Preferred)

[Baptiste Lapeyre]: When Apsis makes a request to fetch contact data after the webhook notification, Tribe includes the original contact ID in the response payload.

The flow:
1. Tribe webhook: "Contact person X was created"
2. Apsis requests: "GET /contact/X"
3. Tribe response: Returns full contact data PLUS `_original_lead_contact_id: <UUID>`
4. Apsis creates the profile AND checks for the merge ID
5. If merge ID is present, Apsis looks up the original profile and merges it
6. If no merge ID, profiles remain separate

[Speaker 1]: I like this better because we don't need to store state on our side between the webhook and this call. All the information comes in one response.

[Lukasz Grabowski]: We extend the response to return the original contact ID, then we just handle this in the code and do the merge after creating the profile.

### Merge API Idempotency

[Speaker 1]: If we receive this merge ID multiple times by mistake, it's fine. Merging the same pair of profiles once results in them being already merged. Attempting to merge again won't cause harm—it's idempotent.

[Baptiste Lapeyre]: I should try to avoid sending it multiple times for performance reasons anyway.

---

## Implementation Considerations

### Payload Structure

[Speaker 1]: The original contact ID should be a GUID (business-wise contact person identifier). In the response, if an object has a property like `_original_lead_contact_id`, Apsis will know to merge this new profile with the existing profile identified by that ID.

Example response payload structure:
```
{
  "contact_person_id": "new-uuid-12345",
  "person_id": "person-uuid-abc",
  "email": "john@example.com",
  "phone": "+1234567890",
  "_original_lead_contact_id": "original-uuid-67890",
  ...other fields...
}
```

### Generic Connector Compatibility

[Michal Rosikiewicz]: This endpoint is part of the generic connector used by all enterprise systems integrated with Apsis One. We need to be careful not to break other integrations that use this same API.

[Lukasz Grabowski]: We can't just add this for Tribe alone. The generic API is shared across all integrations, so we need to think about how to fit this for all of them without breaking existing integrations.

[Speaker 1]: There are ways to handle this. HTTP headers, additional properties in the JSON object—generic connectors should ignore unexpected properties. If they don't, we can work around it.

**Mitigration strategy**: Add the new `_original_lead_contact_id` property as optional metadata. Systems that don't use it will ignore it; systems that do will process it.

### API Response vs Webhook Approach

[Tomasz Kowalski]: For other systems using the generic connector, we can verify whether this behavior should be enabled via a flag or feature toggle.

[Michal Rosikiewicz]: We already have this merge mechanism in place for other enterprise systems using the generic connector. It might be a matter of enabling a configuration flag rather than building entirely new functionality.

[Speaker 1]: But we need to understand what we already have and what we need to build. Lukasz will create full documentation to clarify this.

---

## Decision and Action Items

### Agreed Solution Summary

1. **Tribe responsibility**: When creating a new Contact Person that originated from an Apsis Lead, include the original Contact Person ID as metadata
2. **Apsis responsibility**: When receiving contact data with the `_original_lead_contact_id` field, create the profile normally, then merge it with the original profile identified by that ID
3. **Scope**: This applies only to the first company association after the Apsis Lead company
4. **Idempotency**: The merge operation is idempotent; duplicate merge requests won't cause issues

### Next Steps

[Lukasz Grabowski]: I will:
1. Create comprehensive documentation for the epic, stories, and technical specifications
2. Identify whether this can leverage existing merge mechanisms in the generic connector or requires new development
3. Schedule a grooming session with the team to review the technical proposal
4. Sync with Baptiste on implementation details
5. Prepare a delivery timeline

[Henrik Boye]: Will:
1. Align with product team (Tommy and Agneta) to ensure all use cases are covered
2. Confirm that the merge strategy matches business requirements

[Baptiste Lapeyre]: Will:
1. Implement the metadata payload changes on Tribe's side
2. Coordinate with Apsis team once specifications are finalized
3. Indicate whether webhook or fetch response payload is preferred from Tribe's implementation perspective

**Contact protocol**: 
- For Apsis side product questions: contact Tommy (product)
- For technical implementation: Baptiste can be reached directly

---

## Technical Uncertainties and Follow-ups

### Async Handling in Webhook Queue

There was discussion about whether the webhook approach could work if all calls are queued (FIFO - first in, first out). Some team members questioned whether other systems handle this scenario successfully.

[Lukasz Grabowski]: We need flow charts or sequence diagrams to fully understand the timing and whether async is truly a blocker for a pure webhook merge approach.

[Tomasz Kowalski]: We could sync with Erik to create visualizations of the sync flow.

**Resolution**: This will be analyzed during the grooming/documentation phase. The fetch response approach was chosen as the preferred solution for now due to its simplicity in avoiding state management.

### Generic Connector Flag vs New Development

[Michal Rosikiewicz]: The merge mechanism might already exist as a feature flag in the generic connector code used by other enterprise systems.

[Speaker 1]: This requires clarification. If it's already present, it's a configuration matter. If not, development is required.

**Action**: Lukasz to investigate the existing codebase and determine the scope of work needed.

---

## Key Takeaways

1. **Root cause**: Tribe's data model requires Contact Person entities (one per company association), each syncing to Apsis as a separate profile. Unlike switching to Person-level sync, this is a business requirement driven by valuable contextual data stored at the Contact Person level.

2. **Async timing is critical**: Tribe cannot reliably call a merge API after profile creation due to async webhook handling. The solution must include merge metadata in the data response itself.

3. **Smart merging required**: Not all Contact Person creations should trigger merges. Only the first company association after initial lead sync should merge. Later associations are intentional duplicates for different roles.

4. **Payload-based solution preferred**: Tribe sends the original Contact Person ID as a property in the fetch response payload. Apsis interprets this and merges profiles after creation. This eliminates state management and timing issues.

5. **Generic connector implications**: The change affects the shared generic connector API, requiring careful testing to ensure backward compatibility with other enterprise integrations.

6. **Product alignment needed**: Henrik will confirm with Agneta (Tribe product) and Tommy (Apsis product) that the merge scenarios match the intended business logic.

---

## Unresolved Questions and Next Steps

1. **Exact configuration flag behavior**: Does the generic connector already support conditional merge logic via feature flags, or is new development required?
2. **Flow diagram validation**: Need sequence diagrams to confirm whether pure webhook-based merge could work with proper queueing.
3. **Backward compatibility testing**: How will we ensure other enterprise integrations using the generic connector aren't affected?
4. **Delivery timeline**: Dependent on investigation into existing code and complexity assessment.

**Expected follow-up**: Lukasz will provide specifications and timeline within the coming days; team will reconvene during grooming session.