---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One Integrations
topics: [Keyspace architecture and naming, Entity-specific keyspaces, Form submission handling, CRM system entity types, Data isolation and merging strategies, Integration configuration and installation]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace discriminators, Installation manager, Entity keyspaces, Form submission events, CRM ID mapping, E-deal integration, Efficy Enterprise 12.0/12.1, Tribe CRM, Microsoft Dynamics, FSC Corporate]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics Integration, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration, Tribe Integration, E-deal Integration, Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This session covers the technical architecture of keyspaces in Apsis One integrations, focusing on how different CRM systems create and use entity-specific keyspaces during installation and form submission workflows. The discussion explains the discriminator format (hash-based for uniqueness), why keyspaces are never deleted, how lead/silhouette entities are separated from contact entities to prevent unwanted merging, and the differences in entity support across various CRM connectors. The session concludes with discussion of a pending customer request to override default keyspace behavior for a custom integration scenario.

---

## Keyspace Discriminator Architecture

### Discriminator Format and Purpose

The keyspace discriminator follows a consistent pattern across all CRM integrations:

```
integrations:keyspaces:[8-character hash]:[logical_crm_name]
```

[Erik Andersson]: The first 8 characters of the hash are derived from the section discriminator. This hash provides uniqueness when multiple installations of the same CRM system exist on different sections.

[Lukasz Grabowski]: The hash is strictly for uniqueness purposes, not for reverse identification. You cannot resolve the discriminator back to the original section.

[Erik Andersson]: The choice of 8 characters was somewhat arbitrary, but the purpose is straightforward—ensuring each CRM installation has a unique keyspace identifier.

### Why Hash-Based Uniqueness is Required

[Erik Andersson]: You can have multiple installations of a CRM system. Without a hash-based discriminator, there would be no way to differentiate keyspaces across different sections. The hash combined with the logical CRM name ensures uniqueness.

In test scenarios, this is evident when systems are installed, uninstalled, and reinstalled multiple times on the same section—each installation can reference the same logical CRM entity.

---

## Keyspace Retention and Reinstallation

### No Deletion on Uninstall

[Lukasz Grabowski]: When we uninstall an integration, do we delete the keyspace?

[Erik Andersson]: No, we do not delete keyspaces. If you reinstall the same CRM integration on the same section, the existing keyspace is adopted and reused.

### Business Rationale for Retention

There is a critical data preservation requirement: **profiles synced to Apsis must remain in Apsis unless explicitly removed via GDPR cleanup or similar actions**.

The rationale includes:
- **Win-back scenarios**: If a customer is lost and then returns, their historical data should be available
- **Event history preservation**: Historical contact events and activity data would be lost if keyspaces were deleted
- **Consent synchronization**: Stored consent decisions need to persist independent of integration reinstallation

[Erik Andersson]: This data retention policy is enforced in the installation flow through the installation manager's main install function, where the keyspace setup is performed as an early step.

---

## Entity-Specific Keyspaces for Leads and Silhouettes

### Why Additional Keyspaces Are Created

When a CRM integration supports entities beyond the primary contact entity (such as leads or silhouettes), additional keyspaces are created during installation.

**Core Problem Solved**: Different entity types must have separate keyspaces to prevent them from being treated as full contacts in export operations.

[Erik Andersson]: For E-deal, we create a separate silhouette keyspace because E-deal responds with silhouette IDs from form submissions. If these IDs were added to the main CRM keyspace, they would be interpreted as full contacts. This would cause issues in consent exports back to the CRM, where we would incorrectly identify and attempt to sync silhouettes as if they were complete contact records.

[Lukasz Grabowski]: So the additional keyspace is separate from the main CRM installation keyspace. You cannot use the main keyspace for silhouettes.

[Erik Andersson]: Correct. Using the main keyspace for silhouettes causes problems, particularly with consent handling. If a CRM ID has a consent change, listeners would pick it up and we would start sending consent changes back to Apsis—behavior that is outside the intended scope.

### Discriminator Format for Entity Keyspaces

Entity-specific keyspaces use the same base discriminator as the installation keyspace, with the entity name appended:

```
integrations:keyspaces:[8-character hash]:[logical_crm_name]:[entity_name]
```

For example, an E-deal silhouette keyspace might be:

```
integrations:keyspaces:[hash]:FSC_corporate:silhouettes
```

[Erik Andersson]: Adding an explicit entity type (e.g., `.contact`) to the discriminator would be more precise, but was not implemented due to the data migration overhead for existing customers.

---

## Keyspace Lookup and Storage

### Storage in Database

When an integration is installed, the system generates and stores keyspace discriminators for each supported entity type in the database. This enables efficient lookup at runtime.

[Erik Andersson]: We store the mapping: for a specific installation (account, section, integration ID), here is what the keyspace discriminator will look like. When we need to interact with that keyspace, we look up the ID using the stored discriminator.

### Runtime Lookup Process

When processing CRM data:

1. Download entities from CRM (e.g., "get all persons")
2. System knows the entity type downloaded
3. Lookup the keyspace ID for that entity type using the stored discriminator
4. Use that keyspace ID for profile updates

[Erik Andersson]: If we receive a form submission response indicating the entity is of type `silhouette`, we check our mappings for the silhouette keyspace discriminator, retrieve its ID, and use that ID when updating silhouette records.

---

## Form Submission Workflow and Entity Responses

### Form Sync Registration

When a form is configured to sync to a CRM system:

1. The "Sync to CRM" option is only visible if an integration is installed on the section
2. When the form is published, event listeners are registered for that campaign
3. For form campaigns, listeners are registered for `start_viewed` and `submit` events

[Lukasz Grabowski]: So the form is public, submitted, and then a submit event is sent to the CRM integration—but only if an integration is enabled on the section?

[Erik Andersson]: Exactly. If integration is not installed or does not support form syncing, the "Sync to CRM" option is not displayed. If you later uninstall the integration, the event listeners are removed. Under normal circumstances, form submit events are never sent to integrations that aren't installed or don't support form syncing.

### Form Submission Data Structure

When a user submits a form in Apsis, the system sends:

```
{
  campaign_events_and_points: {
    profile_key: "...",
    fields: {
      email: "user@example.com",
      first_name: "John",
      last_name: "Doe",
      custom_field: "value"
    }
  }
}
```

**Identifying Information**: The system includes any identifying data available:
- Email address (if filled in form)
- Phone number (if available)
- Existing CRM ID (if available via pre-filled forms)
- APSIS profile key

[Erik Andersson]: Historically, pre-filled forms were not possible, so existing CRM IDs couldn't be sent. This changed with the pre-filled form feature.

### CRM Response Handling

The CRM system can respond in multiple ways depending on what it finds:

#### New Record Created
The CRM creates a new record and responds with:
- Entity type (e.g., `contact`, `silhouette`, `lead`)
- New record ID

#### Existing Record Found
The CRM finds an existing record (same email, same person, existing customer) and responds with:
- Entity type of the existing record
- Record ID of the existing record

---

## Entity Type Differences Across CRM Systems

### E-deal / FSC Corporate

**Entity Types Supported**: `contact`, `silhouette`

[Erik Andersson]: E-deal responds with silhouette entities when leads are created from form submissions. They do not create actual customers (full contact records) from these submissions.

Keyspaces created on installation:
- Main contact keyspace: for synced contacts
- Silhouette keyspace: for leads captured via forms

When a form submission occurs:
1. E-deal may create a silhouette and respond with silhouette ID
2. System stores this ID in the silhouette keyspace and merges it with the profile via the export keyspace
3. This prevents duplicate creation if the same person submits again
4. Later, if the CRM converts the silhouette to a contact, they send a merge request

### Efficy Enterprise 12.1

**Entity Types Supported**: `contact` only

[Erik Andersson]: Efficy Enterprise 12.1 does not have a concept of leads. They always create contacts, even from form submissions. This simplifies the integration significantly.

Keyspaces created on installation:
- Only the main contact keyspace

When a form submission occurs:
1. Efficy Enterprise 12.1 always responds with a contact
2. Only one keyspace is used for all operations
3. No separate entity handling is needed

This requires Efficy Enterprise 12.1 to manage temporary vs. permanent contacts internally.

### Tribe CRM

**Entity Types Supported**: `lead` (in theory), `contact` (in practice)

[Erik Andersson]: Tribe is particularly complicated. While they have a lead entity in their system, they do not respond with leads when we submit forms. Instead, their platform has a dynamic dropdown menu where customers select which entity type should be created from a form submission.

The Problem:
- Tribe's dropdown supports 10+ different entity types
- Apsis needs pre-configured entity definitions (unique field names, entity names)
- When Tribe's field endpoint is queried for "lead", they return the field definitions of whatever entity is currently selected in their dropdown

[Erik Andersson]: The lead keyspace in Apsis represents Tribe's dynamic dropdown. When the dropdown is set to "contact", querying for lead fields returns contact field data. This is unintuitive and creates maintenance challenges. The system works, but it's awkward because the "lead" keyspace doesn't actually represent leads—it represents whatever is dynamically selected.

Workaround in place: The current implementation accommodates Tribe's behavior, but Erik Andersson is working to simplify this for future versions.

### Microsoft Dynamics (Legacy)

**Entity Types Supported**: `contact` only

Similar to Efficy Enterprise 12.1, legacy Dynamics only supports contacts.

Keyspaces created on installation:
- Only the main contact keyspace

### Microsoft Dynamics (by SiteShop)

**Entity Types Supported**: `contact`, `lead`

Keyspaces created on installation:
- Main contact keyspace
- Lead keyspace

This version supports both leads and contacts, similar to E-deal but with different entity naming.

---

## Preventing Unwanted Profile Merges

### The Core Issue

[Erik Andersson]: We used to merge CRM-synced contacts with the email keyspace. However, product requested we remove this because multiple contacts in the CRM can have the same email address. If we merged by email, that uniqueness would be lost.

[Lukasz Grabowski]: When can profiles be merged, and when do we perform merges?

[Erik Andersson]: In integration, we only merge within the same CRM integration—specifically, we merge lead keyspace with contact keyspace. We never merge with the email keyspace or any other keyspace.

### Why CRM-Specific Merging is Required

**Principle**: If contacts from a CRM system need to be merged, they should be merged in the source CRM system, not in Apsis.

Rationale:

1. **Sync Direction**: When profiles are merged inside Apsis, the merge is not propagated back to the CRM system
2. **Data Integrity**: The CRM system is considered the master data source. If you merge two profiles in Apsis and update attributes, the CRM system will see them as out of sync
3. **Email/Personal Data Risk**: If a profile from the CRM is merged with another profile (e.g., email keyspace profile with first name), the merged contact will have different personal data than the CRM system. Sending an email to this person could include completely incorrect personal information

[Erik Andersson]: This is sensitive. The CRM system should be the master. If merges happen in Apsis on CRM-originated data, we risk data quality and privacy issues.

### Merge Request Endpoints

When leads are created in Apsis via form submission and later converted in the CRM:

The CRM can send merge requests to indicate:
- Silhouette with ID X is now the same person as contact with ID Y
- Apsis will then merge these two identifiers together

Or the CRM can request deletion:
- Silhouette with ID X did not lead anywhere, please delete it

---

## Keyspace Configuration and Installation Flow

### Main Installation Process

[Erik Andersson]: In the installation manager's main install function, we perform these steps:

1. Create the main keyspace for the CRM system (using the configuration-based discriminator)
2. Loop through additional entities supported by this CRM integration
3. For each additional entity, create an entity-specific keyspace

Example pseudo-code structure:

```
installer_keyspace_create(configuration)  // Main contact keyspace

for entity in additional_entities:
    entity_keyspace_create(entity_name)   // Lead, silhouette, etc.
```

### Determining Supported Entities

The supported entities are determined by:
- CRM integration type (E-deal, Efficy, Dynamics, Tribe, etc.)
- CRM version (e.g., Efficy 12.0 vs 12.1)
- CRM capabilities (whether leads are supported)

This is pre-configured in the connector configuration, not dynamically detected.

---

## Key Takeaways

1. **Keyspace discriminators** use a hash-based approach (`[8-char-hash]:[crm_logical_name]`) to provide uniqueness across multiple installations of the same CRM system

2. **Keyspaces are never deleted** upon integration uninstall to preserve historical contact data and support win-back scenarios

3. **Entity-specific keyspaces** (for leads, silhouettes) are created during installation to prevent different entity types from being merged or incorrectly exported

4. **Lead/silhouette separation** is critical: form submissions that create leads should not be treated as full contacts in downstream export operations

5. **Form submission flow** sends identifying information (email, phone, CRM ID) to the CRM and expects an entity type response indicating what was created or found

6. **CRM systems vary significantly** in their entity support:
   - Efficy Enterprise 12.1: contacts only
   - E-deal/FSC Corporate: contacts and silhouettes
   - Tribe: dynamic dropdown (complex, inelegant)
   - Dynamics (SiteShop version): contacts and leads
   - Dynamics (legacy): contacts only

7. **Merging strategy**: CRM-originated contacts should only be merged within their own keyspace (e.g., lead-to-contact). Merging with other keyspaces (email, etc.) risks data integrity and consistency with the CRM master

8. **Email uniqueness per CRM**: Unlike the email keyspace, multiple contacts in a CRM can have the same email, so CRM contacts are keyed by CRM ID, not email

---

## Unresolved Questions and Action Items

### Pending Customer Request (Not Addressed in This Session)

**Context**: A customer inquiry has been submitted regarding custom keyspace behavior for the "JoinCX" CRM connector.

**Customer Request**: Instead of using the system-generated keyspace discriminator, override it to use the email keyspace for a custom integration using JoinCX.

**Issues Raised by Erik Andersson**:
- Any change to the default behavior would apply to all JoinCX customers, not just this one
- There is no business case identified for this request
- Implementation would require code changes, which Erik considers unfavorable
- Alternative solutions might exist but require more focused discussion time

**Status**: Deferred to a follow-up meeting scheduled for mid-week (Wednesday) to discuss potential non-code solutions. Lukasz Grabowski will respond to the customer noting that the team needs more investigation time and will reconvene next week.

**Next Steps**:
- Erik Andersson will reply to the email thread (in Swedish) clarifying the status
- Follow-up meeting scheduled for Wednesday to explore alternatives
- No code changes anticipated at this stage