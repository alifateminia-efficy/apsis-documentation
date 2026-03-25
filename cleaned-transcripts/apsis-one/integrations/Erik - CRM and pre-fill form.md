---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One Integrations
topics: [CRM Form Syncing, Pre-filled Forms, Profile Resolution, Key Space Management, Data Synchronization, Duplicate Prevention, Consent Mapping, Form Submissions]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Audience Subscription Worker, Kafka Queue, Outbound Worker, Merge Worker, Mappings Manager Service, Form Tool, Event Tool, Delta Sync Worker, Profile Identification]
session_type: knowledge-transfer
subdomains: [Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This knowledge transfer session covers the end-to-end flow of CRM form syncing and pre-filled forms in Apsis One Integrations. The discussion combines form tool functionality with integration architecture, examining how form submissions are processed, profiles are identified and merged across key spaces, and the critical technical decisions required to prevent duplicate profiles when syncing with CRM systems. The session also explores acceptance criteria for pre-filled form features and identifies unresolved questions about profile resolution behavior.

---

## Current Form Syncing Architecture in Integrations

### How Form Syncing Works Today

When a form is created on a section with a CRM system that supports form syncing (similar to event tool functionality), an integration can be enabled to sync form activity. The flow works as follows:

[Erik Andersson]: When a user selects the CRM sync option in a form, the form tool fires a request to the integration service indicating the user wants to sync the activity (identified by activity ID) to a specific CRM system. The integration registers event listeners based on the form type:

- For **form tools**: `open`, `submitted`, `started`, and `viewed` events are registered
- For **form event tools**: all events specified in the architecture are registered (e.g., `attended`, `cancelled`)

### Form Submission Event Processing Pipeline

The form submission event flows through several stages:

1. **Audience → Audience Subscription Worker Queue**: The form submission event originates from the Audience service and is sent to an `audience subscription worker` queue
2. **Event Format Conversion**: The subscription worker picks up the event and converts it to the integration format, extracting profile fields:
   - CRM ID (if available)
   - Email address
   - Phone number
   - All required metadata (activity ID, timestamp, event type, profile key)

3. **Validation**: The worker validates that sufficient identification data exists. If no CRM ID is found (which is normal for public forms), there must be either:
   - An email address, OR
   - An SMS/phone number
   
   [Erik Andersson]: > "If there is no CRM ID, if there's no e-mail address and no SMS, then this submission is useless. We can't do anything with it because we can't attach it to any existing resource."

4. **Batching via Kafka**: The event is sent to Kafka to a batching worker that aggregates form events
5. **Outbound Worker**: The batched events are sent to the outbound worker, which performs the actual request to the CRM system

### CRM System Responses and Merge Behavior

When the integration sends a form submission to a CRM system, three outcomes are possible:

**Outcome 1 - No Action**: The CRM system responds with an empty response, acknowledging receipt but taking no action

**Outcome 2 - Matched Existing Contact**: The CRM system finds an existing contact matching the email or phone and responds with:
```
matched_record {
  crm_id: <the CRM ID for the matched contact>
}
```
In this case, the integration:
- Adds the CRM ID as an attribute to the Apsis profile created from the form
- Performs a **merge** between the form-created profile and the CRM's key space (e.g., FSC Enterprise, Dynamics, E-deal key space)

**Outcome 3 - New Contact Created**: The CRM system creates a new contact and responds with:
```
new_record {
  crm_id: <the CRM ID for the newly created contact>
}
```
The merge process is identical to Outcome 2.

### Key Space Architecture and Merging

[Erik Andersson]: When you install a CRM integration like FSC Enterprise, Apsis automatically creates a **bootstrapped key space** for that integration:
- FSC Enterprise 12.1 → `FSC Enterprise 12.1` key space (CRM ID is the key)
- Microsoft Dynamics → `dynamics` key space
- E-deal → `e-deal` key space

When a form is submitted with an email address that matches an existing CRM contact:
1. A new profile is created in Apsis with the profile key from Audience
2. The CRM responds with the CRM ID for that contact
3. The merge worker adds this CRM ID to the profile
4. A merge is performed between the email key space (or SMS key space) and the CRM's key space

This ensures that future updates from the CRM system via Delta Sync worker will find the profile using the CRM key space and won't create duplicates.

---

## Critical Principle: No Attribute Updates from Integration

[Erik Andersson]: A fundamental rule of the integration: **Apsis never updates existing attributes in the CRM system from form submissions.** When a form is submitted, the integration only notifies the CRM that the event occurred:

> "We are just letting them know like yo someone submitted this form like do what you want with it. We are not updating first names or last name or adding e-mail addresses."

The CRM system may:
- Create an internal event entry linked to the contact
- Provide a reference to this event in their campaign system
- But **never update contact attributes** from the form data

This is because **the CRM is the master of contact data**. If data needs to be modified, it must be changed in the CRM system first, where it will flow back to Apsis via synchronization.

---

## Pre-filled Forms: Core Concepts and Flow

### What Pre-filled Forms Do

[Lukasz Grabowski]: Pre-filled forms are currently hidden behind a feature flag and are being enabled for customers soon. Instead of a public form available to anyone, a pre-filled form is sent to a specific profile within Apsis. When the recipient opens the form link:

- Apsis knows which profile is accessing the form (via the link)
- Form fields are automatically populated with the profile's existing data (email, first name, last name, mapped attributes, etc.)
- The recipient can view and optionally modify the pre-filled information
- Upon submission, the form submission event is processed through the same flow as public forms

### Demonstration Flow

In the live demo, Lukasz demonstrated:
1. Creating a form synced to FSC Enterprise 12.1 with fields: email, first name, last name
2. Sending the form link to a profile that was previously synced from the CRM
3. Opening the form — the fields were pre-filled with the profile's data
4. Viewing the profile in the CRM key space — it has a CRM ID attribute

---

## Critical Challenge: Profile Resolution with Pre-filled Forms

### The Central Problem

[Erik Andersson]: The most important unresolved technical question: **Which key space is used to find the profile when a pre-filled form is submitted?**

When a pre-filled form is sent to a profile that exists only in the CRM key space (e.g., synced from FSC Enterprise with a CRM ID), and that profile has no email in the email key space, the form submission must still correctly identify the profile.

The question: Does the form tool use:
- The email key space (if an email is submitted)?
- The integration's CRM key space?
- A combination of both?

If the form tool creates a new profile in the email key space without merging it with the CRM key space, **a duplicate profile is created in Apsis** — one in the email key space (with the form submission) and one in the CRM key space (from the original sync). This is a critical failure mode.

[Erik Andersson]: > "If you only create one in the e-mail key space and this is not merged with ours, then we have suddenly created a duplicate in Apsis."

### The Duplicate Scenario

Scenario setup:
1. A contact is synced from FSC Enterprise to Apsis
2. Contact exists only in the FSC Enterprise 12.1 key space (CRM ID is the key)
3. Contact's email is known but the profile was never created in the email key space
4. A pre-filled form is sent to this profile and submitted with the email address

**Bad outcome** (currently possible): The form tool looks up by email, creates a new profile in the email key space, and attaches the form submission to it. The original CRM profile has no knowledge of this form submission.

**Correct outcome**: The form tool must somehow detect that this email belongs to an existing profile in the CRM key space, and merge both key spaces so they point to the same profile.

### Investigation Needed

[Lukasz Grabowski and Erik Andersson]: This scenario needs to be tested before the meeting with Henrik:
- Create a profile in CRM and sync it to Apsis (exists only in CRM key space)
- Manually create an email address for this profile in the CRM system
- Send a pre-filled form to this profile with that email
- Submit the form
- Verify: Does the same profile (one ID) show up when searched by both CRM ID and email?

---

## Data Protection: Preventing Overwrites of CRM-Sourced Data

### The Core Risk

[Erik Andersson]: When a profile has a CRM ID and is synced from the CRM to Apsis, the profile's attributes (first name, last name, etc.) come from the CRM. If a pre-filled form allows these attributes to be modified and submitted, a data discrepancy is introduced:

Example:
1. CRM contact: First name = "Eric", Last name = "Andersson"
2. Profile synced to Apsis: First name = "Eric", Last name = "Andersson"
3. Form submitted with: First name = "Henrik"
4. Apsis now shows: First name = "Henrik"
5. CRM still shows: First name = "Eric"
6. Next Delta Sync from CRM: Overwrites Apsis back to "Eric" (because integration doesn't support pushing updates to CRM)

Result: Discrepancy and user confusion.

[Erik Andersson]: > "If you introduce a new form field called rating or whatever, but that is not corresponding to an existing attribute mapping with the CRM system, you will introduce discrepancies which like the whole integration platform is about protecting the data from discrepancies."

### Recommended Solution: Use Field Mappings

[Erik Andersson]: The integration has a **Mappings Manager Service** that can be queried with an account and section ID to determine:
- Whether an integration exists on the section
- Which field mappings exist between Apsis and the CRM

**Proposal**: When a pre-filled form is submitted:
1. Query Mappings Manager with the section and integration ID
2. Retrieve the list of mapped fields
3. **Block updates to any field that appears in the mappings**
4. **Allow updates only to unmapped (enrichment) attributes**

This ensures that data flowing from the CRM is never overwritten by form input.

### Current Behavior

[Erik Andersson]: Whether mapped fields can be overwritten depends on the form configuration. Forms currently have an option like "Update existing profile data" or "Allow override data." The behavior differs based on this setting, but it's unclear exactly how it prevents overwrites of CRM-sourced fields.

---

## Enrichment Attributes: New Data from Forms

### Distinction: Mapped vs. Unmapped Fields

[Erik Andersson]: The approach differs for unmapped fields:

**Mapped fields** (e.g., email, first name, last name that are mapped to CRM):
- Should NOT be modifiable in the form when the profile has a CRM ID
- Any modifications will be overwritten on the next Delta Sync from CRM
- Blocking updates prevents confusion

**Unmapped/enrichment fields** (e.g., "Customer Satisfaction Rating"):
- CAN be added and modified in forms
- Will be stored in Apsis and never touched by Delta Sync
- Since there's no field mapping, the CRM won't sync these back

[Erik Andersson]: > "If you introduce a new form field called satisfactory rating which you have mapped to a completely standalone attribute inside of apps is called like satisfactory rating. Then of course you can add that new data to the profile in Apsis because we will not touch it when we do the syncs because there is no field mapping."

### Critical Limitation: No Reverse Sync

[Erik Andersson]: **Important disclaimer**: Enrichment attributes added via forms in Apsis are NOT synced back to the CRM system. The integration does not support pushing attribute updates from Apsis to the CRM:

> "This data will not exist in the CRM system. It will only exist in APSIS because we don't support the syncing of attribute changes in Apsis to the CRM, just so Henrik is also aware of this and not like, yeah, yeah, we collect all cool data in Apsis and it is automatically added in the CRM."

This is a fundamental architectural decision: **CRM is the master; Apsis enriches locally.**

---

## Consent Mapping and Bidirectional Sync

### Consent is Different from Attributes

[Erik Andersson]: Unlike attribute updates (which are unidirectional: CRM → Apsis), **consent changes are bidirectional**. When an end user modifies their subscription consent on a form:

1. User submits form indicating they want to opt-in to a subscription
2. If that subscription is in the consent mapping (mapped to CRM), the integration sends the consent change to the CRM
3. CRM updates the contact's consent status
4. Delta Sync brings that consent status back to Apsis

This is necessary for legal compliance and data accuracy.

### Pre-filled Form Consent Issue

[Lukasz Grabowski]: In the current implementation, pre-filled forms don't populate consent checkboxes. When viewing a pre-filled form, subscription consents are not shown/checked even though the profile has consent data in Apsis.

**Current behavior**: Consent checkboxes are empty in pre-filled forms
**Expected behavior**: Consent checkboxes should be pre-filled with the profile's current consent status

This is a form tool issue to be fixed, but once fixed, consent bidirectional sync should work automatically because:
- Form tool will change the consent attribute on the profile
- Integration will detect the change via listener
- Integration will send it to CRM (if subscription is in consent mapping)

---

## Acceptance Criteria and Outstanding Questions

### Requirements from Henrik (Product)

Based on the session review of acceptance criteria:

1. ✅ **When recipient opens form link, profile data is pre-filled**: Confirmed working in demo. Email and attributes are pre-populated.

2. ⚠️ **Form link is locked to profile**: Email is locked, but needs verification for SMS-identified profiles.

3. ❌ **CRM ID is not available to edit**: Currently, CRM ID field might be editable in form designer. **Decision needed**: Should CRM ID be hidden from editor when an integration exists on the section?
   
   [Michal Rosikiewicz and Erik Andersson consensus]: Hide CRM ID field from editor if integration is enabled on the section. Do NOT allow modification under any circumstances.
   
   [Erik Andersson]: > "If you for whatever reason would be able to actually update your CRM ID, I can tell you that we will be in deep **** because this is one of the things you can under no circumstance do if you have an integration because we will lose complete track of that profile."

4. ⚠️ **CRM source attributes that differ from input are not overwritten in Apsis**: Depends on form configuration; currently they CAN be overwritten. Product decision needed: Block overwrites of mapped fields when profile has CRM ID?

5. ❌ **Requests sent to CRM for enrichment attributes**: Not supported today. Integration only sends form events, not attribute updates. This is new functionality that requires CRM side support too.

6. ❓ **Submissions do not create duplicate profiles**: Needs testing. Must verify profile resolution behavior.

7. ❓ **Audit log shows field sources and changes**: Unclear if Apsis has audit logs for form submissions and attribute updates. If required, needs implementation.

8. ✅ **Consent can be shown and updated**: Should work once pre-filled form consent population is fixed.

---

## Critical Gotchas and Warnings

### Gotcha 1: Email Key Space vs. CRM Key Space

[Erik Andersson]: When a profile is synced from CRM to Apsis, it exists **only in the CRM key space** (e.g., FSC Enterprise 12.1 key space where CRM ID is the key). It does NOT exist in the email key space, even though the profile may have an email address.

If a form submission later creates a profile in the email key space without merging to the CRM key space, the two profiles become separate, leading to duplicates.

### Gotcha 2: CRM ID as a Form Field

[Erik Andersson]: Under no circumstance should a form allow editing of the CRM ID field if an integration is installed:

> "If you have an integration, you can never be allowed to do this... if you were to do this, then you will modify the key. And yeah, that will have consequences."

Modifying the CRM ID breaks the entire sync mechanism because the integration uses the CRM ID to track profiles in Delta Sync.

### Gotcha 3: Delta Sync Overwrites Form Data

[Erik Andersson]: If a mapped attribute is modified in a form (e.g., first name changed from "Eric" to "Henrik"), and then a Delta Sync occurs from the CRM, the CRM's value ("Eric") will overwrite the form submission ("Henrik"):

> "They will then override whatever you might have submitted in the form will be cleared out and we will re add Eric because of the reason that we don't support updating the data from Appsys to the CRM."

This is by design but causes confusion if users aren't warned.

### Gotcha 4: Enrichment Attributes Don't Sync to CRM

[Erik Andersson]: New attributes added via form submissions (e.g., "satisfaction rating") only exist in Apsis. They are never propagated to the CRM system. The integration doesn't support pushing profile updates back to CRM.

### Gotcha 5: Consent Mapping Scope

[Erik Andersson]: Consent changes are only synced to CRM if the subscription is in the consent mapping. If a form adds consent for a subscription that isn't mapped, the integration won't react because there's no listener registered for it.

---

## Architecture Decisions and Recommendations

### Approach 1: Query Mappings Manager (Recommended by Erik)

When a pre-filled form is submitted to a profile with a CRM ID:

1. Query the Mappings Manager service with account + section + integration ID
2. Retrieve mapped fields
3. Automatically prevent updates to mapped fields
4. Allow updates only to unmapped enrichment fields
5. For unmapped fields, accept the update and store in Apsis (no CRM sync)

**Advantage**: Protects CRM data while allowing enrichment
**Complexity**: Requires form tool to query integration service

### Approach 2: Block All Updates for Profiles with CRM ID

- If profile has a CRM ID, prevent ANY attribute modification in the form
- Only allow new unmapped fields to be added

**Advantage**: Simplest, safest
**Disadvantage**: Prevents enrichment, blocks legitimate form use cases

### Approach 3: Product Configuration

- Add form configuration option: "Allow field updates for mapped attributes"
- Default: FALSE (don't allow overwrites)
- Let customers decide per form

**Advantage**: Flexible
**Disadvantage**: Requires customer education; risk of misconfiguration

---

## Investigation Tasks and Action Items

### Priority 1: Profile Resolution Testing
**Owner**: Tomasz Kowalski (with Erik's guidance on scenario setup)
**Deadline**: Before meeting with Henrik (next Monday)

**Scenario to test**:
1. Sync a contact from FSC Enterprise 12.1 to Apsis (exists only in CRM key space)
2. Ensure contact has an email address in the CRM
3. Send pre-filled form to this profile with the email field
4. Submit the form with the email address
5. Verify:
   - Search by CRM ID → get profile ID X
   - Search by email → get profile ID Y
   - Are X and Y the same? (they should be)
   - Is there only one profile or two?
   - Does the form event appear on the correct profile?

**Why it matters**: Determines whether pre-filled forms will create duplicates

### Priority 2: Current Override Behavior Documentation
**Owner**: Lukasz Grabowski / Form Tool team
**Deadline**: Before Henrik meeting

Clarify: When is it currently possible to override mapped fields? What form settings control this?

### Priority 3: Attribute Source Tracking
**Owner**: TBD
**Deadline**: After Henrik meeting

Determine if Apsis can track which attributes came from CRM vs. were enriched locally, and if audit logs exist.

### Priority 4: CRM ID Visibility in Form Editor
**Owner**: Form Tool / Product
**Deadline**: Sprint planning

**Decision**: Hide CRM ID field from editor when section has active integration

---

## Key Takeaways

1. **Form submissions trigger the same integration flow as event tool submissions**: Forms are converted to integration events and processed through Audience Subscription Worker → Kafka → Outbound Worker → Merge Worker.

2. **CRM is the master; Apsis is a read-mostly cache**: The CRM owns contact data. Updates flow CRM → Apsis via Delta Sync. Form submissions only notify the CRM of events, they don't update CRM attributes.

3. **Key spaces must be carefully managed**: Profiles synced from CRM exist only in the CRM key space (e.g., FSC Enterprise 12.1). Form submissions to these profiles must ensure the key spaces are merged to prevent duplicates.

4. **Pre-filled forms create a new risk**: When a form is sent to an existing profile (especially one with a CRM ID), the form tool must correctly identify the profile by querying the right key space. Incorrect profile resolution creates duplicates.

5. **Data discrepancies are the enemy**: Allowing form submissions to modify attributes that are mapped to the CRM introduces discrepancies because:
   - User changes attribute in form (first name: Eric → Henrik)
   - Integration doesn't support pushing this change to CRM
   - Next Delta Sync from CRM overwrites it back (Eric)
   - User sees their change reverted

6. **Enrichment is allowed, but local-only**: New unmapped attributes can be added and modified in forms, but they won't sync to the CRM. This is by design.

7. **Consent is bidirectional**: Unlike attributes, consent changes are synced both ways (form → integration → CRM → Apsis).

8. **The CRM ID field is sacred**: Allowing edit/modification of the CRM ID field in a form will break the sync mechanism and create duplicates. This must be prevented at all costs.

---

## Unresolved Questions and Follow-up Items

1. **Profile Resolution Algorithm**: How does the form tool find the correct profile when submitted? Does it query only the email key space, or can it query the CRM key space? How is the key space determined?

2. **Duplicate Prevention Mechanism**: What prevents a profile synced from CRM (only in CRM key space) from creating a duplicate when a form is submitted via email address?

3. **Approval for Field Mapping Protection**: Should mapped fields be automatically protected from overwrites when a profile has a CRM ID? This is a product decision.

4. **Consent Pre-fill in Forms**: Why aren't consent checkboxes pre-filled in pre-filled forms? Is this a bug or intentional?

5. **Audit Logging**: Where should form submission source information (CRM vs. form) and change history be stored? Does Apsis have audit log infrastructure?

6. **Enrichment Attribute Sync to CRM**: If Henrik wants enrichment attributes to appear in the CRM, this requires new development on both integration and CRM side. Confirm scope and priority.

7. **SMS Key Space Behavior**: How does profile resolution work for SMS-only profiles (no email)? Does the same duplication risk apply?

**Meeting with Henrik scheduled for**: Next week (Monday), to discuss acceptance criteria and product decisions around data overwrites and CRM sync behavior.