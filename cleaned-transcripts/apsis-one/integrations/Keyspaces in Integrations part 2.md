---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One Integrations
topics: [Keyspace architecture and discriminators, Entity-specific keyspaces for leads and silhouettes, Form submission handling and CRM response processing, CRM system differences in entity types, Data merging and deduplication strategies, Form sync configuration and event listeners]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace discriminator hashing, Installation manager, Entity keyspace mappings, Form submission events, CRM integrations (E-deal, FSC Corporate, FSC Enterprise 12.1, Dynamics, Tribe), Silhouette entities, Lead entities, Profile key management]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Different Types of Connectors"]
---

## Session Overview

This session covers the architecture and mechanics of keyspace management in Apsis One integrations, with a focus on how different CRM systems handle entity types (contacts, leads, silhouettes) and how form submissions are processed and synced back to CRM systems. The discussion includes detailed explanations of discriminator generation, installation flow, entity-specific keyspace creation, and the nuances of different CRM connectors' data models and capabilities.

---

## Keyspace Discriminator Architecture

### Hash-Based Uniqueness for Multi-Instance Support

The keyspace discriminator follows a specific pattern: `integrations:keyspaces:[8-character-hash]:[crm_logical_name]`

The hash is derived from the first 8 characters of the **section discriminator hash**. [Erik Andersson]: The reason for this hash is that you can have multiple installations of a CRM system. We needed something unique for this specific section. The choice of 8 characters was somewhat arbitrary.

[Lukasz Grabowski]: This hash serves only for uniqueness purposes, not for reverse-lookup or identification. We don't need to resolve which section a keyspace belongs to from the hash itself.

[Erik Andersson]: No, it's just for some kind of uniqueness. I don't know why we took specifically 8, but it was a choice.

### Why Keyspaces Are Never Deleted

When a CRM integration is uninstalled, the keyspace is not deleted. If the same integration is reinstalled on the same section, the existing keyspace is adopted.

[Erik Andersson]: We don't delete keyspaces because there was a requirement that profiles synced to Apsis should stay in Apsis unless explicitly removed (e.g., via GDPR cleanup). If you win back customers, you want their data to remain. This is particularly important for event history—while contact data can be re-synced, historical event data would be lost if keyspaces were deleted.

---

## Installation Flow and Keyspace Creation

### Main vs. Entity-Specific Keyspaces

The installation manager's main install function creates keyspaces in a two-step process:

1. **Main keyspace** for the primary contact entity, set up using configuration values
2. **Additional entity keyspaces** for secondary entity types (leads, silhouettes) if the CRM supports them

[Erik Andersson]: We have a for loop where we create additional entities. The reason we create separate keyspaces for E-deal, for example, is because E-deal will respond with a silhouette ID from a form submission. We cannot add that silhouette ID to the normal (CRM) keyspace because it would be interpreted as a full contact, and we would find that entry in consent exports back to the CRM. Contacts and leads should be separate until there is an explicit merge request.

### Entity Keyspace Discriminator Format

Entity-specific keyspaces extend the base discriminator with the entity name:

```
integrations:keyspaces:[hash]:[crm_logical_name]:[entity_name]
```

For example, for E-deal silhouettes: `integrations:keyspaces:xxxxx:FSCCorporate:Silhouettes`

[Erik Andersson]: The reason we did not add `.contact` to be even more specific is that it would have required a lot of data migration for existing customers.

### Database Storage of Keyspace Mapping

When an integration is installed, the system generates and stores the keyspace discriminator in the database. At runtime, lookups are performed:

[Erik Andersson]: We store this in the database. So when you install, we generate: how will the keyspace discriminator look for this specific entity? For this premise installation, we have the keyspace for person on this account, section, and integration ID. When we want to interact with this, we do a lookup and say: please give me the ID for the keyspace with this discriminator, and then we utilize that for profile updates.

---

## CRM-Specific Entity Type Differences

### E-deal: Multiple Entity Types (Contacts and Silhouettes)

E-deal supports both contacts and silhouettes. When a form is submitted:
- E-deal responds with a **silhouette ID** (not a contact)
- This indicates a lead that may later become a customer

When bootstrapping E-deal, both a contact keyspace and a silhouette keyspace are created.

### FSC Corporate: Silhouettes Support

Similar to E-deal, FSC Corporate supports a silhouette entity type and receives dual keyspace setup.

### FSC Enterprise 12.1: Contacts Only

[Erik Andersson]: FSC Enterprise 12.1 doesn't have a concept of leads. They are always only creating contacts. That's why we have not needed to create additional entity keyspaces for them. When you install FSC Enterprise 12.1, you will only see the FSC Enterprise 12.1 keyspace. They will respond with contacts when we use submit form, they will give us contacts when we download, and they will only give us contacts when they send webhooks.

Tribe does support a lead entity in theory, but the implementation is problematic (see section below).

### Legacy Dynamics: Contacts Only

Legacy Dynamics (both old and generic connector version 12.1) only supports contacts. Single keyspace creation on installation.

### Dynamics by Site Shop: Leads Support

Dynamics by Site Shop supports leads, so a separate lead keyspace is created during installation.

---

## Form Submission Handling and Event Listeners

### Form Sync Prerequisites

The "sync to CRM" option is only visible on forms if:
1. An integration is installed on the section
2. The CRM supports form submissions (`can_sync_form_activities = true`)

[Erik Andersson]: You can't even select sync to CRM unless you have it installed. If the integration is installed but `can_sync_form_activities` is false, the sync to CRM option is not displayed. If you publish the form, no request is sent to integration, and no event listener is registered.

### Event Listener Registration

When a form is configured to sync to a CRM:
- Event listeners are registered for **start**, **viewed**, and **submit** events
- Event listeners are only registered for installed integrations that support form sync
- If an integration is uninstalled, its event listeners are removed

[Erik Andersson]: If you were to uninstall an integration that you are listening to, we will remove the event listeners that we have defined. Under no normal circumstances will we get actual form submit events to an integration that either isn't installed or doesn't support it.

### Form Submission Data Structure

When a form is submitted, the payload includes:
- **Profile key** from Apsis
- **Form field data** (email, first name, last name, custom fields)
- **Identifying data** under a special `fields` property: email, phone number, or existing CRM ID (if available)

[Erik Andersson]: The CRM will use to identify a person or contact either an email address, phone number, or if it has an already existing CRM ID or lead ID. Historically this has not been possible until pre-filled forms were implemented, which allows sending the CRM ID in the form submission.

---

## Form Response Processing and Entity Merging

### CRM Response Scenarios

When a form is submitted to the CRM, the CRM can respond in two ways:

#### 1. New Record Created

The CRM returns:
- Entity type (contact, silhouette, etc.)
- New record ID

[Erik Andersson]: They can say: from this form submission, for the profile with this profile key, we created a new record. This new record is of type contact and it has this record ID. Or, for E-deal, they would respond with a silhouette entity, not a contact, because they create a lead, not an actual customer.

**Processing**: The system merges the Apsis export keyspace (with the profile key) with the entity-specific keyspace (silhouette keyspace, contact keyspace, etc.) to prevent duplicate profiles:

[Erik Andersson]: We will add this record ID as the silhouette ID for the contact. We will do it via our silhouette keyspace and we will do a merge using the export keyspace with this profile key and our silhouette keyspace with the silhouette ID. So we tie those two together because otherwise you will start creating duplicates all the time.

#### 2. Existing Record Found

The CRM returns an existing record ID because:
- The same person submitted with the same email
- The person already exists in the CRM system

[Erik Andersson]: They can say: I already have a lead with this information or I already have an existing customer using this email address. They can give us whatever entity they support here, but it differs between CRM systems.

---

## Keyspace Merging Constraints

### No Merging with Email or Other Keyspaces

In the integration layer, keyspaces from CRM-synced contacts are **never merged** with email keyspaces or other non-CRM keyspaces.

[Erik Andersson]: We are only using the keyspace for the specific profile. In integration, we are never merging with any other keyspaces apart from merging the lead keyspace with the contact keyspace. But we will never merge with the email keyspace or anything else.

**Rationale**: The CRM is the authoritative data source. Merging within Apsis (outside the CRM) creates data inconsistency and doesn't propagate changes back to the CRM.

[Erik Andersson]: My opinion is that if contacts that come from the CRM system need to be merged, they should be merged from the CRM because that will reach us and then that will reach Apsis. When you merge them inside Apsis, that will not be propagated to the CRM system. You might update attributes, but the CRM system should be considered the master of the data. If you were to send an email to this person, you risk adding completely different personal data than the person actually has, which would not be super good. It's a bit sensitive whenever you do merges on anything that comes from a CRM system.

---

## The Tribe Integration Complexity

Tribe presents a unique challenge because it supports dynamic entity selection via a dropdown, but Apsis cannot dynamically add arbitrary entities without prior configuration.

[Erik Andersson]: Tribe is very annoying because they will never respond with a lead whenever we submit forms, because lead for Tribe is something virtual. Inside Tribe, you have a dropdown where you select whenever Apsis submits a form which entity should be created from this submission.

### The Problem

The issue is that Apsis doesn't support completely dynamic entities. Configuration requires knowing:
- The unique field for the entity
- The entity name

[Erik Andersson]: We don't support completely dynamic entities inside Apsis. You can't just add a new random entity because we don't know what is the unique field for this entity or the name. We need to have that configured in the connector somehow.

### Current Workaround

Apsis bootstraps a lead entity keyspace for Tribe, but the lead entity actually represents Tribe's dynamic dropdown:

[Erik Andersson]: When we ask Tribe, please give me the fields for this lead, they are giving us the values for what is selected in the dropdown. Say they have selected "I want to create contacts." Then when we make the request, give me the fields available for lead, we will get the contact data. If they have selected "sailing boats," we will get the sailing boat data. This lead entity in Apsis represents their dynamic dropdown. This is super irritating and I'm trying to simplify it for them, but that's how it is.

---

## Pending Customer Request and Decision

A customer (managed via a custom integration outside Apsis) has requested that Apsis use the **email keyspace** instead of the standard CRM keyspace for contact identification.

[Erik Andersson]: They want us to set up a custom way to override the default behaviour in the generic connector and say: instead, please install this CRM but use the email keyspace instead.

**The Issue**: This request affects only one customer but would potentially apply to all customers using the same CRM if implemented in the connector code.

[Erik Andersson]: If we were to do it here, this will apply for every customer using this CRM. If we wanted to do this in a custom way, it will be very complicated. Also, I see absolutely no business case for this if it is one customer that has asked this.

### Decision

The team decided to defer this request. [Lukasz Grabowski]: I think we don't have time, resources, or interest to look at this. It's not designed to handle this. It's some really custom thing for one customer, and I would say the chance we will do anything about it is really small.

[Erik Andersson]: There are ways to make this happen without coding, but I would want more focus time to explain them.

---

## Key Takeaways

1. **Keyspace discriminators** use 8-character hashes to ensure uniqueness across multiple CRM installations on the same section, without requiring reverse lookup capability.

2. **Keyspaces are never deleted** because contact and event history must be preserved for customer win-back scenarios and GDPR compliance.

3. **Entity-specific keyspaces** (for leads, silhouettes) are created during installation if the CRM supports multiple entity types, preventing conflation of leads and contacts during sync and consent operations.

4. **Form submissions to CRM** are only processed if the integration is installed and supports form sync; event listeners are dynamically registered and removed based on installation state.

5. **CRM-synced contacts** are strictly isolated from email and other keyspaces in the integration layer. Data merging should happen in the CRM system, not in Apsis, to maintain the CRM as the source of truth.

6. **Different CRM systems** have different entity models (some support silhouettes/leads, others don't), requiring connector-specific keyspace configurations.

7. **Tribe's implementation** is particularly complex because its dynamic entity dropdown cannot be fully automated in Apsis without prior configuration of each possible entity type.

8. **Custom integration requests** that deviate from the standard connector design should be carefully evaluated for business value before code changes are made, especially if they would affect all customers using that connector.

---

## Unresolved Questions & Action Items

- **Pending customer request**: A custom integration customer wants to use email keyspace instead of CRM keyspace. Team decision: defer and investigate non-code solutions. [Erik Andersson] to reply to customer contact indicating the team will revisit with more focused analysis next week (target: Wednesday of the following week).

- **Simplifying Tribe integration**: [Erik Andersson] mentioned ongoing effort to simplify how Tribe's dynamic entity dropdown is handled, but no timeline was established.