---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One Integrations
topics: [keyspace discriminator structure, keyspace lifecycle, lead vs contact separation, silhouette entity handling, form submission flow to CRM, per-CRM entity configurations, profile merging risks, JoinCX custom keyspace request]
speakers: ["Erik Andersson (Integration domain expert)", "Lukasz Grabowski (Developer/Team Lead)"]
key_components: [Apsis One, CRM integrations, keyspaces, silhouettes/leads, form submission pipeline, E-deal, FSC Corporate, FSC Enterprise 12.1, Tribe, Microsoft Dynamics, Dynamics by SiteShop, JoinCX generic connector]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Duplicate profiles"]
---

## Session Overview

Erik Andersson walks Lukasz Grabowski through the keyspace architecture used in Apsis One's CRM integrations, covering how keyspace discriminators are constructed and why they are never deleted on uninstall. The session explains why leads/silhouettes must be stored in separate keyspaces from main contacts, and how the form submission flow triggers CRM responses that result in silhouette IDs being registered on profiles. Erik also surveys how each supported CRM system differs in its entity model (contacts only vs. leads/silhouettes), and the session closes with a discussion of a one-off customer request to override keyspace behaviour in the JoinCX generic connector — which both speakers assess as undesirable.

---

## Keyspace Discriminator Structure and Uniqueness

The **keyspace discriminator** for a CRM integration follows this format:

```
integrations.<section_discriminator_hash>.<crm_logical_name>
```

- The hash segment is the **first 8 characters of the hash of the section discriminator**.
- The purpose is purely **uniqueness** — it is not designed to be resolved back to identify which section it came from.
- The 8-character length was a deliberate choice, though not driven by a strict technical requirement.

> [Erik]: "It's just for some kind of uniqueness. I don't know why we took specifically 8, but it was a choice."

Because a section can have multiple installations of a CRM system (e.g., install, uninstall, reinstall), the hash ensures each installation configuration is uniquely addressable.

---

## Keyspace Lifecycle: No Deletion on Uninstall

Keyspaces are **never deleted** when a CRM integration is uninstalled.

- If the same CRM is reinstalled on the same section, the existing keyspace is **adopted**, not recreated.
- The rationale is a product requirement: profiles synced to Apsis should remain in Apsis unless explicitly removed (e.g., via a GDPR wipe).

> [Erik]: "There was a requirement that the profiles synced to Apsis should stay in Apsis unless explicitly otherwise said — for example by someone doing a GDPR clean — because you want to win back customers and you want their data to be there. In particular for events, the contacts we can always sync back, but the historical event data would have been gone."

Keyspace setup happens in the **installation manager** inside the main install function, as a dedicated step.

---

## Main Contact Keyspace vs. Entity Keyspaces (Leads/Silhouettes)

### The Main (Contact) Keyspace

When a CRM is installed, a **main keyspace** is created for the primary contact entity. This is keyed from the installation configuration alone.

Previously, this keyspace was merged with the **email keyspace**. This was removed at product's request because multiple contacts in a CRM can share the same email address, and merging would collapse them into a single profile in Apsis — destroying the CRM's ability to have distinct records per email.

> [Erik]: "There was a request from product — we had to remove that because there should be able to be multiple contacts in the CRM that have the same CRM email, and that possibility disappears if we are merging."

In integrations, the only keyspace used for a contact profile is **the CRM-specific keyspace for that installation**.

### Entity Keyspaces for Leads and Silhouettes

A separate **entity keyspace** is bootstrapped at install time for CRM systems that have lead or silhouette entities. The discriminator for these follows the same format as the main keyspace but appends the entity name:

```
integrations.<section_discriminator_hash>.<crm_logical_name>.<entity_name>
```

Example: for E-deal silhouettes, the entity name `silhouettes` is appended.

> [Erik]: "If you wanted to be even more specific, you could have added `.contact` to the regular key space to really say which entity it was for. The reason we did not go with this is that it would have required a lot of data migration for existing customers."

The keyspace discriminator strings are **stored in the database** at install time, so when the system needs to interact with a specific entity type, it does a lookup by discriminator to retrieve the keyspace ID.

### Why Leads/Silhouettes Must Not Share the Contact Keyspace

If a silhouette ID were stored in the main contact keyspace, the system would treat the silhouette as a full contact. This causes two problems:

1. **Consent sync contamination**: The integration has listeners for consent changes. If a silhouette's CRM ID were in the contact keyspace, consent change events would be picked up and sent back to the CRM for the silhouette — which is out of scope. Only form submissions and silhouette ID registration should occur for silhouettes.
2. **Duplicate creation**: Without the separate keyspace to tie the profile key to the silhouette record ID, every form submission from the same person would create a new duplicate.

> [Erik]: "We will add the record ID as the silhouette ID for the contact and do it via our silhouette keyspace, doing a merge using the export keyspace with this profile key and our silhouette keyspace with this one — because otherwise you will start creating duplicates all the time."

---

## Form Submission Flow to CRM and Silhouette Registration

### How Form-to-CRM Sync is Triggered

1. When a user selects "Sync to CRM" on an Apsis form, the integration registers **event listeners** for that campaign: `start`, `viewed`, and `submit` events.
2. The "Sync to CRM" option is only visible if a CRM integration is installed on the section.
3. If a CRM does not support form submissions (e.g., the `can_sync_form_activities` flag is `false`, as with legacy Dynamics), the option is not displayed, no event listeners are registered, and no form submit events will ever reach that integration.
4. If a CRM integration is uninstalled, its registered event listeners are removed.

### What Is Sent on Form Submission

The payload sent to the CRM includes:
- Form field values (email, first name, last name, custom fields like "has a pet")
- The **Apsis profile key**
- Any identifying data that exists: email address, phone number, or a pre-existing CRM ID or lead ID (the latter became possible after pre-filled form support was implemented)

These identifying values are placed under a special property called **`fields`** in the payload, which the CRM uses to identify or create a person.

### CRM Response and Silhouette ID Registration

The CRM responds with the entity it created or found:

- **Contact response**: The CRM created or found a full contact record. The contact's record ID is stored in the main contact keyspace.
- **Silhouette response** (e.g., E-deal, FSC Corporate): The CRM responds with entity type `silhouette` and a new silhouette ID. The integration:
  1. Looks up the silhouette keyspace discriminator from the stored mappings.
  2. Retrieves the keyspace ID.
  3. Adds the silhouette record ID to the profile via the silhouette keyspace.
  4. Performs a merge between the export keyspace (using the profile key) and the silhouette keyspace (using the silhouette ID), tying the two together to prevent future duplicates.
- **Existing record response**: The CRM may respond with an already-existing contact or lead (matched on email, for example), in which case that existing record ID is used.

### Silhouette Lifecycle: Merge or Delete

Once a silhouette is registered in Apsis, the CRM system drives what happens next:

- **Merge**: The CRM determines the lead became a real customer. They call the integration's merge endpoint, providing the person entity ID and the silhouette ID. The integration merges them on the Apsis side.
- **Delete**: The CRM determines the lead went nowhere. They call the integration to delete the silhouette.

> [Erik]: "If contacts that come from the CRM system need to be merged, they should be merged from the CRM — because that will reach us and then reach Apsis. When you merge them inside of Apsis, that will not be proliferated to the CRM system."

---

## Profile Merging Risks and the CRM as Data Master

Merging CRM-sourced profiles from within Apsis (rather than from the CRM side) is considered **dangerous**:

- In integrations, profiles are only ever merged between the **lead/silhouette keyspace** and the **contact keyspace** — never with the email keyspace or any other keyspace.
- If an external tool merges a CRM profile with an email-keyspace profile, the resulting merged profile may have attributes from the email profile that differ from the CRM's data. The CRM is the **master of the data**, so the merged profile will be out of sync.
- Sending an email to such a merged profile risks including personal data that doesn't actually belong to that person.

> [Erik]: "It has kind of always been my opinion that if contacts that come from the CRM system need to be merged, they should be merged from the CRM."

---

## Per-CRM Entity Configuration Survey

Not all CRMs require a lead/silhouette keyspace. The entity model varies per connector:

| CRM | Entity Keyspaces Created on Install | Notes |
|---|---|---|
| **Legacy Dynamics** (old in-house) | Contact only | No lead support; only one keyspace created |
| **FSC Enterprise 12.1** (generic connector) | Contact only | No lead concept; always creates contacts; responds with contacts on form submit, download, and webhooks |
| **E-deal** | Contact + Silhouette | Bootstraps a silhouette keyspace; responds with silhouette entity on form submit |
| **FSC Corporate** | Contact + Silhouette | Same as E-deal; silhouette keyspace bootstrapped on install |
| **Dynamics by SiteShop** | Contact + Lead | Supports leads; lead keyspace is created |
| **Tribe** | Contact + Lead (nominal) | ⚠️ **Highly irregular** — see below |

### Tribe: Special Warning

Tribe bootstraps a lead entity keyspace on install, but **never responds with a lead** on form submissions. In Tribe, the entity created from a form submission is controlled by a **dropdown inside Tribe's own configuration**, where the customer selects which entity type to create (10+ options available).

The problem is that Apsis cannot fully support dynamic entities — the integration needs to know the entity's unique field and name ahead of time, configured in the connector.

**The practical consequence**: When Apsis requests field definitions for Tribe's "lead" entity, Tribe returns the fields for whatever is selected in that dropdown. If the customer selected "contacts," Apsis gets contact fields. If the customer selected "sailing boats," Apsis gets sailing boat fields.

> [Erik]: "That's why this lead entity in Apsis represents their dynamic dropdown. This is super irritating and I'm trying to simplify it for them, but that's how it is for the time being."

---

## JoinCX Custom Keyspace Request (Unresolved)

### The Request

A customer using the **JoinCX generic connector** has requested that instead of installing with the standard JoinCX-named keyspace discriminator, the integration should use the **email keyspace discriminator** instead. Their reason: they use a custom integration outside of Apsis that relies on email addresses, not CRM IDs, so they want the email keyspace to be the basis.

### Why This Is Problematic

1. **Scope**: Any change made to the generic connector's installation flow would apply to **all JoinCX customers**, not just this one.
2. **Complexity**: Implementing a per-customer override of the keyspace discriminator in the generic connector would be architecturally messy.
3. **Business case**: Both Erik and Lukasz agree there is no sufficient business case to justify a code change for a single customer.

> [Erik]: "I see absolutely no business case for this if it is one customer that has asked this."
> [Lukasz]: "It's not designed to handle it like this — it's some really custom thing for one customer."

### Possible Non-Code Solutions

Erik indicated there may be ways to address this without code changes but needed more focused time to explain them properly. This was deferred.

---

## Key Takeaways

- The keyspace discriminator is an 8-char hash prefix + CRM logical name (+ optional entity name), stored in DB at install time and used for all subsequent lookups.
- Keyspaces are **never deleted** on uninstall — by design, for data retention and GDPR compliance.
- Leads/silhouettes **must** have their own keyspace, separate from the main contact keyspace, to prevent both consent-sync contamination and duplicate profile creation.
- The email keyspace is **explicitly not used** in integrations, to support multiple CRM contacts sharing the same email.
- Profile merges for CRM-sourced data should always originate from the CRM, not from within Apsis, to keep data in sync.
- CRM systems differ significantly in their entity models — FSC Enterprise 12.1 and legacy Dynamics are contact-only; E-deal and FSC Corporate use silhouettes; Tribe's lead entity is a proxy for a dynamic dropdown and behaves unexpectedly.
- Form-to-CRM sync is only active when explicitly configured per form, and only for CRMs where `can_sync_form_activities` is `true`.

---

## Unresolved Questions / Action Items

- **JoinCX custom keyspace request**: Erik to reply to the email thread (in Swedish) clarifying status. Lukasz to communicate that the team needs more time and will respond the following week. Follow-up meeting to be scheduled for **Wednesday of the following week**.
- **Non-code solutions for JoinCX**: Erik mentioned possible approaches that do not require code changes; these need a dedicated focused session to explain properly.
- ⚠️ **Ambiguity**: It is unclear whether the silhouette keyspace discriminator always uses the literal string `silhouettes` or whether the entity name is derived dynamically from the CRM's entity definition. Erik referenced "adding the silhouettes name to it" but did not state the exact string used for each CRM.