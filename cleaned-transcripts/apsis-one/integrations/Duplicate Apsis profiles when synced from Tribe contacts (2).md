---
source_file: "Duplicate Apsis profiles when synced from Tribe contacts (2).txt"
domain: Apsis One Integrations
topics: [duplicate profile creation, Tribe-Apsis sync, contact person vs person entity, profile merging, race conditions in async sync, generic connector architecture, webhook payload design]
speakers: ["Baptiste Lapeyre (Tribe/connector developer)", "Lukasz Grabowski (Apsis developer)", "Michal Rosikiewicz (Apsis developer)", "Tomasz Kowalski (Apsis developer)", "Speaker 1 (Apsis, likely Greg or senior technical lead)", "Henrik Boye (Product)"]
key_components: [Tribe CRM, Apsis One, Apsis connector, generic connector, contact person entity, person entity, Apsis Lead company, profile merge API, webhook/sync endpoint]
session_type: knowledge-transfer
---

## Session Overview

This session investigates a bug where syncing Tribe CRM contacts to Apsis One creates duplicate profiles. The root cause lies in Tribe's data model: a physical **Person** entity must always be linked to a **Contact Person** entity, which in turn must be linked to a company. When a lead arrives from Apsis, a temporary "Apsis Lead" company and contact person are created in Tribe; when a Tribe user later associates that person with a real company, a new contact person is created — and it is the contact person (not the person) that is synced to Apsis, causing a duplicate profile. The group agrees on a solution: Tribe's connector should include the original contact person's ID in the sync payload under a new field (e.g., `original_contact`), allowing Apsis to create the new profile and then merge it with the original lead profile. It is also noted that a potentially simpler path may already exist in the generic connector (an existing flag mechanism used for other enterprise integrations), which needs internal verification before a final proposal is sent to Tribe.

---

## Tribe Data Model: Person vs. Contact Person

### Entity Hierarchy

In Tribe CRM, there are two distinct entity types relevant to this issue:

- **Person**: Represents a physical individual.
- **Contact Person**: Represents the link between a Person and a Company. A person cannot exist in Tribe without a contact person, and a contact person cannot exist without being linked to a company.

This means one physical person can have multiple contact person records — one per company they are associated with.

### Why Contact Person is Synced to Apsis (Not Person)

The current integration syncs the **contact person** entity to Apsis, not the person entity directly. This was in place before Baptiste joined; the rationale (attributed to product stakeholder Agneta) is that contact person carries more valuable data — specifically, information about an individual *in the context of a company*. The person entity alone lacks some of this information (e.g., there are separate email fields on each entity).

> [Speaker 1]: "The data of an individual in context of a company is more valuable than the data of the individual alone. That's why a contact person was chosen to sync down to Apsis."

[Lukasz Grabowski]: Agneta confirmed that "they lack some information on the person level entity."

**Conclusion reached in session**: Switching the sync to use the person entity instead of the contact person is ruled out. The contact person remains the synced entity.

---

## Root Cause of Duplicate Apsis Profiles

### The Flow That Creates Duplicates

1. A lead is submitted via an Apsis form.
2. The Apsis connector creates a **Person** and a **Contact Person** in Tribe, linked to a placeholder **"Apsis Lead" company** (a company created specifically to hold inbound Apsis leads).
3. Apsis receives a CRM ID back from Tribe for this contact person and updates the lead profile with that CRM ID.
4. A Tribe user later associates that person with their real company. Tribe creates a **new Contact Person** linked to the real company.
5. This new contact person triggers a sync notification to Apsis, which creates a **new, separate profile** in Apsis.
6. Result: two Apsis profiles exist for the same physical person — one from the original Apsis Lead company contact person, one from the real company contact person.

Baptiste demonstrated this with a test contact ("D1.2 / T1.2") that had been associated with 16 different companies, resulting in **16 contact persons in Tribe** but only **1 person entity**, and correspondingly **16 duplicate profiles in Apsis**.

### Why a Simple Synchronous Merge Call Doesn't Work

The initial idea of having Tribe call a merge API on Apsis immediately after creating the new contact person was ruled out due to a **race condition / async issue**:

- When a new contact person is created in Tribe, Tribe only sends a notification (webhook) to Apsis that something with a given ID has changed.
- Apsis then makes a separate request back to Tribe to fetch the contact data and creates the profile.
- At the moment Tribe's webhook fires, the new profile does not yet exist in Apsis — so a merge call at that point would fail or operate on a non-existent target.

> [Baptiste]: "It's almost impossible for us to call the merge on your side after that because we are not sure [when Apsis has finished creating the profile]."

> [Speaker 1]: "There is a race condition here."

---

## Business Requirements: When to Merge vs. When Not to Merge

### The Complexity

Not all new contact persons should trigger a merge. A person can legitimately work for two companies simultaneously, in which case two separate Apsis profiles are the correct outcome and should **not** be merged.

> [Henrik Boye, Product]: "One person can have two different roles in two different companies, and then it should be two profiles in Apsis — there should not be a merge between those."

### Agreed Business Rule

Merge should be triggered **only when** the new contact person is the first real-company association for a person who was previously only linked to the **Apsis Lead** placeholder company. In other words:

- **Trigger merge**: Person was in Tribe only under "Apsis Lead" company → now being linked to their first real company.
- **Do not trigger merge**: Person already has a contact person linked to a real company (i.e., they work for multiple companies legitimately).

[Baptiste] confirmed he can implement this check on the Tribe side: inspect whether the existing contact persons for that person are only linked to Apsis Lead companies. If so, include the merge signal. If a real-company contact person already exists, do not include it.

---

## Proposed Technical Solution: Extend Sync Payload with `original_contact` ID

### Mechanism

Instead of a separate merge API call (which has the async/race condition problem), the solution is to include the original contact person's ID **in the sync payload itself** — i.e., in the response Tribe returns when Apsis fetches the contact data after the webhook notification.

Flow with the fix:
1. New contact person created in Tribe (real company association).
2. Tribe sends webhook notification to Apsis (existing behaviour).
3. Apsis calls Tribe to fetch the contact data (existing behaviour).
4. **NEW**: Tribe includes an additional field in the response payload — e.g., `original_contact` — containing the CRM ID of the original Apsis Lead contact person, **but only** when the business rule above is met.
5. Apsis creates the new profile from the payload as normal.
6. **NEW**: If `original_contact` is present in the payload, Apsis looks up the profile identified by that ID and merges the two profiles.

> [Speaker 1]: "If one of these objects has `k_original_lead` or something like that, we will first create this profile using that CRM ID — which we already do — and then look up the other one and merge it to that profile."

### Why This Avoids the Race Condition

Because the merge signal is embedded in the same API response that delivers the new contact's data, Apsis has already created the new profile by the time it attempts the merge. There is no race condition.

> [Speaker 1]: "This makes it easier for us. We don't need to store some state on our side between the webhook and this call."

### Handling Idempotency

[Speaker 1] noted that if this payload field is sent multiple times for the same pair (e.g., due to retries), the merge operation is idempotent — merging an already-merged pair is safe and will simply be a no-op.

> [Speaker 1]: "Merge is idempotent. If we get this new property multiple times and we attempt to merge, this should be fine because the merge will just say 'already merged, we are fine.'"

Baptiste confirmed he will try to avoid sending it more than once as a performance best practice, but it is not a correctness concern.

### Field Naming

The exact field name was not finalized in the session. Suggested in discussion: `original_contact`. The Apsis team will prepare a specification and share it with Baptiste before implementation.

---

## Generic Connector Architecture Concern

### Risk of Breaking Other Integrations

[Michal Rosikiewicz] raised a significant concern: the connector is built as a **generic connector** shared across multiple enterprise CRM integrations. Any changes to the sync payload contract must not break other integrations.

> [Michal]: "This is part of the generic connector and we'll have to change it just for Tribe, but we have to be sure that this won't affect other integrations using the generic connector — and that might be challenging."

[Lukasz Grabowski] confirmed this: "It was built as generic. This API is for all integrations, so we need to think about how to fit it for all. We need to be very careful not to break anything for other integrations."

[Speaker 1] suggested that since well-behaved API clients ignore unknown properties, adding a new optional field to the payload should be safe, but this needs verification.

### Possible Existing Solution (Flag Mechanism)

Late in the session, Michal suggested that the generic connector may **already have** a mechanism for handling this type of merge scenario, used for other enterprise systems — possibly just requiring a configuration flag to be enabled for Tribe.

> [Michal]: "We already have it in the generic connector for other enterprise systems. We just need to enable the flag for Tribe and verify if everything works correctly."

> [Tomasz]: "Like for events, you did."

[Speaker 1] and Lukasz were not fully aware of this existing capability and deferred: "I need to understand it. But it seems like this is solved — we will come back to you with our proposal."

⚠️ **Ambiguity**: It is unclear whether this flag mechanism fully covers the described solution or requires additional development. This was flagged as needing internal investigation before a proposal is sent to Baptiste.

---

## Webhook Architecture Notes

### Current Tribe → Apsis Webhook Behaviour

- When a contact person is created or updated in Tribe, Tribe sends a notification to Apsis with the entity ID.
- Apsis then makes a separate request (pull) to Tribe to fetch the full contact data.
- Apsis creates or updates the profile based on whether the CRM ID already exists on their side.
- Tribe does **not** currently send any merge request in this flow.

[Tomasz Kowalski] noted that webhooks for syncing "congestions" (possibly: contact changes?) for Tribe are not currently enabled — this may be a relevant configuration detail. ⚠️ *[Transcript was unclear here; this should be verified with Tomasz.]*

### Alternative: Include Merge Signal in Webhook Notification Itself

[Speaker 1] also floated the possibility of including the `original_contact` ID in the webhook notification rather than in the subsequent data-fetch response. Baptiste confirmed this is technically feasible from the Tribe side. The Apsis team will evaluate both options (webhook vs. data-fetch response) when preparing the specification.

---

## Key Takeaways

1. **Root cause**: Tribe's data model requires a new contact person per company. Since Apsis syncs contact persons (not persons), each new company association creates a new Apsis profile — resulting in duplicates.
2. **Switching to sync the Person entity is ruled out** — contact person carries business-critical data (company context, separate email fields) that would be lost.
3. **A synchronous merge call from Tribe is not viable** due to async/race conditions in the current webhook-pull architecture.
4. **Agreed solution**: Tribe's connector includes an `original_contact` field in the data-fetch response payload when the business rule is met (first real-company association for an Apsis-lead-originated person). Apsis uses this to merge the new profile with the original lead profile after creation.
5. **Business rule for triggering merge**: Only when the person's only existing contact person is linked to the "Apsis Lead" placeholder company. All other cases (multi-company legitimate associations) do not trigger merge.
6. **Generic connector risk is real** — changes must be scoped carefully to not break other CRM integrations using the same connector.
7. **Possible shortcut**: An existing flag mechanism in the generic connector (used for other enterprise systems) may already implement the required merge behaviour — internal verification needed before any new development begins.
8. The merge operation in Apsis is **idempotent** — safe to call multiple times for the same profile pair.

---

## Unresolved Questions and Action Items

| # | Item | Owner | Notes |
|---|------|-------|-------|
| 1 | Investigate whether the existing generic connector flag mechanism covers this use case | Michal / Tomasz (Apsis) | May eliminate significant development work |
| 2 | Prepare formal API specification / epic for the `original_contact` payload field | Lukasz Grabowski (Apsis) | Includes field naming, payload format, webhook vs. data-fetch response decision |
| 3 | Groom the epic with the Apsis team and align on delivery timeline | Lukasz + Henrik (Apsis/Product) | Timeline not yet determined |
| 4 | Share final proposal/specification with Baptiste for Tribe-side implementation confirmation | Apsis → Baptiste Lapeyre | Baptiste confirmed both webhook and data-fetch payload approaches are feasible on Tribe side |
| 5 | Henrik to align with Tommy and Agneta (product) to confirm no edge-case business scenarios are missed | Henrik Boye | Specifically: confirm the merge-only-on-first-real-company rule covers all product requirements |
| 6 | Produce flow diagram of the full Tribe → Apsis sync sequence | Tomasz / Michal (Apsis) | Currently no visual documentation exists; noted as helpful for resolving internal disagreement on async issue |
| 7 | Clarify whether Tribe webhooks for contact-change events are currently enabled/disabled for the Tribe integration | Tomasz Kowalski | Mentioned briefly but not resolved |