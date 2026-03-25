---
source_file: Erik - Leads.txt
domain: Apsis One Integrations
topics: [Lead gathering and creation, Form submissions to CRM systems, Outbound mappings, Profile merging, Consent synchronization, Batch processing, Event-driven architecture]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Audience subscription worker, Batch production worker, Outbound worker, Kafka, Generic connector, CRM systems (Efficy Enterprise 12.0, Efficy Enterprise 12.1, Tribe, E-deal), Form submission flow, Merge process, Consent endpoint]
session_type: knowledge-transfer
subdomains: [Architecture, Lead creation, Efficy Enterprise 12.1 Integration, Duplicate profiles in Apsis]
---

## Session Overview

This knowledge transfer session covers the lead gathering and creation workflow in Apsis One, with a focus on how form submissions are synced to CRM systems through the outbound integration flow. Erik Andersson walked through the architectural components involved—from form submission through audience subscription workers, batch production, and the outbound worker—and then demonstrated the process live in a sandbox environment using Efficy Enterprise 12.1. Key topics include the distinction between inbound field mappings and outbound mappings, the merge process that links profiles across key spaces, and the critical consent synchronization step for new leads.

---

## Lead Gathering Architecture Overview

### Outbound Flow Fundamentals

All lead gathering in Apsis One happens through the **outbound flow**. The primary mechanism for customers to gather leads is through **forms**, though the same process applies to other event types (emails, tasks, etc.).

A **lead** in the CRM system context is a new record that does not yet exist in the CRM but is collected through Apsis One and sent to the external CRM system.

### Form Submission to Event Processing

The lead creation flow begins when a user submits a form that is configured to sync with a CRM system:

1. **Form submission event**: When someone submits a form, an event is generated and sent from the Audience module to the integration system.

2. **Audience subscription worker**: This is the first step in the outbound flow. It verifies that all required data is present for the sync to proceed.

3. **Event enrichment with outbound mappings**: If outbound mappings are configured for the section, the event is enriched with mapped data. For example, if a form field captures a first name and outbound mappings are configured to send the first name to the CRM, that data is included in the event payload.

4. **Kafka queue**: Events are briefly buffered in Kafka before being passed to the next processing stage.

5. **Batch production worker**: This component gathers multiple events (up to 200 kilobytes total) from the same section. For example, if five people submit a form, the five form submissions are batched into one large request.

6. **Outbound worker**: This worker sends the batch request to the CRM system via the generic connector endpoint.

---

## Outbound Mappings vs. Inbound Field Mappings

### Critical Distinction

[Erik Andersson]: There is important confusion to clarify between **field mappings** and **outbound mappings**:

- **Field mappings** (inbound): These map data flowing FROM the CRM system TO Apsis One. They represent attribute changes in the CRM that synchronize down to profiles in Apsis One.

- **Outbound mappings** (outbound): These map data flowing FROM Apsis One TO the CRM system. They enrich form submit events with profile attribute values from Apsis One, allowing the CRM system to know which attributes from Apsis correspond to which fields in the CRM.

[Erik Andersson]: "The field mappings are attribute changes from the CRM to Apsis, but the outbound mappings are like the enrichment of the form submit events from Apsis to FCC enterprise."

### Two-Stage Mapping Process

Form field mapping in Apsis One is a two-stage process:

1. **Stage 1 (inbound)**: Form fields → Apsis One attributes
   - User maps form field (e.g., "Full Name") to an Apsis attribute (e.g., "first_name", "last_name")

2. **Stage 2 (outbound)**: Apsis One attributes → CRM system fields
   - User configures outbound mappings to tell the CRM system: "When syncing, map the Apsis 'first_name' attribute to your 'FirstName' field"

Both stages must be configured for the full lead creation flow to work properly.

### Purpose of Outbound Mappings

Outbound mappings solve the problem of form field naming ambiguity. If a form has fields named "motorbike", "refrigerator", and "banana", without outbound mappings the CRM system would not know which Apsis attribute should map to which CRM field (first name, last name, email, etc.).

[Erik Andersson]: "The mapped field helps you with this. Not all CRM systems have actually implemented that, but we have that functionality in the CRM system or in the general connector, sorry."

---

## Request and Response Cycle with CRM Systems

### Outbound Request Structure

When the outbound worker sends a batch to the CRM system, the request includes:

- **Profile identifier**: The Apsis profile key
- **Event data**: Raw event fields from the form submission
- **Mapped fields**: Values from Apsis attributes that have been mapped via outbound mappings (e.g., first_name: "John", last_name: "Doe", email: "john@example.com")

### CRM System Response

The CRM system responds with:

- **Newly created records**: CRM IDs for new contacts/leads created from the submission
- **Matched records**: CRM IDs for existing contacts/leads that matched the submitted data
- **No match**: If the CRM cannot identify a matching contact (e.g., no email, no CRM ID, no identifying data), it may return no record at all

[Erik Andersson]: "The CRM system will respond with the CRM IDs or the lead IDs for anything that is either newly created or if it was already existing in the CRM system."

The CRM uses the provided data (CRM ID, email, or other identifiers) to determine whether the submitted profile is new or matches an existing record. This decision is entirely under CRM control.

---

## Profile Merging Process

### Merge Trigger

After receiving a response from the CRM system with CRM IDs, Apsis One must trigger a **merge** operation. This merge connects the profile across different key spaces:

- **Source key space**: The Apsis key space where the profile was originally created (usually email key space for form submissions)
- **Target key space**: A custom CRM key space created for the external CRM system
- **Merge identifiers**: The Apsis profile key maps to the CRM ID returned by the CRM

[Erik Andersson]: "So now that we have the profile key from the export key space, the CRM system has provided us with the CRM ID or this contact in the CRM. So now we can do the merge specifying the export key space on the profile key and our custom CRM key space and the CRM ID."

### Merge Winner Logic

When a merge occurs in Apsis One, the attribute with the **newest/latest version wins**. This can create an issue: if a form submission updates a profile attribute (e.g., via pre-filled forms), that newly-updated attribute will win the merge against older CRM data.

### CRM Data as Master - Post-Merge Download

To ensure CRM data remains the authoritative source, immediately after the merge completes, Apsis One:

1. Downloads all current data for the contact from the CRM system via API
2. Updates the merged profile in Apsis One with the downloaded CRM data

[Erik Andersson]: "But at least for now, we want the data from the CRM system to be the master data. That means that as soon as the merge is finished, we download all of the data from the CRM system for the contacts and make a update inside of Apsis one just to make sure that regardless of the outcome of the of the merge, it is the data from the CRM system that is present on the contacts."

This ensures that regardless of what the form submitted or the merge outcome, the CRM's data is always the final truth in Apsis One.

---

## Consent Synchronization for New Leads

### The Problem

When a new lead is created in the CRM from a form submission, it arrives with no consent information. If consents are not sent to the CRM, the lead cannot be contacted via any channel, making the lead unusable.

[Erik Andersson]: "You have a new lead that registers like they submit a form saying I am interested in receiving technical newsletters. So this lead will be added to the CRM system, but if we don't send over any consent for this new lead that was gathered gathered from the forum or elsewhere, then you would not be able to like do anything with this lead because you wouldn't be about to contacted use e-mail, you could not call them or send them a text because you don't have any consent to sending text messages."

### Automatic Consent Export on New Records

When a new record is detected in the CRM response:

1. Apsis One generates a consent export event for all consents that have configured mappings
2. These consent exports are sent to the CRM system using the newly-obtained CRM ID
3. The CRM system updates the contact with the consent values from Apsis One

[Erik Andersson]: "As soon as we notice that we receive a new record, then we do an export in Apsis for all of the concerns that we have a mapping for. And we send that to the CRM system."

### Consent Update Endpoint

Consent updates use a dedicated **PATCH consent endpoint** on the generic connector, because:

1. This is a legal/compliance requirement
2. End users can modify their consents in Apsis One (unsubscribe links in emails/SMS)
3. Those changes must immediately sync to the CRM to prevent re-contacting opted-out users

[Erik Andersson]: "We never update the attributes in in the generic connector like you will see here like we can fetch the entity, but we cannot update the entities. We can, however, update the consents on entities because here is a legal requirements because the end users can modify their consents inside of apps is when they click on unsubscribe in emails or in SMS and that means that we must send this change to the CRM system because otherwise we will be in a situation where the customer says like hell no, you are not allowed to contact me in emails, but the next time there would be a a sync from the CRM system, we would overwrite their opt out with the opt in in the CRM system and then we would be in all kinds of trouble."

### Bidirectional Consent Sync Only

This is the **only bidirectional sync** in the lead creation flow. All other data flows from CRM to Apsis One (inbound) or Apsis One to CRM (outbound), but consents must flow both directions to maintain legal compliance.

---

## Practical Demonstration: Form Setup and Sync

### Prerequisites for CRM Sync

The CRM system must support form syncing for lead gathering. Efficy Enterprise 12.1 is one such system that supports this feature.

### Form Configuration Steps

1. **Create a form** with fields to capture lead information
2. **Map form fields to Apsis attributes**: Configure which form fields populate which Apsis One attributes
3. **Configure outbound mappings**: Map Apsis attributes to CRM field names
4. **Enable CRM sync**: Check the "CRM sync" checkbox on the form configuration

[Erik Andersson]: "You need to select this if, if, if, if the customer wants to gather things from this, you need to select it. Otherwise we will not listen to the uh uh submit the ones."

### Publishing a Form (Synchronous Operation)

When the form is published, Apsis One makes a **synchronous call** to the CRM system to create a corresponding campaign/form record.

[Erik Andersson]: "This is synchronous. This is the only thing in the whole like lead creation that happens synchronously when you click on the publish, we directly call the CRM. We tell you immediately if that failed or not, like you can't publish the form. If our call to the CRM fails, that is a blocked operation."

If the CRM call fails, the form cannot be published. This is the only blocking operation in the entire lead sync flow.

### Form Submission Observation

After a form is published and submitted:

1. Submit event appears in **audience subscription worker logs** (within a few seconds)
2. Event is batched and processed by **batch production worker**
3. CRM response can be viewed in **outbound worker logs**
4. Response shows:
   - **Profile fields**: The identifying data from the submission (email, etc.)
   - **Mapped fields**: Enriched attribute values from outbound mappings
   - **New/matched records**: CRM IDs returned by the CRM system for this profile

[Erik Andersson] [observed in logs]: "For this profile that was submitted, we have the profile key from Apsis. They have created a new contact from this FC Enterprise. FC Enterprise does not utilize leads, they use contacts as leads and they generated a CRM ID from this."

### Verifying Profile Merge Across Key Spaces

After submission and processing:

1. Search for the profile by **email address**: Found in email key space
2. Search for the profile by **CRM ID**: Found in the CRM-specific key space (e.g., "FCC Enterprise 2 key space")
3. Both searches return the same profile, confirming the merge worked

This dual-key-space linkage ensures:
- Future CRM updates can find the profile using the CRM ID
- No duplicates will be created on subsequent syncs
- Profile history is preserved on a single contact record

---

## Email Uniqueness Constraint

### Single Email Per Profile in Email Key Space

A consequence of the form submission flow is that once an email address is used to create a profile in the email key space, that email becomes unique in that key space.

[Erik Andersson]: "The downside of this of course is that because you are creating the profile in the e-mail key space, this e-mail address has now essentially been rendered unique, meaning that the requirements to have the same e-mail address on multiple profiles in the CRM can be a bit tricky, so multiple, multiple people cannot submit the same e-mail address in this flow."

### Subsequent Submissions with Same Email

If the same email is submitted again via the form:

1. Apsis One recognizes the email already exists in the email key space
2. The form submission updates the existing profile (pre-filled form behavior)
3. A **matched record** is returned from the CRM instead of a new record
4. The merge process occurs again
5. CRM data is downloaded and overwrites any Apsis-side updates

[Michal Rosikiewicz, observed in demo]: "When we will submit this form again with the same e-mail where they will receive any events on this profile like this demo link of it or something like this."

[Erik Andersson]: "You would then not create a new profile like you would use the same one because you are utilizing the e-mail key space."

### Creating Separate Leads with Same Email (via CRM)

It is still possible for the CRM system to create multiple contacts with the same email address directly in the CRM system, because Apsis One does not merge on the email key space during normal CRM sync operations. The CRM system owns that decision.

---

## CRM System Variations

### Efficy Enterprise 12.1 Limitations

During the live demonstration, it was observed that **Efficy Enterprise 12.1 does not fully utilize the outbound mapped fields**, even though Apsis One correctly provides them in the request.

[Erik Andersson]: "The unfortunate thing about Enterprise 12.1 is that they are not actually fully utilizing our mapped fields, we realized yesterday... they didn't put the first name as the actual first name, they just come to the yeah."

However, the Apsis One system is working correctly from its perspective—it sends the mapped data in the request. The limitation is on the Efficy side.

[Erik Andersson]: "However, the important part as far as we are concerned is that you could see that we did provide the first name and the phone number based on the outbound mappings in the request data. If if they are not actually fully utilizing them that that's bad, I'm going to ping them about it, but the system is working as far as we are concerned, like we are providing the data in the requests."

### CRM Systems Supporting Lead Generation

The following CRM systems support the lead generation flow:

- Efficy Enterprise 12.0
- Efficy Enterprise 12.1
- Tribe
- E-deal
- Max (partial support)

[Erik Andersson]: "Tribe behaves the same way, but I think they have actually implemented the outbound mappings properly and F is enterprise and yeah Max Max so is not and E deal also supports the lead generation."

---

## Extensibility: Beyond Form Submissions

### Current Implementation: Forms Only

Currently, outbound mappings are only added to the event data for **form submissions**. This is a design choice, not a technical limitation.

[Erik Andersson]: "The only time we add the mapped fields is for form submissions. If you would like to change this in the future, that is trivial to do. We just have a little check like if it is a form submit then we add the outbound mappings. The only reason we have that is because it was only for form submits."

### Future Possibilities

The generic connector supports adding outbound mappings for other event types:

- Email submissions
- Task notes
- Marketing automation flow activities
- Any other event type

This is a configuration change, not a code change.

[Erik Andersson]: "The generic connector does support it for anything. We can add the outbound mappings data for e-mail submits or task notes or MA flow things or whatever."

### Other Activity Syncing

All other activities sync to the CRM following the same flow as forms:

1. Activity is created with "sync to CRM" enabled
2. Event reaches audience subscription worker
3. Batch production worker batches events
4. CRM system responds with matching or new record IDs
5. Merge process occurs

The only difference is that outbound mappings are not currently included for non-form activities.

---

## Data Flow Synchronicity

### Asynchronous Processing

The entire lead sync flow (except form publishing) is **asynchronous**:

1. Form submission triggers an event
2. Event is queued in Kafka
3. Batch production worker gathers multiple events
4. Outbound worker sends to CRM
5. Merge and consent updates are generated as follow-up events

This means there is a delay between form submission and the profile appearing in the CRM key space in Apsis One.

### Publishing as Exception

Form publishing is the **only synchronous operation**:

[Erik Andersson]: "This is synchronous. This is the only thing in the whole like lead creation that happens synchronously... If our call to the CRM fails, that is a blocked operation."

---

## Key Takeaways

1. **Lead gathering in Apsis One** follows a standardized outbound flow: form submission → audience subscription worker → batch production → outbound worker → CRM system

2. **Outbound mappings** are distinct from field mappings. They enrich form submission events with Apsis attribute values so the CRM knows which attributes correspond to which fields.

3. **CRM response** provides CRM IDs for new or matched contacts. These IDs are used to merge the profile across key spaces.

4. **Profile merging** links the Apsis profile in its original key space (e.g., email) to a CRM-specific key space, enabling all future CRM updates to find and update the same profile without creating duplicates.

5. **Post-merge CRM download** ensures CRM data is always the authoritative source in Apsis One, overwriting any data that may have been updated during the form submission or merge.

6. **Consent synchronization** is the only bidirectional sync. New leads receive consent values from Apsis One, and any future consent changes in Apsis One immediately sync to the CRM for legal compliance.

7. **Email uniqueness** in the email key space means the same email cannot generate multiple leads via form submission, but the CRM system can independently create separate contacts with the same email.

8. **Form publishing is synchronous and blocking**, but all other lead sync operations are asynchronous and non-blocking.

9. **CRM system variations** mean some systems (like Efficy Enterprise 12.1) may not fully utilize outbound mapped fields, even though Apsis One correctly provides them in the request.

10. **Extensibility is trivial**: Adding outbound mappings to other event types (emails, tasks, etc.) is a configuration change, not an architectural change.

---

## Unresolved Questions and Action Items

1. **Diagram location**: Erik offered to provide a custom architectural diagram including outbound mappings. Decision: Create a new diagram (don't modify existing PNG) and add it to the architectural diagrams folder in the MA integrations documentation area. [Assigned to: Erik, Michal]

2. **Efficy Enterprise 12.1 mapped fields**: Erik noted that Efficy Enterprise 12.1 is not fully utilizing the outbound mapped fields sent by Apsis One. **Action**: Ping Efficy to clarify implementation status. [Assigned to: Erik]

3. **Post-merge data handling**: There is a question about whether CRM data should always win post-merge, or if this behavior should be configurable. Current implementation: CRM data always wins. No decision was made on whether this should remain the default or become configurable.

4. **Practical exercises**: Erik strongly recommends that all team members create forms and execute the full sync flow themselves to build muscle memory and deeper understanding of the process. [Recommended for: Michal, Lukasz]