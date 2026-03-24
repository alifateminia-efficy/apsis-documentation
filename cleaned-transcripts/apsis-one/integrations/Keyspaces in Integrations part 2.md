---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One - Integrations
topics: [Keyspace Architecture, CRM Installations, Entity-Specific Keyspaces, Form Submissions, Silhouettes and Leads, Multi-CRM Support, Merge Strategy, Data Governance]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace Discriminator, Installation Manager, Entity Keyspaces, Silhouette Handling, Form Submission Events, CRM Lead Management, Profile Merging, JoinCX Connector]
session_type: knowledge-transfer
---

## Session Overview

This is the second part of a knowledge transfer session on keyspace architecture within the Apsis One Integrations domain. Erik Andersson walks Lukasz Grabowski through the technical design of keyspaces for CRM installations, explaining how discriminators are constructed, why multiple keyspaces are created per installation, and how form submissions and silhouette entities are handled across different CRM systems. The session covers the rationale for not deleting keyspaces on uninstall, why contacts from CRM systems should not be merged outside of the CRM, and the variations in entity support across different CRM integrations (Edeal, FSC Corporate, Dynamics, Tribe, etc.).

---

## Keyspace Discriminator Structure and Uniqueness

### Discriminator Format

The discriminator for a CRM installation keyspace follows this pattern:

```
integrations.keyspaces.<8-char-hash>.<crm-logical-name>
```

**[Erik Andersson]:** The 8-character hash is derived from the first 8 characters of a hash of the section discriminator. This is combined with the logical name of the CRM system.

### Purpose: Uniqueness, Not Reversibility

**[Lukasz Grabowski]:** What's the reason for this hash?

**[Erik Andersson]:** You can have multiple installations of a CRM system. We needed something unique for this specific section.

**[Lukasz Grabowski]:** So this is only for uniqueness, but not for identifying—we don't need to resolve this the other way around to figure out which section it is?

**[Erik Andersson]:** No, it's just for some kind of uniqueness. I don't know why we took specifically 8 characters, but it was a choice.

The hash ensures that each CRM installation gets a unique keyspace discriminator, even if the same CRM is installed multiple times on the same account. Multiple installations (from repeated install/uninstall cycles) will each have different hashes.

---

## Keyspace Persistence and Reinstallation Behavior

### Keyspaces Are Not Deleted on Uninstall

**[Lukasz Grabowski]:** What if we uninstall? Do we delete the keyspace?

**[Erik Andersson]:** No, we don't delete keyspaces.

**[Lukasz Grabowski]:** If we create again the same installation on the same section, so we adopt the existing keyspace?

**[Erik Andersson]:** Yes.

### Rationale: Data Retention and Historical Context

**[Erik Andersson]:** Even if we could in theory delete keyspaces, we don't want to because there was a requirement that profiles synced to Apsis should stay in Apsis unless explicitly otherwise said—for example, through GDPR cleanup. You want to win back customers, and then you want their data to be there. In particular for events, the contacts we can of course always sync back, but the historical event data would have been gone.

This design decision prioritizes data retention for potential customer re-engagement scenarios, especially preserving historical event data that cannot be re-downloaded from the CRM.

---

## Installation Flow and Primary vs. Entity Keyspaces

### Two Types of Keyspaces Per Installation

The installation manager creates two categories of keyspaces:

1. **Main/Regular Keyspace** — Created for the primary contact entity via the main install function
2. **Entity-Specific Keyspaces** — Additional keyspaces for secondary entities like leads or silhouettes

**[Erik Andersson]:** We set up the main keyspace for the CRM system as the first step. Then we have a loop for additional entities, used in cases where we have more entities than just the main contact one.

### When Additional Entity Keyspaces Are Created

For CRM systems that support entities beyond contacts (e.g., leads in Edeal), additional keyspaces are created.

**[Erik Andersson]:** For Edeal, we need to create a keyspace for leads. While we will not download any leads from Edeal to Apsis, Edeal will still respond with a silhouette ID from a form submission. We cannot add that silhouette ID to the normal keyspace because then it would be interpreted as a full contact. We would find that entry if we do a consent export back to the CRM. We don't want contacts and leads to be separate until there is an explicit merge request.

---

## Silhouette Entity Keyspace Design

### Discriminator Structure for Silhouettes

Silhouette keyspaces follow the same base structure as the main keyspace but with the entity name appended:

```
integrations.keyspaces.<8-char-hash>.<crm-logical-name>.<entity-name>
```

**[Erik Andersson]:** For Edeal, the discriminator is the same format up to here (the hash and CRM name), but then we add the silhouette name. You could theoretically add `.contact` to be even more specific, but we didn't do this because it would have required significant data migration for existing customers.

### Database Lookup for Entity Keyspaces

The keyspace discriminators for entity-specific keyspaces are stored in the database during installation:

**[Erik Andersson]:** When you install, we generate and store how the keyspace discriminator will look for this specific entity. For a Premise installation, we have the keyspace for person on this account, section, and integration ID. This is the keyspace with the discriminator we generate. Whenever we want to interact with it, we do a lookup and say: please give me the ID for the keyspace with this discriminator, and then we utilize that for profile updates.

### Keyspace Selection Based on Downloaded Entity Type

When downloading entities from the CRM:

**[Erik Andersson]:** When we download entities from the CRM system, we say "please give me all persons." They give us a list of entries. We know we downloaded things for persons, so we know exactly which keyspace we need to do a lookup for. We utilize that ID when we do updates.

When receiving form submissions or webhooks:

**[Erik Andersson]:** If we receive a response from the CRM system that a form was submitted and they inform us this ID belongs to a silhouette entity, we can check our mappings: OK, we need to update the silhouette ID. We check in this mapping what the keyspace discriminator is for the silhouette keyspace, then retrieve the ID for this keyspace, and utilize it when we do the silhouette ID updates from the form submission response.

---

## Contact vs. Lead Keyspace Separation Strategy

### No Merging with Email Keyspace

**[Lukasz Grabowski]:** All contacts synchronized from CRM land in Apsis in the CRM keyspace. We do not manage them.

**[Erik Andersson]:** Yes. We used to merge with the email keyspace before. However, there was a request from product to remove that because there should be able to be multiple contacts in the CRM that have the same email. That possibility disappears if we merge, so in integration we are only using the keyspace for the specific profile.

### Merging Only Within CRM-Sourced Data

**[Lukasz Grabowski]:** Can we merge profiles? Is there a situation where we merge two profiles—the same person with CRM ID and the same email?

**[Erik Andersson]:** Not in integration. We never merge with any other keyspaces apart from our own CRM-specific ones. We are merging our lead keyspace with our contact keyspace, but we will never merge with the email keyspace or anything else.

### Risk of Merging Outside the CRM System

**[Erik Andersson]:** This can happen from inside Apsis, but then we will start having weird behavior. We are not very fond when other tools start doing things with contacts that come from a CRM system unless those merge requests come from us.

If contacts from the CRM need to be merged, they should be merged in the CRM system itself:

**[Erik Andersson]:** If contacts that come from the CRM system need to be merged, they should be merged from the CRM because that will reach us and Apsis. When you merge them inside Apsis, that will not be proliferated to the CRM system. You might update attributes—if you merge a profile with an email keyspace that has a first name with a CRM profile, the CRM system should be considered the master of the data, so it will be considered out of sync. If you send an email to this person, you risk adding completely different personal data than the person actually has. It's sensitive whenever you do merges on anything that comes from a CRM system.

---

## Form Submission Event Flow

### Enabling Form Sync to CRM

Form submissions can only be synced to a CRM if the integration is installed:

**[Erik Andersson]:** You can't even select "sync to CRM" unless you have the integration installed. If the integration is installed on the section, then you see the option "sync to CRM" on the form.

### Event Listener Registration

**[Erik Andersson]:** When you sync a form to a CRM, we register event listeners for all possible event types for this specific campaign. For form campaigns, we register for the `start_viewed` and `submit` events.

### Conditional Event Listener Cleanup

**[Erik Andersson]:** If you uninstall an integration that you are listening to, we will remove the event listeners we have defined. Under no normal circumstances will we get actual form submit events to an integration that either isn't installed or doesn't support form submissions.

### CRM System Support Check

Not all CRM systems support form submissions:

**[Erik Andersson]:** Dynamics is installed on this section, but `can_sync_form_activities` is false because Dynamics doesn't support sending form submissions. This means the "sync to CRM" option is not displayed. When you publish the form, there will be no request sent to integration, and we will never register any event listener in the first place.

---

## Form Submission Payload and CRM Response Handling

### Identifying Submitted Contacts

When a form is submitted, the payload includes:

**[Erik Andersson]:** They fill in the email field, the first name field, last name field, and check "yes I have a pet." We add the profile key from Apsis. If it exists, we add any kind of identifying data. If the person filled in an email in the form, we add this under a special property called `fields`. The CRM uses email address, phone number, or an already existing CRM ID or lead ID to identify a person or contact.

This was historically not possible until the pre-filled form feature was implemented, allowing CRM IDs to be pre-populated in forms.

### CRM Response Scenarios

The CRM can respond in different ways:

**[Erik Andersson]:** They can say: from this form submission for the profile with this profile key, we created a new record. This new record is of type contact and has this record ID. Or, for Edeal specifically, they would respond with a silhouette entity—not a contact—because creating a contact would mean they created an actual customer, which they typically never do. They give us the entry with entity type silhouette and the new ID.

### Linking Form Submission Results to Profiles

**[Erik Andersson]:** We know which profile the submit happened to, so we add the record ID as the silhouette ID for the contact via the silhouette keyspace. We do a merge using the export keyspace with this profile key and our silhouette keyspace with this ID. We tie those two together because otherwise you will start creating duplicates all the time.

### Handling Existing Contacts

**[Erik Andersson]:** The CRM can say: I already have an entry. The same person submits with the same email address. They can say: I already have a lead with this information or I already have an existing customer using this email, and give us whatever entity they support here. But this differs between CRM systems.

---

## CRM-Specific Entity Support and Keyspace Configuration

### FSC Corporate: Silhouette Support

**[Erik Andersson]:** FSC Corporate has actual silhouettes that they respond with.

When installed, FSC Corporate creates both a main contact keyspace and a silhouette keyspace.

### FSC Enterprise 12.1: Contact Only

**[Erik Andersson]:** FSC Enterprise 12.1 doesn't have a concept of leads. They are always only creating contacts, so we haven't needed to create any additional entities. When you install FSC Enterprise 12.1, you will only see the FSC Enterprise 12.1 keyspace. That is the only thing we will ever use. They will respond with contacts when we use submit form, when we download, and when they send webhooks. It's nice in that aspect—they only work with contacts.

### Edeal: Silhouette Support

Edeal, like FSC Corporate, creates both a main contact keyspace and a silhouette keyspace during installation.

### Dynamics: Legacy Version (Contact Only)

**[Erik Andersson]:** The old legacy Dynamics only has the normal contact entity.

### Dynamics by Sitecore: Lead Support

Dynamics by Sitecore supports leads, so a lead keyspace is created during installation.

### Tribe: Dynamic and Complex Entity Handling

Tribe presents a special case due to its dynamic entity selection model:

**[Erik Andersson]:** Tribe in theory has a lead entity. We bootstrap a lead keyspace when you install. But Tribe is very annoying because they will never respond with a lead whenever we submit forms. For Tribe, lead is something virtual. Inside Tribe you have a dropdown where you select: whenever Apsis submits a form, which entity should be created?

The fundamental issue is that Apsis does not support completely dynamic entity configuration:

**[Erik Andersson]:** We don't support completely dynamic entities inside Apsis. You can't just add a new random entity because we don't know what the unique field is for this entity or what its name is. We need that configured in the connector somehow.

What happens with Tribe's dropdown selection:

**[Erik Andersson]:** When we ask Tribe to give us the fields for this lead, they give us the values for what is selected in the dropdown. If they selected "contacts," when we make the request "give me the fields available for lead," we get the contact data. If they selected "sailing boats," we get the sailing boat data. This lead entity in Apsis represents their dynamic dropdown. This is super irritating and I'm trying to simplify it for them, but that's how it is for the time being.

---

## Active Customer Request: JoinCX Email Keyspace Override

### The Request

A customer using a custom JoinCX integration has requested a deviation from standard keyspace behavior:

**[Erik Andersson]:** They want us to instead generate the standard CRM keyspace (with the join_join_CX discriminator), but instead give the keyspace the email keyspace discriminator. The customer is using a custom integration outside of Apsis and is using email addresses—they are not interested in CRM IDs. They want us to set up a custom way to override the default behavior in the general connector and say: instead, install this CRM but use the email keyspace.

### Why This Is Problematic

**[Erik Andersson]:** This is not true (not technically feasible). If we were to do this, it would apply for every JoinCX customer. If we wanted to do this in a custom way, it will be very, very complicated. Also, I see absolutely no business case for this if it's one customer that has asked this.

### Discussion and Deferral

**[Lukasz Grabowski]:** I think we don't have time, resources, or interest to look at this. It's not designed to handle something like this—it's a really custom thing for one customer, and I would say there's a really small chance we will do anything about it.

**[Erik Andersson]:** There are ways we could make this happen without doing any coding, but I would want more focus time to explain them, and not having you feel stressed before your meeting.

The discussion was deferred to allow for more investigation and design work without time pressure.

---

## Key Takeaways

1. **Keyspace Discriminators Are for Uniqueness**: The 8-character hash in the discriminator ensures each CRM installation gets a unique identifier, even with multiple install/uninstall cycles on the same account. The hash is not reversible—it's not meant to identify back to the section.

2. **Keyspaces Persist After Uninstall**: Keyspaces are never deleted when an integration is uninstalled. This preserves historical data (especially event data) and allows reinstallations to reuse existing keyspaces. This design supports win-back scenarios and GDPR compliance.

3. **Primary and Entity-Specific Keyspaces Are Separate**: Most CRM installations create one main keyspace for contacts. CRM systems supporting leads or silhouettes (like Edeal, FSC Corporate, Tribe) get additional entity-specific keyspaces. This separation prevents leads from being treated as full contacts and enables proper merge workflows.

4. **Never Merge CRM-Sourced Contacts Outside the CRM**: Contacts from CRM systems should only be merged within the CRM itself, not in Apsis. Merging in Apsis breaks the source-of-truth relationship with the CRM and creates sync issues. CRM systems are the master for their own data.

5. **Form Sync Requires Active Installation and CRM Support**: Forms can only sync to CRM if the integration is installed and the CRM system supports form submissions. Event listeners are only registered when both conditions are met and are cleaned up on uninstall.

6. **CRM Systems Vary Significantly in Entity Support**: 
   - **FSC Enterprise 12.1**: Contacts only
   - **FSC Corporate**: Contacts + Silhouettes
   - **Edeal**: Contacts + Silhouettes
   - **Dynamics (legacy)**: Contacts only
   - **Dynamics by Sitecore**: Contacts + Leads
   - **Tribe**: Dynamic dropdown-based entity selection (problematic for Apsis's static entity model)

7. **Tribe's Dynamic Entity Model Is a Known Pain Point**: Tribe's dropdown selection for entity creation on form submission doesn't align with Apsis's need for statically configured, pre-mapped entities. The current workaround creates unnecessary complexity.

8. **Silhouettes Require Explicit Merge Requests from CRM**: When a CRM responds with a silhouette from a form submission, Apsis links it to the contact profile. The CRM must explicitly send a merge request (via dedicated endpoints) to convert a silhouette to a full contact, preventing duplicate creation and ensuring data integrity.

---

## Unresolved Questions and Action Items

### Deferred Discussion: JoinCX Custom Email Keyspace Override

- **Status**: Deferred for deeper investigation
- **Issue**: One customer requests using the email keyspace instead of the standard CRM keyspace for JoinCX connector
- **Concerns**: 
  - Would affect all JoinCX customers if implemented
  - No clear business case for a single-customer request
  - Likely requires code changes
- **Next Steps**: Erik Andersson to explore non-code workarounds and discuss further with Lukasz Grabowski
- **Scheduled Follow-up**: Wednesday of the following week (both speakers confirmed availability)