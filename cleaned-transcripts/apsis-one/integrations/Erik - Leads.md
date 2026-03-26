---
source_file: Erik - Leads.txt
domain: Apsis One Integrations
topics: [Lead creation workflow, Form submission sync, Outbound mappings, Profile merging, Consent handling, CRM system integration, Generic connector]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Audience subscription worker, Kafka, Batch production worker, Outbound worker, Generic connector, Field mappings, Outbound mappings, Merge process, Consent endpoints, FCC Enterprise 12.1, Tribe, Max, E.deal]
session_type: knowledge-transfer
subdomains: ["Lead creation"]
---

## Session Overview

This knowledge transfer session covers the complete lead creation workflow in Apsis One Integrations, focusing on how forms collect new contacts and sync them to CRM systems. Erik Andersson walks through the architectural flow from form submission through batch processing, CRM synchronization, profile merging, and consent handling. The session includes a live demonstration using FCC Enterprise 12.1 to illustrate the practical implementation of these concepts.

---

## Lead Creation Fundamentals

### Definition and High-Level Flow

A **lead** in the CRM system context is a new contact that does not yet exist but is collected through Apsis One and sent to the CRM system. The primary method for gathering leads is through **forms**, though email submissions also trigger this workflow.

The basic flow is:
1. Customer submits a form configured for CRM sync
2. Event is generated and processed through the outbound pipeline
3. CRM system creates or matches a contact record
4. Profile is merged with the CRM ID as the identifying key
5. Consent data is synchronized back to the CRM system

### Why Form Sync Requires CRM Configuration

[Erik Andersson]: For the lead creation system to work, the customer must have a CRM system installed that explicitly supports form syncing. Not all CRM systems support this feature out of the box.

---

## The Outbound Flow Architecture

### Event Processing Pipeline

The lead creation flow follows the "outbound flow" pipeline in Apsis One. The journey of a form submission is:

1. **Event Origin**: Form submit event originates from the audience system
2. **Audience Subscription Worker**: First checkpoint in the outbound flow
   - Verifies all required data is present for sync
   - Enriches the event with **outbound mappings** if configured
   - Example: if a profile has a first name from a form field and outbound mappings are configured for first name, it includes that data in enrichment
3. **Kafka**: Brief message queue stop
4. **Batch Production Worker**: Aggregates events
   - Gathers up to 200 kilobytes of data for a specific section
   - Example: if five people submit a form, their five submissions are batched into one request
5. **Outbound Worker**: Sends the batch request to the CRM system

### Architecture Diagram Note

[Erik Andersson]: The current architectural diagrams in the knowledge base don't include the outbound mappings component, which should be added for completeness.

---

## Outbound Mappings: Two-Stage Rocket Architecture

Outbound mappings represent a critical two-stage system for enriching data sent to CRM systems:

### Stage 1: Form Fields to Apsis One Attributes
The customer sets up mappings from form field names to attributes stored in Apsis One. This happens during form configuration.

### Stage 2: Apsis One Attributes to CRM System Attributes
Once data is stored as attributes in Apsis One, outbound mappings convert those attributes to the CRM system's expected field names when syncing occurs.

[Erik Andersson]: Both stages must be in place for the system to fully work. For example:
- Form field might be named "motorbike" or "refrigerator"
- But outbound mappings tell the system: "when syncing, send this attribute value as the 'first_name' field to the CRM"
- This solves the problem where form field names don't match CRM expectations

### Implementation in Generic Connector

The generic connector endpoint receives batches with this structure:

```
{
  "profile_key": "...",
  "events": [
    {
      "event_data": {
        "form_field_1": "value",
        "form_field_2": "value"
      },
      "mapped_fields": {
        "first_name": "...",
        "last_name": "...",
        "email": "..."
      }
    }
  ]
}
```

The "mapped_fields" section only appears for data that has been explicitly configured in outbound mappings.

### Current Limitation: Form Submissions Only

[Erik Andersson]: Currently, outbound mappings are only added for form submit events. The generic connector supports adding mapped fields for any event type (email submissions, task notes, MA flow events, etc.), but this is not yet enabled. Adding support for additional event types would be trivial to implement—it only requires changing a conditional check.

---

## CRM System Response and Record Creation

### Response Structure

When the CRM system receives a sync request, it responds with information about records created or matched:

```
{
  "new_records": [
    {
      "profile_key": "...",
      "crm_id": "234"
    }
  ],
  "matched_records": [
    {
      "profile_key": "...",
      "crm_id": "567"
    }
  ]
}
```

### Matching Logic

[Erik Andersson]: The CRM system decides whether a record is new or existing using data we provide:
- CRM ID (if previously known)
- Email address
- SMS number
- Or other identifying fields

The CRM system uses this data according to its own matching logic. Some CRM systems can completely ignore data if they cannot match it or don't have sufficient information. For example, if we send a contact with no CRM ID, no email, and no SMS, the CRM system would have nothing to work with.

Each CRM system has its own naming conventions for identifying fields:
- FCC Enterprise 12.1 uses: `K_Contact` (contact field name) and `Email_1` (email field name)
- Tribe uses different naming conventions
- Ideal CRM uses different conventions

[Erik Andersson]: We dynamically utilize the field names from the CRM system's own data, so they can easily identify matches.

---

## Profile Merging and CRM ID Integration

### The Merge Requirement

After the CRM system creates a new record, we need to ensure the Apsis One profile can be found using the CRM ID in future syncs. This requires a **merge operation**:

- Take the existing profile key from the form's key space (email key space)
- Merge it with the custom CRM key space using the CRM ID provided by the system
- Result: the profile is now identifiable by both email and CRM ID

### Merge Winner Logic and Data Mastery

[Erik Andersson]: In Apsis One, when you merge profiles, the attribute with the **newest/latest version** wins. This creates a potential problem in the lead creation flow.

**Example scenario**: User submits a form with a prefilled name that differs from what's in the CRM system. The form submission updates the name attribute, making it the newest version. During merge, the form submission's name would win, not the CRM system's data.

**Solution implemented**: After the merge completes, we immediately download all data from the CRM system for that contact and perform an update inside Apsis One. This ensures **CRM data is always the master data**, regardless of what the form submission contained.

[Erik Andersson]: This behavior may change in the future if desired, but currently CRM data always takes precedence.

---

## Consent Handling and Legal Compliance

### The Consent Challenge for New Leads

When a new lead is created from a form submission, they must have consent set in the CRM system, otherwise:
- Cannot contact via email (no email consent)
- Cannot call (no phone consent)
- Cannot send SMS (no SMS consent)

[Erik Andersson]: Without consent, the CRM system cannot use the lead at all. This is why we immediately export any consents from Apsis One when we detect a new record.

### Bidirectional Consent Synchronization

Consent is the **only bidirectional sync** in the lead creation flow:

1. **Outbound** (Apsis → CRM): When a new record is created, we export all consents that have mappings configured and send them to the CRM system
2. **Inbound** (CRM → Apsis): When users modify consents in Apsis One (by clicking unsubscribe in emails/SMS), we must send this change to the CRM system immediately

[Erik Andersson]: This bidirectional requirement exists because of legal compliance. If a user opts out in Apsis One but we don't tell the CRM system, the next sync from the CRM system would overwrite the opt-out with the opt-in, violating the user's choice.

### Consent Update Endpoint

We use the **PATCH consent endpoint** to update consents on existing entities because:
- We never update general attributes in the generic connector (only fetch)
- Consents have a legal requirement to be updateable by end users
- Once we have the CRM ID from the initial sync, we can update consents for that specific entity

---

## Form Configuration for CRM Sync

### Setting Up Form Field Mappings

When creating a form that will sync to CRM:

1. **Map form fields to Apsis One attributes** (stage 1 of the two-stage system)
   - Form field "Name" → Apsis One attribute "first_name"
   - Form field "Phone" → Apsis One attribute "phone_number"

2. **Configure outbound mappings** (stage 2)
   - Go to Integration page
   - Set up outbound mappings from Apsis One attributes to CRM system field names
   - Example: Apsis attribute "first_name" → CRM field "FirstName"

### Critical Form Configuration: CRM Sync Toggle

[Erik Andersson]: The **CRM Sync** checkbox is mandatory for lead generation. If the customer wants to gather leads from a form, this must be selected. Otherwise, Apsis One will not listen to form submissions from this form.

### Publishing is Synchronous

When you click "Publish" on a form, the system performs a **synchronous** operation:

1. Request is sent to the CRM system asking it to create a new campaign/form of type "form"
2. System waits for the CRM system's response
3. If the CRM system call fails, the form publication fails—it's a **blocked operation**
4. User receives immediate feedback about success or failure

This is the **only synchronous call** in the entire lead creation flow. All other processing (batching, merging, consent syncing) happens asynchronously.

---

## Live Demonstration: Form Submission Workflow

### Form Submission Process

[Michal Rosikiewicz demonstrates with FCC Enterprise 12.1]

1. User submits form with email and other fields
2. Form submit event is immediately generated

### Observing Events in Logs

[Erik Andersson]: To verify the workflow, check the **Audience Subscription Worker** (also called "odd sub worker") logs in CloudWatch:
- Search by email address
- Look for submit events within the last few minutes
- Verify the event contains both raw field data and mapped fields

Example log output shows:
```
Profile fields:
- email: user@example.com

Mapped fields (from outbound mappings):
- first_name: John
- last_name: Doe
- phone: +1234567890
```

### CRM System Response Analysis

In the Outbound Worker logs, the CRM system's response reveals:

```
{
  "new_records": [
    {
      "profile_key": "abc123",
      "crm_id": "234"
    }
  ]
}
```

[Erik Andersson]: This tells us:
- FCC Enterprise created a **new contact** (not a lead, they use contact terminology)
- The CRM system assigned it ID 234
- This ID will now be used for all future syncs related to this profile

### Consent Message Generation

[Erik Andersson]: After receiving the CRM ID, the system automatically generates a **consent update event** and puts it back on the outbound worker queue. This is an internal message that ensures consent data is pushed to the CRM system for the newly created contact.

If consents were configured on the form, you'll see:
```
Consent update event generated for profile_qa subscription topic
```

The "profile_qa" is the topic/subscription type from the form configuration, which maps to the FCC Enterprise 12.1 subscription in the integration settings.

---

## Profile Key Space and Duplicate Prevention

### The Key Space Solution

After form submission sync completes, the profile exists in **two key spaces**:

1. **Email Key Space**: Original identifier from form submission
   - Search: `email:user@example.com` → finds profile

2. **FCC Enterprise Key Space**: New identifier from CRM sync
   - Search: `crm:234` → finds the same profile

[Erik Andersson]: This dual identification prevents duplicates in future syncs because:
- Any subsequent updates from FCC Enterprise will use the FCC Enterprise key space
- The system finds the same profile that was created from the form submission
- No duplicate profile is created

### Why Email Becomes Unique

[Erik Andersson]: Once a profile is created in the email key space from a form submission, that email address becomes **unique** within that key space. This creates a limitation:

- **Cannot create multiple leads with the same email** through the form submission flow
- Form resubmission with the same email will use the existing profile (not create a new one)
- If the form allows updates, the existing profile is updated

However, the CRM system can create completely new contacts with the same email address independently, because the CRM system doesn't merge on email—it uses its own matching logic.

---

## CRM System Comparison and Outbound Mappings Support

### Generic Connector CRM Systems Tested

**FCC Enterprise 12.1**: Does not fully utilize outbound mapped fields
- [Erik Andersson]: "Enterprise 12.1 doesn't actually fully utilize our mapped fields. We realized this yesterday."
- First name and phone number are provided in the request but not properly stored in the CRM system
- System is still working correctly—Apsis One is providing the data; the CRM system isn't using it properly
- [Erik Andersson]: "I'm going to ping them about it, but the system is working as far as we are concerned, like we are providing the data in the requests."

**Tribe**: Properly implements outbound mappings
- [Erik Andersson]: "Tribe has actually implemented the outbound mappings properly"

**Max**: Supports lead generation

**E.deal**: Supports lead generation

### Field Mapping Verification in CRM

After form submission, you can verify the CRM system's integration settings:
- Check the Integration page for field mappings (inbound) and outbound mappings
- Ensure the correct CRM system is selected
- Verify the topic/subscription type matches what's configured on the form

[Erik Andersson]: Field mappings are for data coming IN from the CRM to Apsis One (inbound). Outbound mappings are for data going OUT from Apsis One to the CRM during form submissions. These are often confused but are separate systems.

---

## Data Preservation After Merge and Update

### CRM Data Download Behavior

After the merge process completes and CRM data is downloaded back into Apsis One:

[Michal Rosikiewicz observes]: "The profile shows only last name now. First name and middle name are empty even though we submitted them."

[Erik Andersson explains]: This happens because:
1. Field mappings were configured for these fields (first name, middle name, phone)
2. During the CRM data download, these fields came back empty from FCC Enterprise
3. Empty values are interpreted as "user cleared this in the CRM"
4. Apsis One clears the field locally to match the CRM's data

This demonstrates the **CRM data mastery** principle in action.

### Form Submission Event Records

[Erik Andersson]: Even though the CRM data overwrites the form data during merge, the form submission events themselves are preserved on the profile. This means:
- You have the complete audit trail from initial form submission through final CRM state
- All events remain on the same contact profile
- Historical record is maintained

---

## Duplicate Prevention Through CRM ID Matching

### How Future Syncs Avoid Duplicates

Once the CRM ID is established through the merge process:

1. **Any future updates from the CRM system** use the FCC Enterprise key space to find the profile
2. **System matches on CRM ID**, not email
3. **Same profile is found and updated**, not a new profile created

[Erik Andersson]: This is critical because it ensures:
- No duplicates from consent updates
- No duplicates from contact attribute updates
- No duplicates from full sync operations

All future syncs use the CRM ID as the primary identifier.

---

## Testing and Practice Recommendations

### Hands-On Learning

[Erik Andersson]: "I highly recommend you just like create a form and do this yourself. I mean you've done this a couple of times now already, but just to get a feel for the flow."

The practical demonstration should include:
1. Creating a form with field mappings
2. Configuring outbound mappings on the integration
3. Publishing the form (synchronous operation)
4. Submitting the form with test data
5. Observing the asynchronous workflow in CloudWatch logs
6. Verifying the profile appears in both key spaces
7. Checking the CRM system for the new contact record

### Extending to Other Activity Types

[Erik Andersson]: The lead generation workflow is not limited to forms:
- **Email submissions**: Not currently including mapped fields, but could
- **Task notes**: Could include mapped fields
- **MA flow events**: Could include mapped fields
- **Any other activity type**: Could be enhanced

The implementation pattern is consistent across all activity types.

---

## Key Takeaways

1. **Lead creation** is the process of collecting new contacts via forms (or other activities) and syncing them to a CRM system as new records.

2. **Outbound mappings** use a two-stage system: form fields → Apsis attributes → CRM fields. Both stages must be configured for complete functionality.

3. **The outbound flow** follows a consistent pipeline: Audience Subscription Worker → Kafka → Batch Production Worker → Outbound Worker. Form publication is the only synchronous step.

4. **CRM IDs are critical** because they enable duplicate prevention by creating a second key space for profile identification after merge.

5. **CRM data is master data**—after merge, we immediately download CRM data to ensure Apsis One reflects the CRM system's values, not the form submission values.

6. **Consent is bidirectional** because users can modify consents in Apsis One (via unsubscribe), and these changes must be pushed to the CRM system for legal compliance.

7. **Email becomes unique** in the email key space after form submission, preventing duplicate profile creation from repeated submissions with the same email.

8. **CRM system support varies**—not all CRM systems properly implement outbound mappings, but Apsis One correctly provides the data regardless.

9. **Merge winner logic** (newest attribute wins) is overridden by immediately downloading CRM data, ensuring consistent data mastery.

10. **Form configuration requires three critical elements**: field mappings (to Apsis attributes), outbound mappings (to CRM fields), and the CRM Sync toggle enabled.

---

## Unresolved Questions and Notes

- **FCC Enterprise 12.1 outbound mapping issue**: Erik noted he would "ping them" about why Enterprise 12.1 isn't fully utilizing the mapped fields provided in the request, despite receiving them correctly. This appears to be a CRM system implementation issue, not an Apsis One issue.

- **Future data mastery preference**: Erik indicated uncertainty about whether CRM data should always win in the merge: "If this should still work like this going forward, I don't know, but it would be quite easy to change if you would so want to."

- **Architectural diagram updates**: Need to add outbound mappings component to existing architectural diagrams in the knowledge base.