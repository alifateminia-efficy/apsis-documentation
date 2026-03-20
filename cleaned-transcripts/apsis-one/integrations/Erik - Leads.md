---
source_file: Erik - Leads.txt
domain: Apsis One - Integrations
topics: [Lead Generation, Form Submissions, CRM Synchronization, Outbound Mappings, Profile Merging, Consent Management, Generic Connector, Batch Processing]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz]
key_components: [Audience Subscription Worker, Batch Production Worker, Outbound Worker, Generic Connector, CRM Systems (FCC Enterprise 12.1, Tribe, Max, E-Deal), Kafka, Form Submit Events, Profile Keys, CRM ID, Merge Process, Consent Updates]
session_type: knowledge-transfer
---

## Session Overview

This session covers the complete flow of lead generation and synchronization in Apsis One, with a focus on how form submissions are captured, enriched, and synchronized to external CRM systems. The discussion traces the journey from initial form submission through the audience subscription worker, batch processing, outbound API calls, profile merging, and consent synchronization. A live demonstration using FCC Enterprise 12.1 was conducted to illustrate the practical implementation of these concepts, showing how outbound mappings enrich form submission data before being sent to the CRM system.

---

## Lead Generation Architecture and Data Flow

### Overview of the Outbound Flow

[Erik Andersson]: All lead gathering in Apsis happens in the **outbound flow**. The main mechanism customers use to gather leads is through **forms**, though leads can also be collected through email submissions. A lead in the CRM system is data that does not yet exist in Apsis but is collected through our platform and then sent to the external CRM system.

### Event Lifecycle: Form Submission to CRM Sync

The complete flow for lead generation follows these steps:

1. **Form Submission Event**: A user submits a form that is configured to be synced to the CRM system.

2. **Audience Subscription Worker**: This is the first step in the outbound flow. The audience subscription worker:
   - Verifies that all required data is present for synchronization
   - Enriches the event with **outbound mappings** data if configured
   - For example, if a form field captures a first name and outbound mappings are configured to send the first name, the worker includes this data in the enriched event

3. **Kafka Buffering**: The enriched event briefly passes through Kafka before moving to the next stage.

4. **Batch Production Worker**: 
   - Gathers multiple events (up to 200 kilobytes) for a specific section
   - Example: five form submissions are batched into a single larger batch
   - Passes the batch to the outbound worker

5. **Outbound Worker**: Sends the batch request to the CRM system via the **generic connector**

### Important Distinction: Field Mappings vs. Outbound Mappings

[Erik Andersson]: There is critical confusion to avoid here:
- **Field Mappings**: Configuration that defines how data flows FROM the CRM system TO Apsis (inbound, for attribute changes)
- **Outbound Mappings**: Configuration that enriches form submit events FROM Apsis TO the CRM system (outbound, for event enrichment)

These are two separate stages of the integration and require both to be configured for lead generation to work properly.

---

## Outbound Mappings and Event Enrichment

### How Outbound Mappings Work

[Erik Andersson]: Outbound mappings are a **two-stage process**:

1. **Stage 1**: Map form fields to attributes in Apsis One
2. **Stage 2**: Map Apsis One attributes to CRM system attributes via outbound mappings

The outbound mappings work at the attribute level in Apsis One. The system takes the value from Apsis attributes and maps them to the corresponding attributes in the CRM system.

### The Problem Outbound Mappings Solve

Without outbound mappings, if a form field is named "motorbike", "refrigerator", or "banana", the CRM system has no way to know what data should map to standard fields like first name, last name, or email. The outbound mapping provides this semantic translation layer.

### Data Structure in the API Request

When sending a batch to the CRM system, the outbound worker includes:
- **Event data**: The raw form field values (field names as configured in the form)
- **Mapped fields**: The semantic mapping showing which form data corresponds to which CRM attributes (e.g., "this is the first name", "this is the last name", "this is the email")

Example structure:
```
Profile event data:
  - field_name: "contact_info_1"
    value: "John"
  - field_name: "contact_info_2"
    value: "Doe"

Mapped fields (via outbound mappings):
  - first_name: "John"
  - last_name: "Doe"
  - email: "john.doe@example.com"
```

### CRM System Implementation Variance

[Erik Andersson]: Not all CRM systems fully implement outbound mappings support. The generic connector provides this functionality, but individual CRM implementations may not utilize the mapped fields properly. The responsibility for actually using the mapped fields lies with the CRM system integration.

---

## CRM Response and Profile Identity Resolution

### CRM System Response Format

When the CRM system receives the batch, it responds with:
- **New Records**: Contacts/leads that were newly created in the CRM system
- **Matched Records**: Contacts/leads that already existed and were matched
- **CRM IDs**: Unique identifiers for created or matched records

The CRM system determines existence matching based on:
- Provided CRM ID (if one was previously sent)
- Email address
- SMS identifier
- Other identifying information provided

The CRM system can also choose to ignore a profile entirely if it lacks sufficient identifying information (e.g., no CRM ID, no email, no SMS), in which case no new or matched records will be returned.

### Identity Resolution Challenge

[Erik Andersson]: A critical issue arises from the form submission flow:
- The profile initially exists in the **form key space** (where it was created)
- The profile also exists in the **email key space** (assuming email was captured)
- But the profile is not yet identifiable using the **CRM key space** (which uses the CRM ID)

This creates a potential for duplicates in future synchronizations if not resolved.

---

## The Merge Process and Data Master Strategy

### Triggering the Merge

After receiving the CRM response with the CRM ID, Apsis triggers a **merge operation** that:
- Specifies the original key space (e.g., form key space or email key space)
- The profile key from that space
- The custom CRM key space
- The CRM ID returned by the CRM system

This merge ensures the same profile can be found using the CRM ID in all future synchronizations.

### Merge Winner and Attribute Precedence

In Apsis, when a merge occurs, the attribute with the **newest/latest version** wins. For example:
- If a form submission updates a name field (via pre-filled forms)
- That updated name becomes the latest version
- The latest version wins in the merge

### CRM System as Master Data

[Erik Andersson]: Currently, Apsis treats the **CRM system as the master data source**. Here's the strategy:

1. Merge occurs between the form key space and CRM key space
2. **Immediately after merge completion**, Apsis downloads all data from the CRM system for that contact
3. Performs an update within Apsis to ensure CRM data takes precedence

This means:
- Even if the merge algorithm would favor Apsis data, the subsequent CRM download overwrites it
- The CRM system's version of the truth is always the final state in Apsis
- Any data cleared in the CRM system (empty values) is interpreted as deleted and cleared in Apsis as well

[Lukasz Grabowski]: Question about future strategy: Should the CRM system remain the master, or could this change? 

[Erik Andersson]: This could be modified if needed, but currently the CRM is the master by design. Changing this would be straightforward from a technical perspective.

---

## Consent Synchronization and Bidirectional Sync

### The Consent Export Requirement

[Erik Andersson]: When a new contact/lead is created in the CRM system from a form submission, Apsis must immediately export any associated consents. This is critical:

**Use Case**: A new lead registers via form indicating interest in "technical newsletters". Without consent synchronization:
- The lead is added to the CRM system
- But no consent is sent
- The CRM system cannot contact the lead via email, phone, or SMS (no legal basis)

**Solution**: As soon as Apsis detects a new record response from the CRM, it performs an export of all configured consents and sends these to the CRM system via API.

### Bidirectional Consent Sync - The Only Bidirectional Flow

[Erik Andersson]: Consent is the **only aspect of the lead sync flow that is bidirectional**:

**Reason for Bidirectionality**: When users modify their consents in Apsis (by clicking unsubscribe in emails or SMS), this change must immediately be reflected in the CRM system. Otherwise:
- User opts out in Apsis (clear consent)
- Next CRM sync overwrites opt-out with opt-in from CRM
- Legal violation: contacting user against their wishes

**Technical Implementation**:
- Uses a dedicated **PATCH consent endpoint** (not a general entity update endpoint)
- The generic connector only supports reading entities (fetch), not updating them
- Consents are special because they have legal/regulatory requirements

### Consent Update Process for New Leads

When a new contact is created in the CRM:
1. CRM responds with new CRM ID
2. Apsis generates a **consent update message** for itself
3. This message is placed back on the outbound worker queue
4. The consent is associated with the profile using the newly acquired CRM ID
5. The PATCH consent endpoint is called to update the CRM system

---

## Practical Configuration and Implementation

### Prerequisites for Form-to-CRM Sync

[Erik Andersson]: To enable form synchronization to a CRM system, the following must be true:
1. A CRM system with form sync support must be installed (e.g., FCC Enterprise 12.1, Tribe, Max, E-Deal)
2. The integration must be properly configured in Apsis
3. Both field mappings (for incoming data) AND outbound mappings (for outgoing event enrichment) must be configured
4. The **"CRM sync" checkbox** must be enabled on the form itself

[Erik Andersson]: The "CRM sync" checkbox is mandatory. Without it, Apsis will not listen to form submission events for that form.

### Form Configuration Steps

Based on the live demonstration:

1. **Create Form**: Use a template or build from scratch (e.g., "Contact Form Template")
2. **Map Form Fields to Apsis Attributes**: Each form field must map to an attribute in Apsis One
   - Example: Form field "first_name" → Apsis attribute "first_name"
3. **Configure Outbound Mappings**: Define how Apsis attributes should map to CRM attributes
   - Example: Apsis "first_name" attribute → CRM "K_FIRST_NAME" field
4. **Add Terms and Conditions**: Required field for form validation
5. **Enable CRM Sync**: Check the "CRM sync" option on the form
6. **Select Topic/Subscription Mapping**: Choose the CRM system's corresponding topic
   - Example: "FCC Enterprise 12.1 topic"
7. **Publish**: When published, Apsis makes a **synchronous call** to the CRM system to create the corresponding form/campaign

### The Publish Operation: The Only Synchronous Step

[Erik Andersson]: The entire lead generation flow is asynchronous **except** for the publish operation:
- When form is published, Apsis immediately calls the CRM system
- Requests creation of a form/campaign on the CRM side (type: form)
- This is a **blocked operation**: If the CRM call fails, the form cannot be published
- User receives immediate feedback on success or failure

All subsequent steps (form submission, batching, sync) are asynchronous.

---

## Live Demonstration: FCC Enterprise 12.1 Integration

### Setup and Navigation

The demonstration used:
- **CRM System**: FCC Enterprise 12.1
- **Sandbox Environment**: Pre-configured sandbox for testing
- **Console Access**: Required for viewing logs and verifying profile state

### Configuration Verification in the Integration

Key configuration elements visible in the integration page:
- **Field Mappings**: Shows data flowing FROM FCC Enterprise TO Apsis (inbound)
- **Outbound Mappings**: Shows data flowing FROM Apsis TO FCC Enterprise (outbound enrichment)
- Forms must be created on the correct section that has the integration configured

### Form Submission and Event Tracking

After creating and publishing a test form with email submission:

1. **Form Submit Event Details** (visible in logs):
   - `profile_fields`: Contains identifying information (email address)
   - `event_data`: Raw form submission data
   - `mapped_fields`: Enriched data from outbound mappings (e.g., "first_name": "John", "phone": "+1234567890")

2. **CRM System Response** (visible in outbound worker logs):
   - `new_records`: Array showing newly created contacts
   - Contains the CRM ID generated by the system
   - Example response: "We created a new contact with CRM ID 234"

### Identity Resolution Verification

[Michal Rosikiewicz / Erik Andersson]: After form submission and sync completion:
- Search by **email** in Apsis: Profile found in email key space
- Search by **CRM ID**: Same profile found in FCC Enterprise 2 key space
- **Result**: The profile is now identifiable via both mechanisms, eliminating duplicate risk in future syncs

### Merge and Data Master Behavior Observation

After the merge process and CRM data download:
- Form field data (e.g., "middle_name") that was submitted may be cleared
- This occurs if the CRM system returned empty values for those fields
- **Interpretation**: CRM system didn't capture/store these fields; Apsis clears them to match
- If outbound mappings had properly configured these fields, they would have persisted

[Erik Andersson]: This demonstrates the design principle: **CRM data wins after merge**. If form fields are important, they must be:
1. Mapped from form to Apsis attributes
2. Configured in outbound mappings to send to CRM
3. Actually stored by the CRM system

---

## CRM System Support and Implementation Variance

### Supported CRM Systems

The generic connector supports form submission sync for:
- **FCC Enterprise 12.1**: Core support, but incomplete implementation of mapped fields
- **Tribe**: Proper implementation of outbound mappings
- **Max**: Supports lead generation
- **E-Deal**: Supports lead generation

[Erik Andersson]: Each system may implement outbound mappings differently. Some fully utilize the semantic mapping data; others may ignore it.

### Known Implementation Gaps

**FCC Enterprise 12.1**: Does not fully utilize mapped fields in the current version
- Apsis correctly sends outbound mapping data in requests
- FCC Enterprise doesn't properly use that enriched data
- The integration is working correctly from Apsis's perspective; the issue is in FCC's implementation
- Erik planned to follow up with FCC about proper implementation

### Future Enhancement Possibilities

[Erik Andersson]: The current implementation only adds outbound mappings data for **form submissions**. The generic connector could easily be extended to include mapped fields for:
- Email submissions
- Task notes
- Marketing automation flow triggers
- Any other event type

This would be a trivial change: simply remove the conditional check `if (is_form_submit)` and always include outbound mappings data.

---

## Profile Key Spaces and Uniqueness Constraints

### Key Space Behavior After Form Sync

[Erik Andersson]: After a form submission is synced to the CRM:
- The email address used in the form becomes a **unique identifier** in the email key space
- This means multiple profiles cannot share the same email within the email key space
- **However**: The CRM system can independently create new contacts with the same email address, because those aren't created through the form sync flow

### Duplicate Prevention

The merge process prevents duplicates in subsequent syncs:
- All future CRM updates use the **FCC Enterprise key space** (via CRM ID)
- The same profile is found regardless of whether data arrives via email or CRM ID
- Consent updates and contact changes won't create duplicates

### Form Resubmission Behavior

[Michal Rosikiewicz / Erik Andersson]: If a user resubmits a form with the same email:
1. Apsis finds the existing profile in the email key space
2. Form submission updates the profile (not creating a new one)
3. CRM system receives this as a matched record (not new)
4. Merge process finds the same profile via CRM ID
5. CRM data is downloaded and overwrites any form updates

This aligns with pre-filled form behavior: the form can update existing profiles, but CRM data takes precedence.

---

## Event Flow and Worker Architecture

### Worker Queue Architecture

The form submission event travels through this sequence:
1. **Common SMS Topic** (Kafka): Audience sends form submit event
2. **SMS Topic → Outbound Worker Queue** (FIFO): Shuffling/routing
3. **Audience Subscription Worker Queue** (FIFO): Processes individual events
4. **Batch Production Worker**: Accumulates and batches events
5. **Outbound Worker**: Sends batches to CRM system

[Lukasz Grabowski]: Raised clarification question about whether there should be any worker stages between form submit event and the outbound worker, suggesting flow documentation might be unclear. This was addressed in architectural discussions about adding detailed diagrams to knowledge base.

### Asynchronous Processing with Event Tracking

All steps after form submission are asynchronous:
- Events eventually reach the outbound worker (may take minutes)
- Cloudwatch logs show worker activity for specific time windows
- Logs are searchable by email or profile identifier to track a specific form submission

---

## Architectural Documentation and Knowledge Base

### Diagram Management and Updates

[Lukasz Grabowski / Erik Andersson / Michal Rosikiewicz]: Discussion about maintaining architectural diagrams:
- Current diagrams exist in the architectural diagrams folder
- New diagram created by Erik includes outbound mappings flow
- **Decision**: Create new diagram rather than modify existing PNG
- **Rationale**: Easier and quicker than editing PNG files
- **Location**: Can be added to the knowledge transfer folder or integration architectural diagrams section

### Documentation Gaps Identified

- Event subscription to outbound worker flow could be clearer in diagrams
- Outbound mappings weren't included in some existing diagrams
- Generic connector endpoint documentation could be more detailed

---

## Data Consistency and Concurrency Considerations

### Pre-filled Form Updates and CRM Master Data

[Michal Rosikiewicz / Erik Andersson]: When a form allows profile updates (pre-filled data):
- User submits form with updated data
- CRM system receives as matched record (not new)
- Merge process compares Apsis updated data with CRM data
- CRM data wins (as per master data strategy)
- Updated form fields are overwritten with CRM values

This is expected behavior but can be surprising: form updates don't persist if the CRM system doesn't also update its data.

---

## Key Takeaways

1. **Lead generation in Apsis is entirely event-driven and asynchronous** (except for the synchronous form publish operation). Forms are configured to sync to CRM systems, and form submissions trigger a multi-stage pipeline through workers and batch processing.

2. **Outbound mappings are essential and require two-stage configuration**: Map form fields to Apsis attributes first, then configure outbound mappings to translate Apsis attributes to CRM system attributes. Both must be in place for enrichment to work.

3. **The merge process solves identity fragmentation**: After CRM sync, Apsis merges the original key space (form/email) with the CRM key space using the returned CRM ID, ensuring the same profile is found in all future syncs and preventing duplicates.

4. **CRM data is the master copy**: After merge, Apsis immediately downloads all data from the CRM system, and CRM values take precedence. Form data that isn't captured by the CRM system will be cleared from Apsis.

5. **Consent is bidirectional and legally critical**: New leads must have consents exported to the CRM system immediately, and any consent changes in Apsis must be pushed to the CRM system to prevent legal violations (contacting opted-out users).

6. **The publish operation is the only synchronous step**: All other lead sync operations are asynchronous. Publish failure prevents form creation and provides immediate feedback.

7. **CRM system implementation varies**: While Apsis correctly sends outbound mapping data in requests, not all CRM systems (e.g., FCC Enterprise 12.1) fully utilize the enriched mapped fields. This is a CRM implementation issue, not an Apsis integration problem.

8. **Email becomes unique after form sync**: Once a profile is created via form submission using an email address, that email becomes unique in the email key space, preventing multiple profiles with the same email from being created through the same flow (though the CRM can independently create such duplicates).

9. **Only form submissions currently use outbound mappings**: The implementation specifically checks for form submit events before adding mapped fields. Extension to other event types (email, task notes, marketing automation) would be trivial.

10. **Profile identity resolution enables zero-duplicate syncs**: By tracking the CRM key space and CRM ID, Apsis can correctly identify profiles in all subsequent communications with the CRM system, whether updates come from form syncs, consent changes, or direct CRM system updates.

---

## Unresolved Questions and Action Items

1. **FCC Enterprise 12.1 Mapped Fields Implementation**: Erik to follow up with FCC Enterprise team about their incomplete implementation of outbound mapped fields. The system is sending the data correctly; they should be utilizing it.

2. **Future CRM Master Data Strategy**: Open question whether CRM system should remain the master data source long-term, or if strategy should change. Currently implemented for CRM-as-master, but modifiable if business requirements shift.

3. **Outbound Mappings for Non-Form Events**: Currently, outbound mappings are only applied to form submission events. Future enhancement to apply these mappings to email submissions, task notes, and marketing automation events is straightforward but not yet implemented.

4. **Diagram Location and Naming Convention**: Needs decision on final location for updated architectural diagrams showing outbound mappings in the knowledge base.

5. **Practical Implementation Exercise**: Erik recommends team members independently create forms and execute the sync flow to build intuition beyond the demonstration session.