---
source_file: Keyspaces in Integrations part 2.txt
domain: Apsis One Integrations
topics: [Keyspace Architecture, Discriminator Generation, Entity-Specific Keyspaces, Form Submission Handling, CRM System Variations, Lead vs Contact Entity Management, Profile Merging Strategy, Event Listener Registration]
speakers: [Erik Andersson, Lukasz Grabowski]
key_components: [Keyspace System, Discriminators, Entity Keyspaces, Installation Manager, Form Sync, Event Listeners, Profile Merging, CRM ID Management, Silhouette Entities]
session_type: knowledge-transfer
subdomains: [E-deal (Efficy Corporate), Efficy Enterprise 12.1, Microsoft Dynamics, Tribe]
---

## Session Overview

This session covers the core architecture of keyspaces in the Apsis One Integrations system, focusing on how multiple CRM installations are uniquely identified and how different entity types (contacts, leads, silhouettes) are managed through distinct keyspaces. The discussion covers discriminator generation logic, the rationale behind not deleting keyspaces on uninstallation, how form submissions route to appropriate keyspaces, and important variations across different CRM systems (E-deal, Efficy Enterprise, Tribe, Microsoft Dynamics). The session also briefly touches on a pending customer request regarding custom keyspace behavior that requires further investigation.

---

## Keyspace Discriminator Design and Uniqueness

### Discriminator Structure

The discriminator format follows this pattern:

```
integrations:keyspaces:[8-character-hash]:[CRM-logical-name]
```

[Erik Andersson]: The discriminator is structured as `call matches 1 integrations, keyspaces, lots of random characters`. These random characters are the **first 8 characters of the hash of the section discriminator**, then we add the logical name of the CRM.

### Purpose of the Hash Component

[Lukasz Grabowski]: What's the reason for this hash?

[Erik Andersson]: Because you can have multiple installations of a CRM system. We needed something unique for this specific section. It's just for uniqueness—we don't need to resolve it in reverse to identify which section the keyspace belongs to.

[Lukasz Grabowski]: So this is only for uniqueness, not for identifying. We don't need to resolve this the other way around.

[Erik Andersson]: Exactly. It's just for some kind of uniqueness. I don't know why we took specifically 8 characters, but it was a choice.

### Multiple Installations on Same Account

[Erik Andersson]: In my test account, I have installed and uninstalled and installed and uninstalled multiple times. There are a lot of keyspaces, but every CRM system works the same way. They all have their main profile entity.

---

## Keyspace Persistence and Reuse on Reinstallation

### Why Keyspaces Are Not Deleted

[Lukasz Grabowski]: When we uninstall, do we delete the keyspace?

[Erik Andersson]: No, we don't delete keyspaces.

[Lukasz Grabowski]: So if we create again the same installation on the same section, we adopt the existing keyspace?

[Erik Andersson]: Yes. Even if we theoretically could delete keyspaces, we don't want to because there was a requirement that profiles synced to Apsis should stay in Apsis unless explicitly removed (e.g., via GDPR cleanup).

### Business Rationale for Keyspace Retention

> We want to win back customers for example, and then you want their data to be there. In particular for events, the contacts we can of course always sync back, but the historical event data would have been gone.

The historical event data tied to a keyspace becomes inaccessible if the keyspace is deleted, which could impact customer re-engagement campaigns.

---

## Installation Flow and Keyspace Setup

### Main and Entity Keyspaces

The installation flow is handled in the installation manager's main install function. There are two types of keyspaces created:

1. **Main keyspace**: Created for the CRM system itself, using just the configuration
2. **Entity keyspaces**: Created for additional entities beyond the main contact entity

[Erik Andersson]: We have a little for loop here where we have additional entities. This is in cases where we have more entities than just the main contact one.

### Rationale for Multiple Entity Keyspaces

E-deal provides a specific example: Although E-deal will not download any leads to Apsis, they will respond with a silhouette ID from a form submission. We cannot add that silhouette ID to the normal (contact) keyspace because it would be interpreted as a full contact. Then, if we do a consent export back to the CRM, we would include this silhouette, which is not desired. **Contacts and leads should be separate until there is an explicit merge request.**

---

## Keyspace Separation: Contacts vs. Silhouettes/Leads

### Why Silhouettes Need Their Own Keyspace

[Lukasz Grabowski]: You mean the CRM-created keyspace which we discussed earlier—the dedicated keyspace for the installation. We cannot use this for silhouettes?

[Erik Andersson]: Exactly. In theory you could, but you will have issues, especially for consents. If there is a CRM ID and a consent change occurs, we have listeners for this and would start sending consents for silhouettes back from Apsis—which is not within scope. We should only send form submissions and register the silhouette IDs on the profile.

The CRM system will then either:
- Confirm they created an actual person from this lead and send a merge request, or
- Indicate the lead didn't lead anywhere and request deletion

---

## Form Submission Handling and Entity Routing

### Form Sync Registration

When you enable sync to CRM on a form (which is only possible if the integration is installed), event listeners are registered for that specific campaign. For form campaigns, these listeners register for "start viewed" and "submit" events.

[Lukasz Grabowski]: So if there is a form for the customer and it's public, there is a submission. Then you send a submit event to Apsis, but then you need to identify if this customer on this section has integration enabled, and only in this situation do you send information to the CRM?

[Erik Andersson]: You can't even select "sync to CRM" unless the integration is installed.

### Integration Install Requirement

If an integration is not installed or doesn't support form syncing, the "sync to CRM" option is not displayed on the form. When you publish the form, no request is sent to integration and no event listeners are registered. This prevents form submissions from reaching integrations that aren't active.

> Under no normal circumstances will we get actual form submit events to an installation to an integration that either isn't installed or doesn't support it.

### Listener Lifecycle

If you uninstall an integration you are listening to, the event listeners are removed.

---

## Form Submission Payload and Response Handling

### Submitted Data Structure

When a form is submitted:
- Profile key is added from Apsis
- Identifying data is included (if available): email address, phone number, existing CRM ID, or lead ID
- Form field values are submitted under a special property called `fields`

[Erik Andersson]: The CRM will use identifying data such as email address, phone number, or if it has an already existing CRM ID or lead ID. Historically this was not possible until pre-filled forms were implemented.

### CRM Response Variations

The CRM can respond with different outcomes:

1. **New record created**: They provide the record type and ID
   - E-deal example: Response type is "silhouette" (not "contact"), with a new ID
   
2. **Existing record found**: They indicate they already have a record with this email/information and provide that record

> They can say yeah, from this form submission, for the profile with this profile key, we created a new record. This new record is of type contact and it has this record ID. For E-deal, they would respond with the silhouette entity, not contact, because that means they created an actual customer—which they typically never do from form submissions.

### Handling Silhouette Responses

When E-deal responds with a silhouette entity:
- The silhouette ID is added to the contact profile
- This is done via the **silhouette keyspace** 
- A **merge is performed** using the export keyspace (with the profile key) and the silhouette keyspace (with the silhouette ID)
- This ties the two together to prevent duplicate creation if the same person submits again with the same email

---

## CRM System-Specific Entity Variations

### Efficy Enterprise 12.1 (E-deal Generic Connector)

[Erik Andersson]: Efficy Enterprise 12.1 doesn't have a concept of leads. They are always only creating contacts. So we have not needed to create any additional entity keyspaces for them.

When you install Efficy Enterprise 12.1:
- Only one keyspace is created: the main Efficy Enterprise 12.1 keyspace
- Responses always contain contacts only (form submissions, downloads, webhooks)
- The CRM must handle the internal separation of temporary vs. permanent contacts

### Microsoft Dynamics

#### Legacy Dynamics
- Supports only contact entity
- No lead support

#### Dynamics by Siteshop
- Supports leads
- A separate lead keyspace is bootstrapped on installation

#### Dynamics (generic version)
- Contacts only, same as legacy

### E-deal (Efficy Corporate)

[Erik Andersson]: Efficy Corporate has their silhouette setup. Whenever you install E-deal or Efficy Corporate, we will bootstrap both the contact keyspace and the silhouette keyspace.

- Creates both contact and silhouette keyspaces
- Responds with silhouettes for form submissions (not full contacts)
- Supports explicit merge requests

### Tribe

Tribe has a particularly complex behavior:

[Erik Andersson]: Tribe has a lead entity in theory, so we bootstrap a lead entity keyspace when you install. But Tribe is very annoying because they will never ever respond with a lead whenever we submit forms. For Tribe, "lead" is something virtual—you have a dropdown where you select what entity should be created from the form submission.

**The Core Issue**: The Tribe dropdown is dynamic—customers can select from 10+ different entities. Apsis doesn't support completely dynamic entities because we need to know:
- The unique field for the entity
- The name of the entity
- How to configure it in the connector

[Erik Andersson]: When we ask Tribe "please give me the fields for this lead," they give us the values for whatever is selected in the dropdown. So if they've selected "create contacts," we get contact data. If they've selected "sailing boats," we get sailing boat data. The lead entity in Apsis represents their dynamic dropdown, which is super irritating. I'm trying to simplify it for them, but that's how it is for the time being.

---

## Profile Merging Strategy and Constraints

### No Merging Between Keyspaces in Integration

[Lukasz Grabowski]: Can we merge profiles? For example, merge two profiles where one has a CRM ID and the other has the same email?

[Erik Andersson]: Not in integration. We never merge with any other keyspaces apart from... within the integration flow, we are merging our lead keyspace with our contact keyspace. But we will never merge with, say, the email keyspace or anything else.

### Why External Merging Is Problematic

[Erik Andersson]: This can happen from inside Apsis, I guess, but then we will start having weird behavior. We are not very fond when other tools start doing things with contacts that come from a CRM system unless those merge requests come from us.

If contacts that come from the CRM system need to be merged, they should be merged in the CRM system first, not in Apsis:

> If contacts that come from the CRM system need to be merged, then they should be merged from the CRM because then that will reach us and then that will reach Apsis. Because when you merge them inside of Apsis, that will not be proliferated to the CRM system.

### Data Consistency Risks

[Erik Andersson]: If you merge a profile that has an email keyspace entry with a first name with a CRM profile that has different data, the CRM system should be considered the master of the data. It will be considered out of sync. If you were to send an email to this person, you risk adding completely other personal data in the email than the person actually has, which would not be super good. So it's a bit sensitive whenever you do merges on anything that comes from a CRM system.

---

## Keyspace Database Storage and Lookup

### Entity-Specific Keyspace Configuration

Keyspace discriminators for specific entities are stored in the database during installation:

[Erik Andersson]: We store this in the database. So when you install, we generate how the keyspace discriminator will look for this specific entity. For a premises installation, we have the keyspace for person on this account section and integration ID—that will be the keyspace with this discriminator.

### Dynamic Keyspace Lookup

When interacting with a specific entity:
1. We look up the keyspace ID by discriminator
2. When downloading entities from the CRM, we know the entity type and can look up the correct keyspace
3. When receiving form responses, we check the entity type and use the corresponding keyspace

[Erik Andersson]: When I download entities from the CRM system, they give me a list of entries. We know we downloaded things for persons, so we know exactly which keyspace we need to do a lookup for. We utilize that ID when we do updates.

If the CRM responds that a silhouette was created:
- We check the entity type: "silhouette"
- We look up the silhouette keyspace discriminator in our mappings
- We retrieve the keyspace ID and use it for the silhouette ID updates

---

## Pending Customer Request: Custom Keyspace Behavior

[Note: This section was discussed as an unresolved item requiring further investigation]

### The Request

A customer (using JoinCX integration) is requesting custom keyspace behavior. Instead of the system generating the standard CRM keyspace, they want to use the email keyspace. They are using a custom integration outside of Apsis and are interested in email addresses rather than CRM IDs.

[Erik Andersson]: They want us instead of generating this keyspace where we say we are generating this JoinCX keyspace because it's joined this their joined CX CRM system. They would want us to give the keyspace the email keyspace discriminator because the customer is using a custom integration outside of us and they are using the email addresses there. They are not interested in the CRM IDs.

### Why This Is Problematic

[Erik Andersson]: If we were to do it here in the general connector, this will apply for every JoinCX customer. If we wanted to do this in some custom way, this will be very, very complicated. Also, I see absolutely no business case for this if it is one customer that has asked this.

[Lukasz Grabowski]: I think we don't have the time, we don't have resources, and it's not in our interest to look at this. It's not designed to handle such a custom thing for one customer, and I would say this is a really small chance that we will do anything about it.

### Possible Solutions (To Be Investigated)

[Erik Andersson]: There are ways to solve this, but I would really prefer not to do any changes in the code because it's going to be ugly.

**Status**: Requires more focus time to explain solutions. Scheduled for Wednesday follow-up meeting.

---

## Key Takeaways

1. **Keyspace discriminators are uniquely scoped** to individual CRM installations using an 8-character hash + CRM logical name, ensuring multiple installations on the same account don't conflict.

2. **Keyspaces are never deleted** on integration uninstallation to preserve historical event data and enable customer re-engagement. If the same integration is reinstalled, the existing keyspace is reused.

3. **Entity-specific keyspaces** (leads, silhouettes) are created separately from the main contact keyspace to prevent data model conflicts, especially for operations like consent exports and merges.

4. **Form submissions are only processed** if the integration is installed and the "sync to CRM" option is explicitly enabled on the form. The form field type determines which keyspace is used for the response.

5. **CRM systems vary significantly** in their entity models:
   - Efficy Enterprise 12.1: Contacts only
   - E-deal/Efficy Corporate: Contacts + Silhouettes
   - Tribe: Dynamic entity selection (complex, requires workaround)
   - Microsoft Dynamics: Contacts, with some versions supporting leads

6. **Profile merging in Apsis should be avoided** for CRM-originated contacts. Merges should happen in the source CRM system to maintain data consistency and ensure changes propagate back to Apsis.

7. **CRM system is always the master** of contact data. Merging CRM contacts with Apsis-native profiles (e.g., email keyspace profiles) risks data inconsistency and incorrect personalization.

---

## Unresolved Questions and Action Items

- **Custom Keyspace Behavior Request**: Customer wants to use email keyspace instead of CRM keyspace for JoinCX integration. Requires further investigation of non-code solutions. Scheduled for Wednesday follow-up between Erik and Lukasz.

- **Tribe Entity Complexity**: The dynamic entity dropdown behavior in Tribe requires ongoing investigation to simplify the user experience. No timeline committed.