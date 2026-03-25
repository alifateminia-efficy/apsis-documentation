---
source_file: Erik - Leads.txt
domain: Apsis One Integrations
topics: [Lead creation and gathering, Form submissions to CRM systems, Outbound flow architecture, Outbound mappings, Profile merging, Consent handling, CRM synchronization]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Outbound flow, Audience subscription worker, Batch production worker, Outbound worker, Generic connector, Efficy Enterprise 12.1, Form submissions, Outbound mappings, Profile merging, Consent updates, CRM key space, Email key space]
session_type: knowledge-transfer
subdomains: [Architecture, Outbound Flow, Efficy Enterprise 12.1, Lead creation]
---

## Session Overview

This knowledge transfer session covers the end-to-end lead generation and gathering process in Apsis One, with a focus on how form submissions are synced to CRM systems through the outbound flow. Erik Andersson walks through the technical architecture, demonstrates the practical implementation in Efficy Enterprise 12.1, and explains the critical concepts of outbound mappings, profile merging, and consent handling that ensure leads are properly created in the CRM while avoiding duplicates in Apsis.

---

## Lead Gathering Overview: Forms as the Primary Method

**Lead Definition and Collection Method**

A lead in the context of CRM synchronization is defined as data that does not yet exist in the CRM system but is collected through Apsis and then sent to that CRM system. While lead gathering can happen through multiple channels (including emails, as observed in previous sessions), **forms are the primary designed mechanism** for lead collection.

The process starts when someone submits a form that has been configured to sync with the CRM system. This is the triggering event for the entire lead creation flow.

---

## Outbound Flow Architecture

### High-Level Flow Structure

The outbound flow consists of several worker stages that process form submissions sequentially:

1. **Audience Subscription Worker** (first stage)
2. **Batch Production Worker** (second stage)  
3. **Outbound Worker** (final stage)

### Audience Subscription Worker: Validation and Enrichment

When a form submit event arrives from the Audience service via Kafka, the **audience subscription worker** is the first stop. This worker has two responsibilities:

- **Verification**: Checks whether all required data is present to enable a sync to the CRM system
- **Enrichment via Outbound Mappings**: If the submitting profile has been configured with outbound mappings, the worker enriches the event with mapped data

For example, if a form field captures a first name and outbound mappings are configured to send the first name to the CRM, the worker includes that data in the enriched event.

[Erik Andersson]: > "If you have any outbound mappings configured for your section, we will enrich the event sending with this data. So for example if the profile has a first name on it because you set up a form field that gathers that and you have configured your outbound mappings to send over the first name. We will include that and the last name and the e-mail and everything."

### Batch Production Worker: Aggregation

After passing through Kafka, the event enters the **batch production worker**. This worker aggregates multiple form submissions into a single batch, gathering up to 200 kilobytes of data for the specific section. For example, if five people submit a form, these five submissions are collected into one batch and passed to the outbound worker.

### Outbound Worker: CRM Request Sending

The **outbound worker** is where the actual request is sent to the CRM system. It sends the aggregated batch of form submissions to the CRM via the generic connector endpoint.

---

## Outbound Mappings: Two-Stage Configuration

### Distinction from Field Mappings

An important distinction must be made between **field mappings** and **outbound mappings**:

- **Field Mappings**: Data flowing FROM the CRM system TO Apsis (inbound). These represent attribute changes from the CRM.
- **Outbound Mappings**: Data flowing FROM Apsis TO the CRM system. These enrich form submit events with Apsis profile attributes that map to CRM contact attributes.

[Erik Andersson]: > "The field mappings are attribute changes from the CRM to Apsis, but the outbound mappings are like the enrichment of the form submit events from Apsis to FCC enterprise."

### Two-Stage Rocket: Form Fields → Apsis Attributes → CRM Attributes

For outbound mappings to work effectively, **both stages of mapping must be in place**:

1. **Stage 1**: Map form fields to Apsis One attributes (this is the standard form field mapping)
2. **Stage 2**: Map Apsis One attributes to CRM system attributes (this is the outbound mapping)

If only stage 1 is configured, the form fields will be captured in Apsis but won't be enriched when sent to the CRM.

[Erik Andersson]: > "The outbound mappings they work on the attribute level in Apsis one. So you take the value from the attributes in Apsis one and map that to the attributes in the CRM system. So it's from the form fields to Apsis 1 attributes, from Apsis 1 attributes to the outbound mappings. Both needs to be in place for this to fully work."

### Generic Connector: Support for Mapped Fields

The **generic connector** includes functionality to send mapped fields in the request payload. Not all CRM systems have fully implemented this, but the connector is designed to support it. This solves the problem where form field names (e.g., "motorbike", "refrigerator", "banana") may not correspond to CRM attribute names (e.g., first name, last name, email).

---

## Request and Response: The Batch Submission

### Request Structure to CRM

When the outbound worker sends a batch to the CRM system, the payload includes:

- **Event Data**: Raw form submission data (the field names as submitted)
- **Mapped Fields**: Enriched data based on outbound mappings configuration, indicating which Apsis attributes should map to which CRM fields (e.g., "This profile's first name is John, last name is Doe, email is john@example.com")

[Erik Andersson]: > "So this is what we will send to them. These fields are like the field names of the form, but these fields are the outbound mappings which says if you create a new lead from this submission, then the lead should have this first name, it should have this last name and it should have this e-mail."

### Response Structure from CRM

The CRM system responds with:

- **New Records**: Contact/lead IDs for newly created contacts
- **Matched Records**: Contact/lead IDs for existing contacts that were matched to the submission
- **CRM ID**: A unique identifier (e.g., CRM ID 234) for each record, whether newly created or matched

The CRM system determines whether a record is new or existing based on matching logic it implements. It may use data provided by Apsis (CRM ID, email, SMS, etc.) to make this determination, or it can choose to ignore the submission entirely if insufficient identifying information is provided.

[Erik Andersson]: > "The CRM system will then using this data, they will decide if it either matches or if it is something new, or they can also like completely ignore it and not do anything with that profile, in which case they will not include any data in the new records or the matched records."

---

## Profile Merging: Creating a CRM Key Space

### The Merge Necessity

After the CRM system creates or matches a contact and provides a CRM ID, Apsis must **merge the profile** to associate it with the CRM key space. Here's why:

- The form submission creates a profile in the **form key space** (and potentially the email key space)
- The CRM system now has this contact in its system with a unique **CRM ID**
- Apsis must merge these identifiers so that future updates from the CRM system can find the same profile without creating duplicates

[Erik Andersson]: > "Now that we have the profile key from the export key space and the CRM system has provided us with the CRM ID or this contact in the CRM. So now we can do the merge specifying the export key space on the profile key and our custom CRM key space and the CRM ID."

### Merge Logic: Newest Attribute Wins

In Apsis, when a merge occurs, **the attribute with the newest (latest) version wins**. This is important to understand because:

- If a form submission updates a profile field (e.g., pre-filled forms allow users to modify pre-populated data), that updated value has a newer timestamp
- After the merge, that newer value would be retained in the merged profile

However, this behavior can cause issues when CRM data should be the master data source.

### CRM Data as Master: Post-Merge Synchronization

To ensure CRM data remains the authoritative source, Apsis implements a **post-merge data pull**:

After the merge is complete, Apsis immediately downloads all data from the CRM system for the newly matched/merged contact and performs an update in Apsis One. This ensures that regardless of which value would have "won" in the merge based on timestamp, the CRM data is what ultimately exists in the Apsis profile.

[Erik Andersson]: > "But at least for now, we want the data from the CRM system to be the master data. That means that as soon as the merge is finished, we download all of the data from the CRM system for the contacts and make a update inside of Apsis one just to make sure that regardless of the outcome of the of the merge, it is the data from the CRM system that is present on the contacts."

**Caveat**: The behavior regarding which system should be master data is a design decision that could be changed in the future if needed. For now, CRM is master.

---

## Consent Handling: The Bidirectional Exception

### Consent Export on New Lead Creation

When the CRM system responds with a **new contact** (rather than a matched one), Apsis must immediately export any consent data associated with the lead. This is critical for compliance and operability:

**Use Case**: A new lead registers via form stating "I am interested in receiving technical newsletters." If this consent is not sent to the CRM system immediately:
- The lead is added to the CRM
- But with no consent data
- The lead becomes unusable because there's no permission to email, call, SMS, or contact them in any way

[Erik Andersson]: > "Consider this use case. You have a new lead that registers like they submit a form saying I am interested in receiving technical newsletters. So this lead will be added to the CRM system, but if we don't send over any consent for this new lead that was gathered from the forum or elsewhere, then you would not be able to like do anything with this lead."

### Consent Updates: The Only Bidirectional Flow

**Consent is the only data flow that is bidirectional** between Apsis and the CRM system. This is required because:

- End users can modify their consents in Apsis (by clicking unsubscribe in emails or SMS)
- These changes must be immediately sent to the CRM system
- If not sent, a future sync from the CRM could overwrite the user's opt-out with an opt-in, creating legal and compliance violations

[Erik Andersson]: > "Whenever the consent is changed inside of Apsis, we send this over to the CRM system. This is the only thing which is bidirectional in the sink."

### Consent Update Implementation

Apsis uses a **PATCH consent endpoint** to update consents on existing contacts in the CRM. This is the only endpoint in the generic connector that allows updates (there is no endpoint to update general attributes; only consents can be patched).

The process:
1. CRM responds with a new CRM ID
2. Apsis generates a consent update message for that new contact
3. The message is placed back on the outbound worker queue
4. The outbound worker calls the PATCH consent endpoint with the new CRM ID and consent data

[Erik Andersson]: > "We have a patch consent endpoint because we never update the attributes in in the generic connector like you will see here like we can fetch the entity, but we cannot update the entities."

---

## Form Publishing: The Synchronous Operation

### Only Synchronous Step in Lead Creation

When a user clicks **Publish** on a form in the Apsis UI, a **synchronous request** is made to the CRM system to create a new campaign/form object. This is the **only synchronous operation** in the entire lead creation flow.

[Erik Andersson]: > "This is synchronous. This is the only thing in the whole like lead creation that happens synchronously when you click on the publish. We directly call the CRM. We tell you immediately if that failed or not, like you can't publish the form. If our call to the CRM fails, that is a blocked operation."

**Impact**: If the call to the CRM fails, the form cannot be published, and the user receives immediate feedback.

### Asynchronous Submission and Sync

In contrast, all subsequent form submissions and CRM synchronizations happen **asynchronously** through the worker queue:
- Form submissions are queued and processed through the audience subscription worker
- Batch aggregation is asynchronous
- Outbound worker processing is asynchronous
- Merge and consent updates are asynchronous

---

## Practical Demonstration: Efficy Enterprise 12.1 Form Sync

### Integration Configuration

For form syncing to work, the CRM system must support the feature. **Efficy Enterprise 12.1** is one such system. Key configuration points:

- **Field Mappings**: Configured on the integration page; these flow FROM FEC to Apsis
- **Outbound Mappings**: Also configured on the integration page; these flow FROM Apsis to FEC for form submissions
- **CRM Sync Toggle**: The form must have the **CRM Sync** checkbox enabled; otherwise, form submissions will not trigger sync

[Erik Andersson]: > "To remember now this is does not mean that this these are mappings or attribute changes. We don't sync those to the CRM. The field mappings are attribute changes from the CRM to Apsis, but the outbound mappings are like the enrichment of the form submit events from Apsis to FCC enterprise."

### Form Setup Example

A typical form setup includes:
- Form fields (e.g., Email, First Name, Middle Name, Mobile Number)
- Standard field mappings to Apsis attributes
- A terms and conditions field (recommended to be required)
- CRM Sync enabled
- Selection of the target CRM system (e.g., "Efficy Enterprise 12.1")

### CloudWatch Logs: Verifying the Flow

After a form submission, the flow can be verified via CloudWatch logs by searching the **audience subscription worker** logs:

The logs show:
- **Profile Fields**: The identifying fields captured (e.g., email address)
- **Event Data**: The raw form submission data
- **Mapped Fields**: The enriched data from outbound mappings (e.g., first name, phone number)

The logs also show the response from the CRM system:
- **New Records**: CRM ID generated for the newly created contact
- **Matched Records**: (empty if no existing contact was matched)

[Michal Rosikiewicz - in demonstration]: "I can see we enriched it only with first name and phone number."

[Erik Andersson]: "Exactly. So here now you can see that from this profile that was submitted, we have, we have the e-mail address. So we added that as an identifying field under the profile fields. We don't yet have any CRM contacts. We have the event data because like with everything you have filled in."

### Profile Merging in Action

After submission, the profile can be searched in Apsis using **two key spaces**:

1. **Email Key Space**: Using the email address submitted in the form
2. **CRM Key Space**: Using the CRM ID returned by the CRM system (e.g., "FEC Enterprise 2" key space)

Both should resolve to the same profile, confirming the merge was successful.

[Erik Andersson]: > "Now you find that in our FCC Enterprise 2 key space. That means that you can find of course from the e-mail that you created from the form submission, but you can also find it in our FCC Enterprise 2 key space using the CRM ID. This also means that for any future updates from the CRM system, this will not lead to a duplicate in Apsis."

### CRM Data Updates After Merge

The logs also show a **consent update event** being generated and queued. This is the initial consent export for the new lead. The profile shows:
- **Consent Subject**: The mapped consent/topic name (e.g., "profile_QA")
- **Status**: Initial opt-in state
- **Target Topic**: Maps to the configured topic in the integration (e.g., "FEC Enterprise 12.1 topic")

---

## Handling of Outbound Mappings: Enterprise 12.1 Gap

### Partial Implementation in Enterprise 12.1

During the demonstration, it was discovered that **Efficy Enterprise 12.1 does not fully utilize the mapped fields** sent by Apsis. While Apsis correctly sends first name and phone number in the request based on outbound mappings, FEC does not populate those fields in the created contact.

[Erik Andersson]: > "So the unfortunate thing about Enterprise 12.1 is that they are not actually fully utilizing our mapped fields, we realized yesterday."

[Michal Rosikiewicz - from logs]: "Yeah, yeah, true. Also a phone number I think is missing."

### Apsis System Status: Working as Designed

However, **Apsis is working correctly**. The system successfully:
- Receives the form submission
- Applies outbound mappings
- Sends enriched data in the request to FEC
- Receives the CRM ID response
- Merges the profile

The gap is on the FEC side for receiving and processing the mapped field data. Erik committed to follow up with the FEC team regarding this implementation gap.

[Erik Andersson]: > "However, the important part as far as we are concerned is that you could see that we did provide the first name and the phone number based on the outbound mappings in the request data. If if they are not actually fully utilizing them that that's bad, I'm going to ping them about it, but the system is working as far as we are concerned, like we are providing the data in the requests."

---

## CRM Data as Master: Post-Merge Overwrite

### Behavior When CRM Returns Empty Values

After the merge, Apsis downloads all data from the CRM for the contact. If the CRM returns empty values for fields that were submitted in the form, Apsis interprets this as "the CRM has cleared this data" and accordingly **clears those fields in Apsis as well**.

This ensures data consistency and that the CRM remains the master source.

[Michal Rosikiewicz - from observation]: "Yeah, it actually also updated this profile and it removed first name, middle name and yeah, we can have only last name that was on the forum."

[Erik Andersson]: > "And this is because at the end of the merge process we download the data from the CRM, but because we we see that like because you have set up a mapping for these fields and we download the middle name and the mobile number and we get an empty value from the CRM then we interpret this as like the user has cleared this out in the CRM, so we clear it out inside of Apsis as well."

### Design Decision: Flexibility for Future Change

This behavior—where CRM data always wins—is a **design decision that could be changed** if different behavior is desired. Erik notes that changing this would be trivial to implement if needed in the future.

---

## Email Key Space Uniqueness: Single Email Per Profile

### Implication of Email Key Space Creation

When a form is submitted with an email address, that email address becomes a unique identifier in the **email key space**. This creates a constraint:

**Multiple people cannot submit forms with the same email address** and create separate leads. If the same email is submitted again:
- The system recognizes the email already exists in the email key space
- It updates the existing profile rather than creating a new one
- This is the expected behavior for form submissions

[Erik Andersson]: > "The downside of this of course is that because you are creating the profile in the e-mail key space, this e-mail address has now essentially been rendered unique, meaning that the requirements to have the same e-mail address on multiple profiles in the CRM can be a bit tricky."

### CRM-Side Uniqueness Not Enforced

However, **the CRM system can independently create multiple contacts with the same email address**. Apsis does not merge on email in the normal CRM sync flow, so:

- A CRM system could have 10 contacts with the same email
- Apsis would not automatically merge them
- But Apsis form submissions with that email would update the single Apsis profile associated with that email

[Erik Andersson]: > "But of course you could create completely new contacts from the CRM system that has the same e-mail e-mail address because we don't merge on the e-mail key space in our normal sync flow."

**Note**: Erik noted this is more of a form and email behavior question than an integration question, as the behavior is tied to how Apsis handles form submissions and email uniqueness.

---

## Repeat Submissions: Matched Record Flow

### Behavior on Re-submission with Same Email

If a user submits a form again using the same email address:

1. **First submission**: Creates a new profile in email key space, CRM creates a new contact, merge establishes CRM key space
2. **Second submission**: Same email matches existing profile in email key space, form data updates the profile
3. **CRM sync**: The CRM receives a second form submit, but this time it has the CRM ID from the previous submission
4. **CRM response**: Returns a **matched record** instead of a new record (because it can match based on the CRM ID)
5. **Merge and sync**: Same merge process occurs, CRM data is downloaded again, empty values overwrite previous form data

[Michal Rosikiewicz - questioning]: "But when we will submit this uh form again with the same e-mail where they will receive any events on this profile like this demo link of it or something like this."

[Erik Andersson]: > "But now we would get a matched record from the CRM system instead. And then again we will do our merge flow and then download the empty data from the CRM system. So this should be erased in a not too long time."

---

## Generalization: Other Event Types and Future Expansion

### Current Scope: Form Submissions Only

Currently, **outbound mappings are only applied to form submit events**. The check in the code is simple:

```
if (event.type == "form_submit") {
  add outbound_mappings_data;
}
```

### Future Expansion Possibility

The generic connector is designed to support outbound mappings for **any event type**, including:
- Email submissions
- Task notes
- Marketing automation flow events
- Other custom events

If business requirements change, adding outbound mappings to other event types would be trivial.

[Erik Andersson]: > "The generic connector does support it for anything. We can add the outbound mappings data for e-mail submits or task notes or MA flow things or whatever."

---

## Supported CRM Systems for Lead Generation

The following CRM systems currently support lead generation/form sync:

- **Efficy Enterprise 12.1**: Supports lead/contact creation, but has gaps in consuming mapped fields
- **Tribe**: Reportedly fully implements outbound mappings
- **Efficy Enterprise** (other versions): Supported
- **E-deal (Efficy Corporate)**: Supports lead generation

---

## Key Takeaways

1. **Lead gathering in Apsis is primarily form-based** and flows through the outbound synchronization pipeline: audience subscription worker → batch production worker → outbound worker → CRM system.

2. **Outbound mappings require two-stage configuration**: form fields → Apsis attributes (stage 1) AND Apsis attributes → CRM attributes (stage 2). Both must be in place for enriched data to reach the CRM.

3. **The merge process is critical for deduplication**: After the CRM provides a CRM ID, Apsis merges the form submission profile with a CRM key space to ensure future updates don't create duplicates.

4. **CRM data is the authoritative master**: After merging, Apsis immediately downloads data from the CRM and overwrites local values, ensuring CRM data always wins—even if an Apsis attribute had a newer timestamp.

5. **Consent is the only bidirectional flow**: Consent is exported to the CRM for new leads and is the only data type that can be updated via a PATCH endpoint. All consent changes in Apsis are pushed to the CRM to prevent legal/compliance violations.

6. **Form publishing is the only synchronous operation**: All other steps (submission, batching, syncing, merging) happen asynchronously via worker queues. Form publication is blocked if the CRM call fails.

7. **Email uniqueness in the form flow constrains multiple leads**: Once a profile is created in the email key space, that email becomes unique for Apsis purposes. Repeat submissions with the same email update the existing profile rather than creating new leads.

8. **CRM ID uniqueness prevents future duplicates**: By merging on the CRM key space and always using it for subsequent syncs, Apsis ensures that CRM updates find the correct profile without creating duplicates.

9. **Not all CRM systems fully consume mapped fields**: Efficy Enterprise 12.1 currently doesn't fully utilize the mapped field data sent by Apsis, despite Apsis correctly sending it. This is a CRM system gap, not an Apsis issue.

10. **Outbound mappings are currently form-submit-only but extensible**: The architecture supports applying outbound mappings to any event type in the future with minimal code changes.

---

## Unresolved Questions and Action Items

1. **FEC Enterprise 12.1 Mapped Fields Gap**: Erik committed to contacting Efficy to understand why mapped fields (first name, phone) are not being populated in created contacts despite being sent in the request payload.

2. **Folder Location for Architectural Diagrams**: Lukasz and Erik discussed adding Erik's outbound mappings diagram to the shared architecture diagram folder but did not finalize a location. The diagram should be added to the "Knowledge Transfer" or "Integration" architectural diagrams area.

3. **CRM Data Master vs. Apsis Master**: The decision that CRM data always wins post-merge is noted as changeable in the future if different behavior is desired. No active change was discussed, but this is a design decision to revisit if needed.

4. **Form Re-submission Behavior**: The exact behavior of updating pre-filled forms with re-submissions requires clarification in documentation, as it involves the interaction between form logic, profile key spaces, and CRM matching logic.