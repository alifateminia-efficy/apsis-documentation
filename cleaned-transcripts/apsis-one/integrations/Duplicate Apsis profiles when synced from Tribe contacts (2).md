---
source_file: "Duplicate Apsis profiles when synced from Tribe contacts (2).txt"
domain: Apsis One Integrations
topics:
  - Duplicate Apsis profiles caused by Tribe contact person syncing
  - Tribe data model (Person vs Contact Person vs Company)
  - Lead creation and sync flow from Apsis to Tribe
  - Proposed merge solution using original contact ID metadata
  - Race condition in async webhook flow
  - Generic connector compatibility concerns
speakers:
  - "Baptiste Lapeyre (Tribe connector developer)"
  - "Lukasz Grabowski (Apsis developer)"
  - "Michal Rosikiewicz (Apsis developer)"
  - "Tomasz Kowalski (Apsis developer)"
  - "Henrik Boye (Product, Apsis)"
  - "Speaker 1 (Apsis, likely senior developer or tech lead — unidentified)"
key_components:
  - Tribe CRM
  - Apsis One
  - Generic Connector
  - Webhook / sync endpoint
  - Profile merge API
  - Apsis Lead company (placeholder company in Tribe)
session_type: knowledge-transfer
subdomains: ["Duplicate profiles", "Lead creation"]
---

## Session Overview

This session investigates a bug where Apsis One creates duplicate profiles whenever a Tribe CRM contact person is associated with a new company. The root cause lies in the Tribe data model: each time a **Person** (physical individual) is linked to a new company, a new **Contact Person** entity is created in Tribe, and the Apsis connector syncs each Contact Person as a separate Apsis profile. The group works through several proposed solutions — including a synchronous merge call, sending lists of related contact IDs, and embedding merge metadata in the existing sync payload — ultimately converging on a solution where Tribe includes an `original_contact` ID in the sync response payload when a new Contact Person originates from an Apsis Lead. An important discovery is made late in the session: the Apsis generic connector may already support this merge mechanism via a flag, requiring only enablement for Tribe rather than new development.

---

## Tribe Data Model: Person, Contact Person, and Company

Understanding the Tribe data model is prerequisite to understanding the bug.

- A **Person** represents a physical individual.
- A **Contact Person** links a Person to a Company. It is the entity that carries the individual's data *in the context of a specific company*.
- In Tribe, a Person cannot exist without a Contact Person, and a Contact Person must be linked to a Company.
- A Person can be linked to multiple companies, which means multiple Contact Person records can exist for the same physical individual.

[Baptiste Lapeyre]: The connector is configured to sync **Contact Person**, not Person. This was done before Baptiste joined. The reason, as understood from Agneta (Tribe product owner), is that Contact Person holds more valuable data — specifically, it carries information about an individual in the context of their company, including fields (e.g. email) that are only populated at the Contact Person level and not at the Person level.

> "I think she mentioned that the contact person is more valuable and we shouldn't switch to person." — Speaker 1, recounting a prior conversation with Agneta.

Switching from syncing Contact Person to syncing Person was discussed and ruled out by the team based on product guidance.

---

## Root Cause of the Duplicate Profile Bug

### Lead Creation Flow (Normal Path)

1. A lead is submitted via an Apsis One form.
2. The Apsis connector creates both a **Person** and a **Contact Person** in Tribe simultaneously. The Contact Person is linked to a placeholder company called **"Apsis Lead"** (a company automatically created in Tribe when a lead is received from Apsis).
3. Apsis receives the Tribe-assigned CRM ID back and updates the lead profile with that ID.

### Duplication Trigger

When a Tribe user later associates that Person with their actual company:
- Tribe creates a **new Contact Person** linked to the real company.
- The connector notifies Apsis that something changed (via webhook with the new Contact Person ID).
- Apsis fetches the data and creates a **new, separate profile** — because it has no awareness that this Contact Person belongs to the same physical person as the original lead profile.
- The original profile (linked to "Apsis Lead" company) remains, resulting in a **duplicate**.

[Baptiste Lapeyre, illustrating with test data]: One person named "T1.2" had **16 Contact Persons** in Tribe (linked to 16 different companies), resulting in 16 separate profiles in Apsis — all for the same physical individual.

---

## Why a Synchronous Merge Call Does Not Work

The simplest solution — having Tribe call the Apsis merge API immediately after creating the new Contact Person — was discussed and rejected due to an **async race condition**:

1. Tribe creates the new Contact Person and immediately fires a webhook notification to Apsis (containing only the new Contact Person's ID).
2. Apsis then makes a separate outbound request back to Tribe to fetch the full contact data, and creates the new profile asynchronously.
3. At the time Tribe would need to call the merge API, Apsis has not yet created the new profile — so there is nothing to merge yet.

[Michal Rosikiewicz]: "It's not that they have this match event sent to us. It's about sending a new contact person to us."

This async gap makes it impossible for Tribe to reliably trigger a merge immediately after Contact Person creation.

---

## Business Scenarios: When to Merge vs. When Not to Merge

This is a critical nuance that prevents a blanket "always merge" solution.

- **Scenario A (merge required):** A lead comes in via Apsis form → gets parked under "Apsis Lead" company in Tribe → a Tribe user associates that person with their real company. In this case, the new Contact Person should be merged with the original lead profile in Apsis.
- **Scenario B (no merge — keep separate profiles):** A person is genuinely employed by two companies simultaneously. Each Contact Person represents a distinct business relationship and should remain a **separate profile** in Apsis.

[Henrik Boye, product]: "One person can have two different roles in two different companies and then it should be two profiles in Apsis and there should not be a merge between those."

The system currently never merges (causing duplicates in Scenario A). Always merging would break Scenario B. The solution must be selective.

---

## Agreed Technical Solution: Embed Original Contact ID in Sync Payload

### Mechanism

The agreed approach is to embed a metadata field — tentatively named something like `original_contact` — in the **sync response payload** that Tribe returns when Apsis calls back to fetch contact data.

The flow becomes:
1. New Contact Person is created in Tribe.
2. Tribe sends webhook notification to Apsis.
3. Apsis calls back to Tribe to fetch the full contact data (existing flow, unchanged).
4. **New:** If this Contact Person originated from an Apsis Lead and this is the first time it is being associated with a real company, Tribe includes the **CRM ID of the original lead Contact Person** in the response payload (e.g. as a field `original_contact` or `original_lead`).
5. Apsis creates the new profile using the returned data (existing behaviour).
6. **New:** If `original_contact` is present in the payload, Apsis looks up the profile identified by that CRM ID and **merges** the newly created profile into it.

[Speaker 1]: "If one of these objects will have `original_contact` or something like that, we will first create this profile using that CRM ID, and then we actually look up the other one and merge it to that profile."

### Why This Resolves the Race Condition

Because the merge is triggered by Apsis itself, in-process, as part of handling the sync response — not by an external Tribe call — there is no race condition. The profile exists by the time the merge is attempted.

### Baptiste's Proposed Logic for When to Include the Field

Baptiste outlined the specific condition under which Tribe will include `original_contact`:

- Check whether the Person has **exactly one other Contact Person** that is linked to the **"Apsis Lead" company**.
- If yes → this is the Scenario A (first real-company association after coming from Apsis) → include `original_contact` in the payload.
- If the Person is already linked to any other non-Apsis-Lead company → this is Scenario B (genuine multi-company person) → do **not** include `original_contact` → no merge.

[Baptiste Lapeyre]: "I won't send it for the first time it's duplicated — I will check if there is one other contact person that is linked to an Apsis Lead company which was created when the lead was synced from Apsis."

### Idempotency of Merge

[Speaker 1]: If the same `original_contact` ID is sent multiple times (e.g. due to repeated sync events), merging the same pair twice is safe — the system will recognise they are already merged and no harm is done. However, Baptiste confirmed he expects to send this field only once.

---

## Generic Connector Compatibility Concern

[Michal Rosikiewicz] raised a critical implementation concern: the sync endpoint in question is part of the **generic connector**, shared by all CRM integrations — not Tribe-specific. Any change to this contract must not break other integrations using the generic connector.

[Lukasz Grabowski]: "It was built as generic. So this API is for all integrations. We need to think how to fit it for all and we need to be very careful to not break anything for other integrations."

[Speaker 1] suggested that because the new field would be an **additional** JSON property in the response, well-behaved generic connector implementations would ignore unknown fields. This needs verification.

### Possible Existing Solution (Late Discovery)

Late in the session, Michal indicated that the generic connector may **already support a merge mechanism** that just needs to be **enabled via a flag for Tribe** — similar to how it was done for events. This would make the solution simpler than building new functionality.

[Michal Rosikiewicz]: "We already have it in generic connector for other enterprise systems that are using generic connector."

[Speaker 1]: "I need to understand it. But seems like this is solved... we will probably come back to you with our proposal."

⚠️ **Ambiguity:** It was not fully confirmed by end of session whether the existing flag-based mechanism fully covers the described solution or whether new development is still needed. Lukasz agreed to investigate and document before reconnecting with Baptiste.

---

## Webhook Format Discussion

A brief discussion occurred about whether the `original_contact` ID could alternatively be sent in the **webhook notification** (the initial call Tribe makes to Apsis) rather than in the sync response payload.

Both options were acknowledged as feasible on Tribe's side. The group leaned toward the **sync response payload** approach as it avoids needing to store intermediate state between the webhook receipt and the subsequent data-fetch call.

---

## Next Steps and Action Items

- **Lukasz Grabowski (Apsis):** Create epic and stories documenting the solution; investigate whether the existing generic connector merge flag covers this use case; prepare a specification/proposal and share with Baptiste.
- **Baptiste Lapeyre (Tribe):** Await Apsis's specification; implement the `original_contact` field in Tribe's sync response payload under the agreed business logic condition. Baptiste confirmed this should be quick to implement on the Tribe side.
- **Henrik Boye (Apsis Product):** Align with Agneta and Tommy (Tribe product) to confirm no edge cases are missed; validate the two business scenarios (merge vs. separate profiles).
- **Tomasz Kowalski / Michal Rosikiewicz:** Sync with Erik if needed to clarify internal async flow details; potentially create a flow diagram to resolve remaining internal uncertainty.

---

## Key Takeaways

1. **The bug is structural:** Tribe's data model requires a new Contact Person for each company association, and the connector syncs Contact Persons — not Persons — making duplicates inevitable without a merge step.
2. **Syncing Person instead of Contact Person is ruled out** on product grounds: Contact Person holds richer data (e.g. company-specific email fields) that would be lost.
3. **A synchronous merge call from Tribe is ruled out** due to the async nature of the Apsis webhook → data-fetch flow (race condition).
4. **Agreed solution:** Tribe embeds an `original_contact` CRM ID in the sync response payload only when the scenario matches "first real-company association of an Apsis lead." Apsis creates the new profile and then merges it with the profile identified by `original_contact`.
5. **The merge should only happen once** (on first company association). Subsequent company associations for the same person indicate a genuine multi-company scenario and should produce separate Apsis profiles.
6. **The generic connector may already support this** via an existing flag mechanism — this must be verified before committing to building new functionality.
7. Merge operations in Apsis are **idempotent** for the same pair — safe to call multiple times.

---

## Unresolved Questions

- Does the existing generic connector merge flag fully address this scenario, or does new development remain needed? (Lukasz to investigate.)
- Exact field name for the metadata (e.g. `original_contact`, `original_lead`) — to be agreed in the specification Lukasz will produce.
- Does the proposed Baptiste-side logic (check for Apsis Lead company linkage) cover all edge cases, including older contacts that pre-date this flow? Baptiste flagged awareness of this: "if for some reason there is an old contact that has been changed to some company, it will not affect that."
- Full confirmation from Agneta/Tribe product on the two business scenarios and any additional edge cases Henrik will gather.
- Whether the solution should use the **webhook** or the **sync response payload** to carry the `original_contact` ID — deferred to the specification phase.