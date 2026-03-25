---
source_file: Erik - Leads.txt
domain: Apsis One Integrations
topics: [Lead Creation and Gathering, Outbound Flow, Form Submissions to CRM, Outbound Mappings, Profile Merging, Consent Management, CRM Synchronization]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Audience Subscription Worker, Kafka, Batch Production Worker, Outbound Worker, Generic Connector, Efficy Enterprise 12.1, Merge Process, Consent Endpoints, CRM Key Spaces]
session_type: knowledge-transfer
subdomains: [Outbound Flow, Lead creation, Generic Connector, Efficy Enterprise 12.1]
---

## Session Overview

This session covers the complete lead gathering and creation workflow in Apsis One, with a focus on how form submissions are synchronized to CRM systems. Erik Andersson explains the technical architecture from form submission through to profile merging and consent synchronization, then walks through a live demonstration using Efficy Enterprise 12.1 to illustrate how the system handles event enrichment, CRM ID assignment, and duplicate prevention. The discussion addresses the role of outbound mappings, the distinction between inbound and outbound data flows, and the critical steps that happen after CRM response.

---

## Lead Gathering Architecture and Outbound Flow

### Overview of Lead Definition and Collection Methods

A **lead** in the CRM system context is defined as something that does not yet exist but is collected through Apsis and then sent to the CRM system. The primary method for gathering leads is through forms, though the system also supports leads from email submissions. Forms are the designed and preferred method for lead collection.

[Erik Andersson]: "Everything with lead gathering in APSIS happens in the so-called outbound flow. The main way where the customers gather the leads is using forms."

### Event Processing Pipeline: The Four-Stage Flow

When a lead submission occurs (for example, a form submit), it triggers an event that travels through a well-defined pipeline:

1. **Form Submission Event Trigger**: Someone submits a form that has been configured to sync to the CRM system. This creates an event that originates from Audience.

2. **Audience Subscription Worker Stage**: The first step in the outbound flow is handled by the **audience subscription worker** (also called the audience subscription worker). This worker performs two critical functions:
   - Verifies that all required data is present for synchronization
   - Enriches the event with **outbound mappings** data if configured

   For example, if a form field collects a first name and outbound mappings are configured to send the first name, the worker will include that data in the enrichment.

3. **Kafka Buffering**: After processing by the audience subscription worker, the event passes through Kafka.

4. **Batch Production Worker Stage**: The batch production worker gathers events into batches (up to 200 kilobytes each) for the specific section. For example, if five people submit a form, their five form submission events are consolidated into one batch.

5. **Outbound Worker Stage**: The outbound worker sends the accumulated batch request to the CRM system via the generic connector endpoint.

---

## Outbound Mappings: Two-Stage Configuration

[Erik Andersson]: "Our form is a two-stage rocket. So first you set up the mapping here from the form fields to the attributes in Apsis 1. The outbound mappings they because they work on the attribute level in Apsis one."

Outbound mappings represent a critical distinction in the data flow and require a two-stage setup:

### Stage 1: Form Fields → Apsis One Attributes

The first mapping layer connects form field data to attributes in Apsis One. This is the **inbound mapping** and happens when form data is received.

### Stage 2: Apsis One Attributes → CRM System Attributes

The second mapping layer takes values from Apsis One attributes and maps them to attributes in the CRM system. This is the **outbound mapping** and only includes data during event submission to the CRM.

### Why Outbound Mappings Matter

[Erik Andersson]: "If you didn't name your form field like e-mail, first name, last name, it's like motorbike, refrigerator and banana, you would not really know like what data should I put in the first name, what data should I put in the last name. The mapped field helps you with this."

The outbound mappings solve a critical problem: form field names may not match CRM attribute names. By mapping form data through Apsis One attribute names to CRM attribute names, the system ensures data goes to the correct place in the CRM, regardless of naming conventions.

**Important**: Not all CRM systems have fully implemented outbound mapping support in the generic connector, though the functionality exists in the system.

---

## CRM Response and Record Matching

### Request Payload Structure

When the outbound worker sends a batch to the CRM system's generic connector endpoint, the payload includes:

- **Profile Fields**: The raw form submission data (e.g., form field names and values)
- **Mapped Fields**: The enriched data from outbound mappings (e.g., what the CRM should call the "first name" field)
- **Event Data**: The structure and values submitted in the form

Example structure:
```
Profile key: <apsis_profile_key>
Event data: { field names and values as submitted }
Mapped fields: { mapped field name → value, e.g., "first_name" → "John" }
```

### CRM Matching and Response Logic

The CRM system responds with either new records created or matched records found. The response includes:

- **CRM IDs** (or lead IDs) for records that were either newly created or already existing
- **Matched Records**: Records the CRM identified as existing contacts/leads based on the data provided

The CRM uses the provided identifiers (CRM ID from previous interactions, email address, phone number, or other fields) to determine whether a contact already exists or is new. The CRM's matching logic is not regulated by Apsis; each CRM system decides its own matching criteria.

[Erik Andersson]: "They will respond with this data. The CRM system will then using this data, they will decide if it either matches or if it is something new, or they can also like completely ignore it and not do anything with that profile."

If the CRM cannot match the submission (for example, no CRM ID, no email, no phone number provided), it may return no data in the new or matched records section, effectively ignoring the submission.

---

## Profile Merging and De-Duplication Strategy

### The Merge Problem

When a form is submitted, a profile is created in the form's key space (e.g., email key space if email is the form identifier). However, the CRM system has now assigned a CRM ID to this contact. To prevent duplicates in future syncs, Apsis must establish a link between the form key space profile and the CRM key space using the CRM ID.

### The Merge Solution

After receiving the CRM response with CRM IDs, Apsis triggers a **merge operation**:

```
Merge specification:
- Export key space: Form submission key space (e.g., email)
- Export key space profile key: The profile key from the form submission
- Custom CRM key space: The integration-specific CRM key space (e.g., "FCC Enterprise 2")
- CRM ID: The ID assigned by the CRM system
```

[Erik Andersson]: "So now that we have the profile key from the export key space. The CRM system has provided us with the CRM ID or this contact in the CRM. So now we can do the merge specifying the export key space on the profile key and our custom CRM key space and the CRM ID."

### Merge Attribute Win Logic

In Apsis One, when merging profiles, the attribute with the newest (most recent) version wins by default. However, for CRM integrations, the data from the CRM is designated as the **master data**.

**Critical Implementation Detail**: To ensure CRM data always wins, immediately after the merge completes, Apsis downloads all current data from the CRM system for the contact and performs an update in Apsis One. This guarantees that regardless of the merge outcome, the CRM system's data is the authoritative version in Apsis.

[Erik Andersson]: "But at least for now, we want the data from the CRM system to be the master data. That means that as soon as the merge is finished, we download all of the data from the CRM system for the contacts and make a update inside of Apsis one just to make sure that regardless of the outcome of the of the merge, it is the data from the CRM system that is present on the contacts."

### De-duplication Benefit

Once the merge is complete and the profile is findable in the CRM key space using the CRM ID, any future updates from that CRM system will locate the same profile. This prevents the creation of duplicate profiles in Apsis One when the CRM sends updates back.

---

## Consent Synchronization for New Leads

### The Consent Problem for New Leads

When a new lead is created in the CRM system via form submission, the CRM has only the data sent in the form. If no consent information is transferred, the new lead has no consent to be contacted via email, SMS, or phone, making the lead essentially unusable for marketing communications.

[Erik Andersson]: "You have a new lead that registers like they submit a form saying I am interested in receiving technical newsletters. So this lead will be added to the CRM system, but if we don't send over any consent for this new lead that was gathered gathered from the forum or elsewhere, then you would not be able to like do anything with this lead because you wouldn't be able to be contacted use e-mail, you could not call them or send them a text because you don't have any consent to sending text messages."

### The Consent Solution: Initial Consent Export

When the CRM responds indicating a new record was created, Apsis immediately:

1. Detects that a new record was created (by checking the `new_records` response field)
2. Exports all consents from Apsis One for which outbound mappings exist
3. Sends these consents to the CRM system using the newly acquired CRM ID

This ensures the new lead inherits the consent settings captured during form submission, making them immediately actionable for marketing campaigns.

### The Consent Update Endpoint

Unlike attributes (which cannot be updated via the generic connector), **consents are the only bidirectional synchronization** in the entire integration flow. This is due to legal requirements: when a user changes their consent in Apsis One (by clicking unsubscribe in an email or SMS), this change must be immediately sent to the CRM to prevent compliance violations.

[Erik Andersson]: "We never update the attributes in the generic connector like you will see here like we can fetch the entity, but we cannot update the entities. We can, however, update the consents on entities because here is a legal requirements because the end users can modify their consents inside of apps is when they click on unsubscribe in emails or in SMS."

The system uses a **PATCH consent endpoint** to update consent records. This is the only update mechanism available in the generic connector for existing entities.

---

## Live Demonstration: Form Submission and Sync Flow with Efficy Enterprise 12.1

### Setup and Prerequisites

[Erik Andersson]: "The first obvious requirement is that you need to have a CRM system installed that supports form syncing. Now FCC Enterprise 12.1 is one of these."

The demonstration used:
- Efficy Enterprise 12.1 (FEC) as the target CRM system
- An Apsis One sandbox environment
- A newly created form configured for CRM sync

### Creating and Configuring the Form

The form creation process requires several mandatory steps:

1. **Form Template Selection**: A contact form template was used (vs. newsletter signup template)

2. **Field to Attribute Mapping** (Stage 1):
   - Form fields must be mapped to Apsis One attributes
   - In the demonstration: email, first name, middle name, mobile phone were mapped
   - This is the inbound mapping and happens automatically when form data is received

3. **CRM Sync Enablement**:
   ```
   The "CRM sync" checkbox MUST be selected
   ```
   If this is not selected, form submissions will not be forwarded to the CRM system.

4. **Terms and Conditions** (if required by the form template)

5. **Outbound Mappings Configuration** (Stage 2):
   - Navigate to the integration configuration page
   - Configure which Apsis One attributes map to which CRM system attributes
   - In the demonstration: first name and phone number were mapped for outbound sync
   - **Important distinction**: Field mappings on the integration page show data flowing FROM CRM TO Apsis (inbound). Outbound mappings show data flowing FROM Apsis TO CRM.

[Erik Andersson]: "The field mappings is only for the data going in to apps. This has nothing with the data we send out. The outbound mappings is the contrary. This is the events, in this case the form submit events that we send from Apsis to FC Enterprise 12/1."

### Form Publication: The Synchronous Operation

When the form is published, the system makes a **synchronous call to the CRM system**:

[Erik Andersson]: "What is happening now when you click on the publish is that you are making a request to us, notifying us that someone created a form now and they want to sync that to the CRM system. We will then in turn call the CRM system and say please create a form like a new campaign that is of type form."

This is the ONLY synchronous operation in the entire lead creation flow. If the CRM system call fails, the form publication is blocked and the user is immediately notified.

### Form Submission Event in CloudWatch Logs

After a form submission, the event can be found in CloudWatch logs by searching the **odd sub worker** logs for the submitted email address within the past few minutes.

In the demonstration logs, the following information was visible:

```
Profile fields:
  - Email: (submitted email)
  
Mapped fields:
  - first_name: "John" (from outbound mapping)
  - phone_number: "+1234567890" (from outbound mapping)

Event data:
  - All form field values as submitted

CRM response:
  - new_records: [CRM contact created with ID "234"]
  - CRM system: "FCC Enterprise 2"
```

### Response Processing and Consent Event Generation

After the CRM responds with the new CRM ID, Apsis generates an additional **consent update event**:

[Erik Andersson]: "So here you see that now that we triggered a, we generated A consent message update and we essentially like in the outbound worker, we create a message and put it back on the outbound worker. Like we we generate a message for ourselves."

This consent event is routed to the appropriate topic based on the subscription mapping for that CRM system (e.g., "FEC Enterprise 12.1 topic").

### Profile Key Space Changes After Merge

After the merge completes, the profile becomes findable in two key spaces:

1. **Email key space**: From the original form submission
2. **CRM key space** (e.g., "FCC Enterprise 2"): Using the CRM ID assigned by the system

[Erik Andersson]: "So you find it in the e-mail key space, but now exactly search for that CRM ID. Now you find that in our FCC Enterprise 2 key space. That means that you can find of course from the e-mail that you created from the form submission, but you can also find it in our FCC Enterprise 2 key space using the CRM ID."

This dual findability prevents duplicates in future syncs.

### Data Mastery After CRM Synchronization

After the merge and CRM data download, Apsis One reflects the authoritative CRM data:

[Erik Andersson]: "And this is because at the end of the merge process we download the data from the CRM, but because we we see that like because you have set up a mapping for these fields and we download the middle name and the mobile number and we get an empty value from the CRM then we interpret this as like the user has cleared this out in the CRM, so we clear it out inside of Apsis as well."

In the demonstration, fields like "middle name" and "mobile phone" that were submitted in the form but not fully implemented in the CRM's outbound mapping handling were cleared, reflecting the CRM's state.

---

## Duplicate Prevention and Email Address Uniqueness

### The Email Uniqueness Constraint

A key side effect of the form submission and merge flow is that the email address submitted becomes **unique** in Apsis One. This means:

- **Cannot Create Duplicates from Form**: If the same email is submitted again via the same form, the profile is updated rather than duplicated.
- **CRM Can Have Duplicates**: The CRM system itself may have contacts with the same email address (created outside of Apsis), and these will not merge with the form-submitted profile.

[Erik Andersson]: "The downside of this of course is that because you are creating the profile in the e-mail key space, this e-mail address has now essentially been rendered unique, meaning that the requirements to have the same e-mail address on multiple profiles in the CRM can be a bit tricky, so multiple, multiple people cannot submit the same email address in this flow."

### Subsequent Submissions with the Same Email

If the same email is submitted again:

1. Apsis finds the existing profile in the email key space
2. Updates that profile with new form data
3. The CRM system receives a matched record (not a new record)
4. The merge process runs again
5. CRM data is re-downloaded, ensuring CRM remains the master

[Erik Andersson]: "And if they are, how would that work in the form to like I I submit, I submit a profile using the same e-mail address. You would then not create a new profile like you would use the same one because you are utilizing the e-mail key space."

---

## CRM System Differences in Implementation

### Efficy Enterprise 12.1 Limitations

During the demonstration, it was discovered that **Efficy Enterprise 12.1 does not fully implement the outbound mappings functionality**:

[Erik Andersson]: "The unfortunate thing about Enterprise 12.1 is that they are not actually fully utilizing our mapped fields, we realized yesterday."

In the demonstration:
- Apsis sent the mapped "first_name" and "phone_number" fields in the request
- The CRM received these fields but did not properly place them in the corresponding first name and phone number attributes
- The values appeared in the contact record but not in the expected fields

[Erik Andersson]: "However, the important part as far as we are concerned is that you could see that we did provide the first name and the phone number based on the outbound mappings in the request data. If if they are not actually fully utilizing them that that's bad, I'm going to ping them about it, but the system is working as far as we are concerned, like we are providing the data in the requests."

### Other CRM Systems Mentioned

The following systems were mentioned as supporting lead generation with varying levels of outbound mapping implementation:

- **Tribe**: Believed to properly implement outbound mappings
- **E-deal (Efficy Corporate)**: Supports lead generation
- **Max**: May not fully support outbound mappings
- **Efficy Enterprise 12.0**: Implied to have similar issues to 12.1

---

## Extensibility: Beyond Form Submissions

### Current Limitation: Form Submissions Only

Currently, the system only enriches events with outbound mappings data for **form submissions**. This is implemented as a conditional check in the code:

```
if event_type == "form_submit":
    add_outbound_mappings_data()
```

[Erik Andersson]: "The only reason we have that is because it was only for form submits. There was a request to gather the leads, but the generic connector does support it for anything."

### Potential Future Expansion

The system is architected to easily extend outbound mappings to other event types:

[Erik Andersson]: "We just have a check like if it is a form submit then we add the outbound mappings. The only reason we have that is because it was only for form submits. We can add the outbound mappings data for e-mail submits or task notes or MA flow things or whatever."

This would allow lead creation from:
- Email submissions
- Task notes
- Marketing automation flows
- Any other activity type

The only requirement is adding the outbound mappings data during event enrichment.

---

## All CRM Systems: Consistent Behavior Pattern

[Erik Andersson]: "This is how the lead gathering system is set up for now. Any activities that we sync from apps is work like the same way as we did here. You create the activity, you set the like sync to CRM. The event will reach all sub worker. It will go to batch production worker. The CRM system will say I have a matching contact or a matching person for this and they will give us the CRM IDs."

Regardless of which CRM system is used (Tribe, Efficy Enterprise, E-deal, etc.), the flow is consistent:

1. Create form or activity
2. Enable CRM sync
3. Submit event
4. Audience subscription worker enriches
5. Batch production worker batches
6. Outbound worker sends to CRM
7. CRM returns IDs
8. Merge process executes
9. Consent is synced for new records
10. Profile becomes findable in CRM key space

---

## Architecture Diagram and Documentation

During the session, there was a discussion about updating the architectural diagrams:

[Lukasz Grabowski]: "Are you going to put somewhere this this diagram?"

[Erik Andersson]: "I can upload it to uh. To anywhere we like."

The team agreed to add a new architectural diagram that explicitly includes **outbound mappings** in the flow, as the existing architectural diagrams in the integration documentation did not show this critical component. The new diagram was to be added alongside (not replacing) existing diagrams to avoid confusion.

---

## Key Takeaways

1. **Two-Stage Mapping is Essential**: Form fields → Apsis attributes → CRM attributes. Both stages must be configured for successful lead creation.

2. **Outbound Flow is Asynchronous**: The entire lead creation flow (from submission to CRM sync) is asynchronous via Kafka, with the exception of form publication, which blocks on CRM response.

3. **Merge Prevents Duplicates**: The merge process using CRM IDs ensures that future CRM updates find the same profile, preventing duplicates.

4. **CRM is Master Data**: After merge, all CRM data is re-downloaded to Apsis, making the CRM system the authoritative source, not the form submission data.

5. **Consent Must Be Synced for New Leads**: New leads without consent are unusable. Apsis automatically exports consents for newly created CRM records.

6. **Email Uniqueness Constraint**: Form submissions create profiles in email key space, making that email effectively unique within Apsis.

7. **Generic Connector Supports Extension**: Outbound mappings can be easily extended to other event types beyond form submissions.

8. **CRM Implementation Varies**: Different CRM systems may not fully implement outbound mapping functionality (e.g., Efficy Enterprise 12.1), but the system still sends the mapped data correctly.

9. **Bidirectional Sync is Consent-Only**: Consents are the only data type that syncs bidirectionally (Apsis → CRM and CRM → Apsis) due to legal compliance requirements.

10. **Profile Findability Enables Future Syncs**: Once merged into the CRM key space, profiles are findable by CRM ID in all subsequent syncs and updates.

---

## Unresolved Questions and Action Items

1. **Efficy Enterprise 12.1 Outbound Mappings Implementation**: Erik noted he would contact Efficy to address why the system is not properly utilizing the mapped fields sent in the request payload.

2. **Architectural Diagram Update**: A new diagram showing outbound mappings in the lead creation flow should be added to the integration documentation folder (location: "MA and integration" → "architectural diagrams").

3. **Future CRM Behavior Clarification**: Whether CRM data should always win in the merge, or if form submission data should be preserved in certain cases, is marked for potential future review and change.

4. **Extended Outbound Mapping Use Cases**: No decisions have been made on whether to extend outbound mappings to email submissions, task notes, or other activities in the near term.