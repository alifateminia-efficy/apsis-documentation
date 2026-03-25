---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One Integrations
topics: [Keyspace Architecture, Entity-specific Keyspaces, Form Submissions and CRM Sync, Lead vs Contact Entities, Data Uniqueness and Merging Strategy, CRM-specific Implementation Differences]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspaces, Discriminators, Entity Mapping, Form Submission Flow, CRM Installation Manager, E-deal, Efficy Corporate, Microsoft Dynamics, Tribe, Efficy Enterprise 12.1, Event Listeners, Silhouette Entities, Profile Keys]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Outbound Flow, E-deal (Efficy Corporate), Efficy Enterprise 12.0, Efficy Enterprise 12.1, Tribe, Microsoft Dynamics, Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This session is a continuation of a keyspace architecture discussion focused on how Apsis One manages different entity types across various CRM integrations. The conversation covers keyspace discriminator construction and hashing, the rationale for maintaining separate keyspaces for different entity types (contacts vs. leads/silhouettes), how form submissions flow to CRM systems and trigger entity creation, why certain CRM systems support leads while others don't, and a pending customer request regarding custom keyspace behavior for the generic JoinCX connector.

---

## Keyspace Discriminator Construction and Uniqueness

### Hash-Based Uniqueness for Multiple Installations

The keyspace discriminator follows a specific format designed to handle multiple installations of the same CRM system on different sections:

**Format:** `integrations:keyspaces:<first-8-characters-of-hash>:<crm-logical-name>`

[Erik Andersson]: The hash represents the first 8 characters of a hash of the section discriminator. This ensures uniqueness when multiple installations of the same CRM system exist on different sections.

[Lukasz Grabowski]: The purpose of this hash is purely for uniqueness, not for reverse identification. There's no need to resolve which section a keyspace belongs to based on the hash.

[Erik Andersson]: We don't know specifically why 8 characters were chosen as the truncation length, but it was a deliberate choice.

### Handling Reinstallation of CRM Systems

When a CRM integration is uninstalled and reinstalled on the same section, the system reuses the existing keyspace rather than creating a new one:

[Lukasz Grabowski]: If we uninstall and then create the same installation again on the same section, do we adopt the existing keyspace?

[Erik Andersson]: Yes, exactly. We don't delete keyspaces upon uninstallation.

**Rationale for Keyspace Retention:** Historical data preservation is critical. Before GDPR cleanup or explicit deletion requests, profile sync access and historical event data must remain in the system. This supports win-back scenarios where customers may return and their data needs to be available. Particularly for events and contacts, while contact sync can occur again, historical event data would be lost permanently if keyspaces were deleted.

---

## Installation Flow and Keyspace Setup

### Main vs. Additional Entity Keyspaces

The installation process creates keyspaces in two stages:

1. **Main Keyspace** - Created for the primary contact entity
2. **Additional Entity Keyspaces** - Created for supplementary entities (leads, silhouettes, etc.)

[Erik Andersson]: The main installation function in the installation manager handles keyspace setup. We create the main keyspace for the CRM system, but then we have a for-loop for additional entities when they exist beyond the main contact entity.

### The Lead/Silhouette Keyspace Necessity

For CRM systems like E-deal that support lead entities, a separate keyspace must be created for silhouettes:

**Why separate keyspaces are essential:**

- E-deal will return a silhouette ID when a form is submitted, but silhouettes should never be treated as full contacts
- Storing a silhouette ID in the main CRM keyspace would cause it to be interpreted as a full contact
- This would result in the silhouette appearing in consent exports back to the CRM
- Contacts and leads must remain separate until an explicit merge request occurs
- Consent changes on CRM IDs must not trigger consent sync for silhouettes — only form submission registration and silhouette ID storage is in scope

[Lukasz Grabowski]: So in the normal keyspace (the CRM-created keyspace we discussed), we cannot use it for silhouettes because of consent handling issues.

[Erik Andersson]: Correct. If a CRM ID had a consent change, listeners would pick it up and start sending consent changes for silhouettes back from Apsis, which is not in scope.

---

## Entity Keyspace Lookup and Storage

### Database Storage of Entity Keyspace Mappings

The system stores entity keyspace discriminator templates in the database during installation:

For each entity type in an installation, the system generates and stores how the keyspace discriminator will look. When interactions occur with that entity, a lookup retrieves the keyspace ID.

**Workflow example:**
1. During installation: Store the keyspace discriminator template for "person" entity for this account/section/integration
2. When downloading entities from CRM: System requests all "persons" → knows it needs the person keyspace
3. Lookup retrieves the keyspace ID for that discriminator
4. Profile updates use this keyspace ID

### Form Submission Entity Type Routing

When a CRM system responds to a form submission with an entity ID, the response includes the entity type:

[Erik Andersson]: If we receive a response from the CRM system that informs us this ID belongs to a silhouette entity, we check our mappings. We need to update the silhouette ID for that profile. We look up the keyspace discriminator for the silhouette keyspace, retrieve the keyspace ID, and use it for silhouette ID updates.

The same pattern applies regardless of whether the response is for a contact, silhouette, or other entity type supported by that specific CRM.

---

## Contact Profile Merging Strategy and Constraints

### Why Contacts Are Never Merged with Email Keyspaces

Apsis One integrations explicitly avoid merging CRM-synced contacts with email keyspaces:

**Historical constraint:** Previously, contacts were merged with the email keyspace, but this was removed due to a product requirement.

**Business reason:** Multiple contacts from the same CRM system can legitimately have the same email address. Merging CRM profiles with email keyspaces eliminates this possibility, so the integration now uses only CRM-specific keyspaces for CRM-sourced profiles.

[Lukasz Grabowski]: Can we merge profiles when they have the same CRM ID and email?

[Erik Andersson]: Not in integration. We never merge with keyspaces apart from our own entity keyspaces (e.g., merging lead keyspace with contact keyspace). We will never merge with the email keyspace or anything else.

### Risks of External Merging of CRM Contacts

If other tools within Apsis One merge contacts that originated from a CRM system, it creates data synchronization issues:

[Erik Andersson]: If you merge two profiles and one is a CRM system profile and the other is an email keyspace profile, you might update attributes. The CRM system should be considered the master of data. If you merge them and then send an email, you risk adding completely different personal data than the person actually has in the CRM system.

**Guiding principle:** If CRM-sourced contacts need to be merged, the merge should originate from the CRM system itself. Merges performed within Apsis One are not propagated back to the CRM, and create sync issues that make the data unreliable.

---

## Entity Keyspace Discriminator Formats

### Standard Entity Keyspace Discriminator Pattern

Entity-specific keyspaces follow the same base format as the main CRM keyspace, with an entity name appended:

**Example (E-deal silhouette keyspace):**
```
integrations:keyspaces:<hash>:fsc-corporate:silhouettes
```

The discriminator structure is identical up to the CRM logical name, then the entity name is appended.

[Erik Andersson]: For E-deal, the discriminator is the same format as normal — integrations, keyspace, section discriminator hash, and the CRM logical name (FSC corporate). The difference is we're adding the entity name (silhouettes) to it.

### Why Entity Names Weren't Dot-Delimited

Originally considered: `integrations:keyspaces:<hash>:fsc-corporate.contact` (adding `.contact` for specificity)

**Decision:** This was not implemented because it would have required significant data migration for existing customers. The team chose to keep the less specific format and rely on database lookups for entity type resolution.

---

## Form Submission Flow to CRM Systems

### Prerequisites for Form Submission Sync

Form submissions can only be synced to a CRM if:

1. The integration is installed on the section
2. The integration supports form submission sync (not all do)
3. The form is explicitly configured to "sync to CRM"

[Erik Andersson]: You can't even select "sync to CRM" unless the integration is installed. If the integration is installed, the option appears on the form. If it doesn't support form submission sync (`can_sync_form_activities` = false), the option won't display.

**Example:** Microsoft Dynamics doesn't support sending form submissions, so `can_sync_form_activities` is false, the sync option is hidden, and no event listeners are registered for forms.

### Event Listener Registration

When a form is configured to sync to a CRM:

- Event listeners are registered for all possible campaign events
- For form campaigns: `start_viewed` and `submit` events are registered
- Uninstalling an integration removes its event listeners
- No form events are sent to integrations that aren't installed or don't support form sync

[Lukasz Grabowski]: So if we have integration enabled on the section, we can see the option "sync to CRM" on the form.

[Erik Andersson]: Exactly. And if you uninstall an integration, we remove the event listeners we defined.

### Form Submission Payload Structure

When a user submits a form, the system collects:

- Form field values (email, first name, last name, custom fields)
- Apsis profile key (from the profile that submitted the form)
- Identifying data: email, phone, existing CRM ID, or pre-filled lead ID if available
- All identifying data is sent under a special `fields` property

[Erik Andersson]: The CRM will use email address, phone number, or existing CRM ID / lead ID to identify the person or contact. With pre-filled forms (recently implemented), we can now also send existing CRM IDs.

The payload is sent to the CRM under `campaign_events_and_points`.

---

## CRM Response Handling for Form Submissions

### Entity Creation and Silhouette ID Assignment

When a CRM responds to a form submission, it may:

1. **Create a new record:** Return the entity type and new ID
2. **Match an existing record:** Return an existing entity type and ID
3. **Request deletion:** (For leads/silhouettes that didn't convert)

### Contact vs. Silhouette Creation Responses

**Standard response (E-deal creating a contact):**
- CRM indicates: "We created a contact with this ID"
- Apsis stores the contact in the normal CRM keyspace

**Silhouette response (E-deal creating a lead/silhouette):**
- CRM indicates: "We created a silhouette (not a full contact) with this ID"
- Apsis merges the profile across two keyspaces:
  - **Export keyspace:** Profile with Apsis profile key
  - **Silhouette keyspace:** Silhouette entity with CRM-assigned ID
- This merge ensures the same profile doesn't create duplicates on resubmission

[Erik Andersson]: We add the silhouette ID as the silhouette ID for the contact via our silhouette keyspace. We do a merge using the export keyspace with the profile key and the silhouette keyspace with the silhouette ID. This ties those two together so you don't create duplicates.

### Matching Existing Records

If the CRM finds an existing record matching the submitted data:

[Erik Andersson]: They can say "I already have an entry" — maybe a lead with this information or an existing customer using this email address. They give us whatever entity type they support, but here is where it differs between CRM systems.

The entity type in the response varies by CRM implementation.

---

## CRM-Specific Entity Support Differences

### E-deal (Efficy Corporate)

- Supports both contacts and silhouettes
- Responds with silhouettes for lead entities from form submissions
- Additional keyspace created: **E-deal silhouette keyspace**
- Pattern: Two entity types, two keyspaces

### Efficy Enterprise 12.1

- Only supports contacts (no leads concept)
- Always creates contacts from form submissions
- Responds with contacts during downloads and webhooks
- Single keyspace: **FEC Enterprise 12.1 keyspace only**

[Erik Andersson]: FSC Enterprise 12.1 doesn't have a concept of leads. They're always only creating contacts. That's why we haven't needed to create additional entity keyspaces for them. When you install FSC Enterprise 12.1, you'll only see that one keyspace. That's the only thing we'll ever use.

**Additional complexity:** FSC Enterprise 12.1 must internally separate temporary contacts (from forms) from actual customers, but that's their concern, not Apsis's.

### Tribe

- Theoretically supports leads as an entity
- **Critical issue:** Tribe never responds with leads on form submissions
- Tribe's lead entity is virtual — customers select from a dynamic dropdown which entity type to create on form submission
- Apsis limitation: Doesn't support truly dynamic entities; entity configuration must be predefined

[Erik Andersson]: Tribe is very annoying because they will never respond with a lead when we submit forms. Lead for Tribe is something virtual inside Tribe. You have a dropdown where you select which entity should be created from an Apsis form submission.

**Workaround complexity:** When Apsis requests field definitions for "lead," Tribe returns the field definitions for whatever entity type is currently selected in the dropdown. If the customer selects "contact," we get contact fields. If they select "sailing boats" (custom entity), we get sailing boat fields.

[Erik Andersson]: The lead entity in Apsis represents their dynamic dropdown. This is super irritating, and I'm trying to simplify it for them, but that's how it is for now.

### Microsoft Dynamics (Legacy)

- Contact-only entity
- Single keyspace: **dynamics** (legacy)
- No lead support

### Microsoft Dynamics by Site Shop

- Supports leads
- Additional keyspace created: **dynamics-site-shop lead keyspace**

### Summary Table of Entity Support

| CRM System | Contact | Lead/Silhouette | Keyspaces Created |
|---|---|---|---|
| E-deal | ✓ | Silhouette | 2 |
| Efficy Corporate (FSC) | ✓ | Silhouette | 2 |
| Efficy Enterprise 12.1 | ✓ | None | 1 |
| Tribe | ✓ | Virtual (problematic) | 2 (lead is dynamic) |
| Microsoft Dynamics (legacy) | ✓ | None | 1 |
| Microsoft Dynamics Site Shop | ✓ | Lead | 2 |

---

## Pending Customer Request: Custom Keyspace Behavior for Generic Connector

### The Request

A customer using the generic JoinCX connector is requesting custom keyspace behavior:

**Current behavior:** The system generates a keyspace discriminator following the standard pattern:
```
integrations:keyspaces:<hash>:joined-cx
```

**Requested behavior:** Instead of using the standard CRM ID-based keyspace, use the email keyspace discriminator:
```
integrations:keyspaces:email
```

[Lukasz Grabowski]: They want to override the default behavior in the generic connector. Instead of using CRM IDs, they want to use email addresses because they're using a custom integration outside of Apsis and email addresses are their identifying factor.

### Issues with the Request

[Erik Andersson]: If we implement this, it would apply to every JoinCX customer, not just this one. We don't have a mechanism for customer-specific customizations to the generic connector logic without significant code changes.

**Concerns:**

1. **Lack of isolation:** Code changes would affect all JoinCX customers
2. **Business justification:** Only one customer requested this; no clear business case for a general feature
3. **Architectural mismatch:** The system is designed to isolate CRM-specific data in CRM-specific keyspaces, not merge it with email keyspaces
4. **Resource cost:** Implementation would be complex and time-consuming
5. **Data handling precedent:** Making CRM data use email keyspaces contradicts the established merging strategy that prevents CRM contacts from being mixed with email keyspaces

[Erik Andersson]: I see absolutely no business case for this if it is one customer that has asked. If we were to do it here, this will apply for every JoinCX customer, and it will be very, very complicated.

### Proposed Next Steps

Given time constraints (Lukasz needed to prepare for another meeting), the discussion was deferred:

- Lukasz will respond to the customer indicating that the request needs more investigation
- A follow-up meeting is scheduled for mid-week (Wednesday) to discuss potential solutions
- Erik will also reply to the internal email thread (in Swedish) explaining the situation
- The decision is preliminary: code changes are unlikely due to complexity, resource constraints, and lack of general business case

[Lukasz Grabowski]: I think we don't have time, we don't have resources. It's not in our interest to look at this, and I think it's not designed to handle something like this. It's a really custom thing for one customer, and I would say there's a really small chance we will do anything about it.

[Erik Andersson]: There are ways we could make this happen without coding, but that needs more focus time to explain.

---

## Key Takeaways

1. **Keyspace Discriminators are Uniqueness Mechanisms:** The hash ensures multiple CRM installations on different sections don't collide. The 8-character truncation was arbitrary but functional.

2. **Keyspace Retention is Critical:** Uninstalling and reinstalling a CRM reuses the existing keyspace. This preserves historical data and supports win-back scenarios. Keyspaces are never deleted unless explicitly requested via GDPR cleanup.

3. **Entity-Specific Keyspaces Prevent Data Corruption:** Silhouettes and leads must use separate keyspaces from contacts to prevent:
   - Unintended consent changes
   - Accidental full-contact interpretation of temporary records
   - Premature appearance of leads in CRM exports

4. **CRM Merging Strategy is Strict:** Apsis One never merges CRM-synced contacts with email keyspaces or other non-CRM keyspaces. Merges must originate from the CRM system itself, or data sync becomes unreliable. This is a deliberate architectural choice to preserve CRM as the master record.

5. **Form Submission Sync Requires Three Conditions:** Integration must be installed, integration must support form sync, and form must be explicitly configured. Event listeners are registered on configuration and removed on uninstall.

6. **Entity Support Varies Significantly Across CRMs:**
   - E-deal and FSC Corporate use silhouettes for leads
   - Efficy Enterprise 12.1 only uses contacts
   - Tribe's lead handling is problematic due to dynamic entity selection
   - Microsoft Dynamics variants differ (legacy = contact-only, Site Shop = leads supported)

7. **Tribe Integration is Architecturally Awkward:** Tribe's virtual lead entity and dynamic dropdown create complexity that Apsis's fixed entity configuration model doesn't naturally support. The workaround functions but is fragile.

8. **Customer-Specific Keyspace Customization is Not Feasible:** A request to override keyspace behavior for the generic JoinCX connector cannot be implemented without architectural rework. One customer's custom use case doesn't justify the engineering effort or the precedent it would set.

---

## Unresolved Questions and Action Items

### Action Items

- [Erik Andersson] Reply to the internal email thread (in Swedish) explaining that the JoinCX custom keyspace request requires more investigation and cannot be prioritized with current resources
- [Lukasz Grabowski / Erik Andersson] Schedule a follow-up meeting for Wednesday (mid-week) to discuss potential non-code solutions to the customer request
- [Erik Andersson] Provide more focused explanation of any workarounds that might solve the customer's problem without code changes

### Open Questions

1. Are there non-code workarounds to enable the customer to use email addresses as the primary identifier with the generic JoinCX connector?
2. Should Tribe's dynamic entity dropdown handling be refactored, or is the current workaround acceptable long-term?
3. Should there be a formal policy for customer-specific connector customizations, or should all customization requests be rejected at the architectural level?