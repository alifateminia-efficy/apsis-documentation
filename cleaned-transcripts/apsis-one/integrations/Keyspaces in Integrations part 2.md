---
source_file: "Keyspaces in Integrations part 2.txt"
domain: Apsis One Integrations
topics: [keyspace discriminator structure, keyspace lifecycle, CRM installation flow, entity keyspaces, silhouette/lead entities, form submission flow, CRM-specific entity support, keyspace merging policy, JoinCX custom keyspace request]
speakers: ["Erik Andersson (Integration Engineer/Domain Expert)", "Lukasz Grabowski (Engineering Lead)"]
key_components: [Apsis One, Integrations service, keyspaces, CRM connectors, E-deal, FSC Corporate, FSC Enterprise 12.1, Microsoft Dynamics, Dynamics by SiteShop, Tribe, JoinCX, General Connector, installation manager, event listeners, form campaigns, silhouette entities]
session_type: knowledge-transfer
---

## Session Overview

This session is the second part of a knowledge transfer covering how keyspaces are structured and managed within the Apsis One Integrations domain. Erik Andersson walks Lukasz Grabowski through the keyspace discriminator format, the rationale for not deleting keyspaces on uninstall, the distinction between main contact keyspaces and entity-specific keyspaces (for leads/silhouettes), and how different CRM systems vary in their entity support. The session also covers how form submission events are routed to CRM systems and how the resulting CRM responses (contact ID, silhouette ID, lead ID) are stored back on Apsis profiles. A pending external request to override default keyspace behaviour for a single JoinCX customer is briefly discussed at the end.

---

## Keyspace Discriminator Format and Uniqueness

### Structure of the Discriminator

The keyspace discriminator for CRM integrations follows this pattern:

```
integrations.<keyspaces>.<8-char-hash>.<crm-logical-name>
```

- The first portion is a fixed prefix: `integrations.keyspaces`
- The hash segment is the **first 8 characters of a hash of the section discriminator**
- The hash exists purely for **uniqueness** — it allows multiple installations of the same CRM system on a section to have distinct keyspaces
- The hash is **not intended to be resolved back** to identify which section or installation it belongs to; it is opaque by design
- The choice of 8 characters was an arbitrary decision at the time

> [Erik Andersson]: "It's just for some kind of uniqueness. I don't know why we took specifically 8, but it was a choice."

### Entity-Specific Keyspace Discriminator

For entities beyond the main contact (e.g., silhouettes, leads), the discriminator extends the base format by appending the entity name:

```
integrations.<keyspaces>.<8-char-hash>.<crm-logical-name>.<entity-name>
```

**Note:** Adding `.contact` to the main contact keyspace discriminator was considered but was deliberately avoided because it would have required a data migration for existing customers.

---

## Keyspace Lifecycle: Installation and Retention

### Creation During Installation

Keyspace setup happens inside the **installation manager**, in the main `install` function. The steps are:

1. **Main keyspace** is created using the installation configuration (covers the primary contact entity)
2. A **for loop over additional entities** creates entity-specific keyspaces where needed (e.g., silhouette, lead)

The generated discriminators are stored in the database, keyed by account/section + integration ID + entity type. At runtime, a lookup by discriminator retrieves the keyspace ID, which is then used for all profile update operations.

### Keyspaces Are Never Deleted on Uninstall

> [Erik Andersson]: "No, we don't delete keyspaces."

If the same CRM is reinstalled on the same section, the existing keyspace is reused. The rationale:

- There was (and remains) a product requirement that **profiles synced to Apsis should stay in Apsis** unless explicitly removed (e.g., via GDPR cleanup)
- This is important for **win-back scenarios**: contact attributes can always be re-synced, but **historical event data would be permanently lost** if the keyspace were deleted
- Contacts can always be re-synced from the CRM, but event history cannot be reconstructed

---

## Main Contact Keyspace vs. Entity Keyspaces (Leads/Silhouettes)

### Why Contacts and Leads Must Be Separate Keyspaces

CRM contacts downloaded into Apsis are stored in the **CRM-specific installation keyspace** (the main keyspace). Leads and silhouettes must **not** be placed in this same keyspace. The reason:

- If a silhouette ID were placed in the main contact keyspace, it would be treated as a full contact
- Apsis has **consent change listeners** watching the main CRM keyspace. If a silhouette appeared there, Apsis would start sending consent updates back to the CRM for that silhouette — which is out of scope
- The integration should only forward **form submissions** and register silhouette IDs on the profile. The CRM is responsible for deciding when a silhouette becomes a full contact

### Silhouette-to-Contact Merge Flow

When a CRM promotes a silhouette to a full contact, they notify Apsis via a **merge request endpoint**:

1. CRM sends: "Silhouette X is the same person as Contact Y, here is the full person record"
2. Apsis merges the silhouette keyspace entry with the contact keyspace entry on its side

Alternatively, the CRM can send a **delete request**: "This silhouette never converted, please delete it."

---

## Keyspace Merging Policy

### Current Behaviour: No Cross-Keyspace Merges

Previously, CRM keyspaces were merged with the **email keyspace**. This was removed at product's request because:

- Multiple CRM contacts can share the same email address
- Merging with the email keyspace would collapse these into a single profile, breaking that model

**Current rule:** Integration only merges within its own keyspace family:
- Lead keyspace ↔ Contact keyspace (within the same integration)
- **Never** merges CRM keyspaces with the email keyspace or any other external keyspace

### Warning: Merges Initiated Inside Apsis

[Erik Andersson — strong opinion]: If merges are triggered from *within* Apsis (rather than from the CRM side), this will **not** propagate back to the CRM system. This creates a data consistency risk:

- Merging a CRM contact profile with an email keyspace profile inside Apsis can silently overwrite CRM-sourced attributes (e.g., first name) with data from the email profile
- **The CRM is considered the master of contact data.** If the CRM's data differs from what is now on the merged Apsis profile, the profile is effectively out of sync
- This could result in incorrect personal data being used in email sends

> [Erik Andersson]: "It's been kind of always my opinion that if contacts that come from the CRM system need to be merged, then they should be merged from the CRM, because that will reach us and then reach Apsis. When you merge them inside of Apsis, that will not be proliferated to the CRM system."

---

## Form Submission Flow and CRM Event Routing

### Prerequisites for Form-to-CRM Sync

- The **"Sync to CRM"** option on a form campaign is only visible if a CRM integration is installed on the section
- Each CRM connector exposes a capability flag: `can_sync_form_activities`. If `false` (e.g., legacy Dynamics), the sync option is hidden and **no event listeners are registered**
- If an integration is uninstalled, its event listeners are removed

### Event Listener Registration

When a form is synced to a CRM, Apsis registers listeners for these form campaign events:
- `start`
- `viewed`
- `submit`

### Submit Event Processing

When a form is submitted:

1. Apsis collects identifying fields from the submission: **email address, phone number**, and any pre-existing **CRM ID or lead ID** on the profile
2. These are sent to the CRM under a special `fields` property within `campaign_events`
3. **Pre-filled forms** (a more recent feature) enable existing CRM/lead IDs to be included in the submission payload — previously this was not possible

### CRM Response Handling

The CRM responds with the entity it created or matched:

- **Contact response:** A new or existing contact record ID is returned. Apsis stores this as the CRM contact ID in the main contact keyspace for that profile.
- **Silhouette/Lead response** (e.g., E-deal, FSC Corporate): The CRM returns a silhouette entity type and ID. Apsis:
  1. Identifies the silhouette keyspace discriminator from its stored mapping
  2. Retrieves the keyspace ID
  3. Does a **merge** using the export keyspace (profile key) + silhouette keyspace (new silhouette ID) — tying the anonymous profile to the silhouette record to prevent duplicate creation on subsequent submissions with the same email

---

## CRM-Specific Entity and Keyspace Configurations

### Overview

Not all CRM systems use leads or silhouettes. The keyspaces bootstrapped on installation vary by connector:

| CRM System | Keyspaces Created | Notes |
|---|---|---|
| **FSC Enterprise 12.1 (Generic Connector 12.1)** | Contact only | No lead concept; always creates contacts for form submissions, downloads, and webhooks |
| **Legacy Dynamics** | Contact only | Same as above |
| **Dynamics by SiteShop** | Contact + Lead | Supports lead entity |
| **E-deal** | Contact + Silhouette | Form submissions always return silhouette; explicit merge request needed to promote to contact |
| **FSC Corporate** | Contact + Silhouette | Same silhouette model as E-deal |
| **Tribe** | Contact + Lead | Lead keyspace is bootstrapped, but Tribe **never responds with a lead on form submission** |

### Tribe — Special Case and Known Irritant

Tribe has a dynamic entity configuration: inside the Tribe CRM, there is a dropdown where the customer selects **which entity type should be created** from an Apsis form submission (10+ options possible).

The problem:
- Apsis does not support fully dynamic/arbitrary entities because it needs to know the entity's unique field and logical name at configuration time
- When Apsis queries Tribe for "the fields available for the lead entity," Tribe returns **the fields for whatever entity is currently selected in their dropdown** — not necessarily a lead
- Example: if the customer has selected "contacts" in Tribe's dropdown, querying lead fields returns contact data; if they selected "sailing boats," it returns sailing boat data

> [Erik Andersson]: "That's why we kind of — this lead entity in Apsis represents their dynamic dropdown. This is super irritating and I'm trying to simplify it for them, but that's how it is for the time being."

**⚠️ Warning:** The Tribe lead keyspace in Apsis is a proxy for whatever Tribe's dynamic entity selection is. The actual entity type created on the Tribe side depends on Tribe's internal configuration, not on what Apsis labels the keyspace.

---

## Pending External Request: JoinCX Custom Keyspace Override

### Request Summary

An external party (customer/partner) has requested that for a **specific JoinCX customer installation**, the integration use the **email keyspace discriminator** instead of the generated JoinCX-specific keyspace discriminator. Their rationale: they are using a custom integration outside Apsis that works with email addresses rather than CRM IDs.

### Current Assessment

[Erik Andersson — opposed to the change]:
- Any change to the keyspace discriminator logic in the General Connector would apply to **all JoinCX customers**, not just the one requesting this
- A customer-specific override would be **very complex** to implement
- There is **no identified general business case** for this — it is a one-off customer request
- There are potentially non-code approaches to achieve this, but they require more focused investigation time

[Lukasz Grabowski — aligned]:
- Agreed that this is not designed to be handled this way
- The team does not have the capacity or resources to pursue this
- Very low likelihood of building anything for this

### Current Status / Next Steps

- Erik to reply to the email thread (in Swedish) to indicate the team needs more time and is not yet ready to commit
- Lukasz is out of office for 3 days; discussion to resume the following Wednesday
- Erik wants Lukasz included in any follow-up meeting with the external party because any implementation work would fall on Lukasz's team

> [Erik Andersson]: "There are ways we can make this happen without doing any coding, but I would want more focus time to explain them."

---

## Key Takeaways

1. **Keyspace discriminator format** is `integrations.keyspaces.<8-char-section-hash>.<crm-logical-name>` — the hash is for uniqueness only, not for reverse lookup.
2. **Keyspaces are never deleted on uninstall** — both for data retention requirements and to preserve historical event data that cannot be reconstructed.
3. **Leads and silhouettes must have separate keyspaces** from contacts — mixing them would cause consent listeners to incorrectly treat silhouettes as full contacts.
4. **The CRM is the master of contact data.** Merges should originate from the CRM side; merges initiated inside Apsis do not propagate back to the CRM and can cause data inconsistency.
5. **Form-to-CRM sync** requires explicit opt-in per form campaign; if a CRM doesn't support it (`can_sync_form_activities: false`), no listeners are registered and no submissions are forwarded.
6. **CRM entity support varies significantly**: FSC Enterprise 12.1 and legacy Dynamics are contacts-only; E-deal and FSC Corporate use silhouettes; Tribe's lead entity is a dynamic proxy for their internal entity selector.
7. **Tribe's lead keyspace is a known workaround** for their dynamic entity system — the actual entity created depends on Tribe's own configuration, not Apsis's labelling.
8. **The JoinCX email-keyspace override request** is architecturally problematic, has no general business case, and the team's current position is to investigate non-code solutions only.

---

## Unresolved Questions and Action Items

- [ ] **[Erik]** Reply to the email thread regarding the JoinCX keyspace request — acknowledge receipt, indicate more investigation time is needed, do not commit to any solution
- [ ] **[Erik + Lukasz]** Schedule follow-up session (Wednesday of next week) to explore non-code approaches to the JoinCX request before any meeting with the external party
- [ ] **[Open]** Erik mentioned there are ways to solve the JoinCX request without code changes — this was not elaborated on due to time constraints and remains to be discussed
- [ ] **[Ambiguous ⚠️]** The exact merge endpoint API contract for silhouette-to-contact promotion (what fields are required, versioning) was referenced but not detailed in this session