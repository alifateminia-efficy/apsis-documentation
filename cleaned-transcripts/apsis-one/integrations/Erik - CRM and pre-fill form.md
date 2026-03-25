---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One Integrations
topics: [Form Syncing with CRM Systems, Pre-filled Forms, Profile Resolution and Merging, CRM Key Space Handling, Data Integrity and Duplicate Prevention, Attribute Updates and Data Protection, Consent Synchronization]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Audience Subscription Worker, Form Events, Outbound Worker, Kafka Queue, Batch Processing, CRM Integration Key Spaces, Profile Merging, Delta Sync Worker, Consent Mapping, Mappings Manager Service]
session_type: knowledge-transfer
subdomains: [Architecture, Inbound Flow, Outbound Flow, Generic Connector, Efficy Enterprise 12.0, Efficy Enterprise 12.1, Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This knowledge transfer session examined how CRM form syncing works in Apsis One integrations, with a focus on pre-filled forms and the critical technical challenges around profile resolution and data integrity. The team discussed the end-to-end flow of form submissions—from Audience capturing the event, through the Audience Subscription Worker, Kafka batching, and finally the Outbound Worker sending to CRM systems. A significant portion focused on the risks and requirements for pre-filled forms, particularly around preventing duplicate profile creation and protecting CRM-sourced data from being overwritten by form submissions. The session revealed several unresolved technical questions about how form fields are matched to profiles in different key spaces when CRM integrations are active.

---

## Current Form Syncing Architecture

### Overview of the Form Sync Flow

[Erik Andersson]: When a form is created on a section with a CRM system that supports form syncing (similar to event tools), the flow works as follows:

1. **Form Creation and CRM Sync Registration**: The user selects a "CRM sync" option on the form, which fires a request to the integration platform. The integration registers listeners for the specific events available on that form type.

2. **Event Types Registered**: 
   - For standard forms: `opened`, `submitted`, `started`, `viewed`
   - For event tools: `registered`, `attended`, `cancelled`, and others as specified in the architecture

### The Multi-Step Processing Pipeline

[Erik Andersson]: When someone submits a form, the flow involves four main stages:

#### Stage 1: Audience Subscription Worker

The submission is sent from Audience to the Audience Subscription Worker queue. The worker:

- Extracts the event and converts it to the internal format
- Identifies the account section and integration ID
- Attempts to extract key profile fields:
  - CRM ID (if present)
  - Email address
  - Phone number
- Adds required metadata:
  - Activity ID
  - Event timestamp
  - Event type
  - Profile key

> If no CRM ID is found, this is expected—the customer won't provide this in the form. However, **a secondary identifier must exist**: either an SMS (phone) or email address. If none of these exist, the submission is unusable because there's no way to attach it to any existing resource. First name and last name alone are insufficient.

[Lukasz Grabowski]: The profile key comes from Audience. They create a new profile with this key before sending the notification to integration.

[Erik Andersson]: For public forms, since there is no CRM ID, the profile is created in the appropriate key space based on the identifier provided: email key space for email addresses, SMS key space for phone numbers.

#### Stage 2: Kafka Batching

Once the Audience Subscription Worker has validated the data, it sends the message to Kafka. A separate batching worker:

- Listens for form events on the queue
- Groups submissions into batches
- Generates a batch ID

#### Stage 3: Outbound Worker - Sending to CRM

The outbound worker sends the batch to the CRM system. This is where the actual CRM communication happens.

[Erik Andersson]: **Critical point**: We are only notifying the CRM system that the form submission happened. We are NOT updating any profile attributes in the CRM system—not first name, last name, or email addresses. We simply inform them: "Someone submitted this form with these identifiers."

### CRM Response Handling

The CRM system can respond in three ways:

#### 1. No Action (Empty Response)
The CRM doesn't recognize the email/phone and does nothing. Apsis accepts this and moves on.

#### 2. Match Found - Existing Contact
The CRM finds an existing contact by the provided email or phone number. They respond with:

```
matched_record {
  profile_key: "<apsis_profile_key>",
  crm_id: "<crm_system_id>"
}
```

This indicates: "We found this contact in our system; here's their CRM ID."

#### 3. No Match - New Contact Created
The CRM creates a new contact:

```
new_records {
  profile_key: "<apsis_profile_key>",
  crm_id: "<newly_assigned_crm_id>"
}
```

---

## Profile Merging After CRM Response

[Lukasz Grabowski]: So when the CRM responds with a matched record, we merge the profile?

[Erik Andersson]: Yes. A merge worker reacts to these CRM responses. When the CRM provides a CRM ID (whether for a matched or new record):

1. We take the CRM ID and add it as an attribute to the form-submitted profile
2. We perform a merge between the form-created profile and our integration's key space

### Key Space Architecture and Why It Matters

[Erik Andersson]: This is crucial for understanding integrations. When you install an integration (e.g., Efficy Enterprise), we create a **bootstrapped key space** specific to that integration:

- **Efficy Enterprise 12.0**: Has its own key space (e.g., `FSC_Enterprise_12.0`) where the CRM ID is the key
- **Efficy Enterprise 12.1**: Has its own key space (e.g., `FSC_Enterprise_12.1`)
- **Dynamics**: Has a `Dynamics` key space
- **E-deal**: Has an `E-deal` key space

These custom key spaces are separate from the email or SMS key spaces.

> The reason we do this merge is critical: In the future, when the CRM system sends us a real-time update via the Delta Sync worker, we won't create a duplicate. If we only had a form-submitted profile with an email address and we hadn't merged it with our custom integration key space, the profile wouldn't be found during real-time sync—because we only search within our custom key space when processing CRM updates.

[Lukasz Grabowski]: How do we know which key space to create the profile in?

[Erik Andersson]: If you enter an email address, it goes in the email key space. If you enter a phone number, it goes in the SMS key space. For public forms with an email, that's the default.

---

## Pre-filled Forms: Requirements and Challenges

### What Are Pre-filled Forms?

[Lukasz Grabowski]: Pre-filled forms are different from standard public forms. Instead of a public form, you send it to a **specific profile inside Apsis**. When the recipient opens the form link, the system knows who they are, and the form is pre-populated with their existing profile data.

The feature is currently behind a feature flag and will be enabled for customers soon.

### The Critical Use Case: Synced CRM Contacts with Pre-filled Forms

[Erik Andersson]: This is where the biggest pitfall emerges. Let's walk through the scenario:

1. You install Efficy Enterprise 12.1
2. You sync all contacts from the CRM into Apsis
3. You create a form and enable CRM syncing
4. You send this form to a synced contact

**The problem**: When you synced the data from the CRM, the profile exists **only in the CRM key space** (e.g., `FSC_Enterprise_12.1`). It does NOT exist in the email key space because it was synced via the CRM ID, not via email.

[Lukasz Grabowski]: So if the contact's email is `john@example.com` and they have CRM ID `12345`, and we synced them via CRM ID, they only exist at the key `12345` in the FSC Enterprise key space. They might not be findable by their email address.

[Erik Andersson]: Exactly. This means when the form is submitted:

- The form service might try to find the profile by email
- It won't find the one in the CRM key space (because that's keyed by CRM ID)
- It might create a NEW profile in the email key space
- **Result**: We now have TWO profiles for the same person—one in the CRM key space, one in the email key space

This is a **duplicate profile situation**, which is precisely what the integration is designed to prevent.

---

## The Profile Resolution Problem

[Erik Andersson]: The core question that needs investigation: **How does the form tool find profiles when integrations are active?**

Specifically:

> When you send a pre-filled form to a profile that exists only in the CRM key space (keyed by CRM ID), does the form tool:
> 
> 1. Use the profile key to find the profile? (This would work correctly)
> 2. Use the email address to search? (This would find the email key space version, creating a duplicate)
> 3. Check if an integration is active and search the custom key space? (This would be the ideal solution)

[Lukasz Grabowski]: This is something we need to investigate. Let me check the behavior.

[After testing]: It appears the profile was found and correctly merged. The profile has both an email key space presence and the CRM key space presence. However, **we need to verify** the following scenario:

**Test Case**: A profile synced from the CRM (exists only in the CRM key space with no email key space presence) receives a pre-filled form link. When they submit it with their email:
- Will one merged profile result?
- Or will we end up with two separate profiles (one by CRM ID, one by email)?

---

## Acceptance Criteria and Data Protection Requirements

### Requirement 1: Profile Prefilling
> When a recipient opens a form link, their profile data is pre-filled.

**Status**: ✅ Works. Email and other synced attributes are pre-filled when a known profile opens the form.

### Requirement 2: Correct Profile Resolution
> Ensure that form submissions on pre-filled forms find and update the correct profile without creating duplicates.

**Status**: ⚠️ **To be investigated**. The specific scenario of a CRM-synced-only profile (no email key space presence) receiving a pre-filled form needs testing.

**Why this matters**: If a profile only exists in the CRM key space and we submit a form finding it by email, we could create a duplicate if no merge logic is triggered.

### Requirement 3: Protect CRM Master Data
> CRM-sourced attributes should not be overwritten by form submissions.

[Erik Andersson]: The way I interpret this: **Data on the profile inside Apsis that came from the CRM should not be overwritten by form data**, because this introduces discrepancies.

**Example of the problem**:
- CRM has: First Name = "Eric", Last Name = "Frederick"
- Profile synced to Apsis with these values
- User submits form with: First Name = "Henrik"
- If we allow overwriting, Apsis now shows "Henrik" but the CRM still has "Eric"
- Next time we sync from CRM, we overwrite the form data back to "Eric"
- **Result**: Confusion and data inconsistency

[Lukasz Grabowski]: So the principle is that CRM is the master. We don't push changes back to the CRM, and we shouldn't allow profile data mapped to CRM attributes to be overwritten locally.

[Erik Andersson]: Exactly. **Our integration rule is**: We do not support updating data from Apsis to the CRM. Therefore, any data that has a field mapping to the CRM should be read-only in Apsis, or at minimum, read-only when a profile has a CRM ID.

However, **there is an exception**: Enrichment attributes (new fields not in the CRM) should absolutely be writable. For example, if you add a "Customer Satisfaction Rating" field to the form that maps to an Apsis-only attribute, that should be updatable because it's not synced back to the CRM.

[Michal Rosikiewicz]: Should we prevent any updates to a profile that has a CRM ID?

[Erik Andersson]: No, that would be too restrictive. The product needs to distinguish between:
- **Mapped fields** (fields with CRM field mappings): Should not be overwritable
- **Enrichment fields** (new Apsis-only attributes): Should be overwritable

However, I'm not certain how the form tool's internals currently support this distinction.

### Requirement 4: Prevent CRM ID Modification

[Erik Andersson]: **CRITICAL**: There is one absolute "no, no"—you must never allow the CRM ID itself to be modified via a form field.

> If a user could somehow change their CRM ID through a form submission, we will lose complete tracking of that profile in the integration. We will definitely introduce duplicates because the merge will break.

[Lukasz Grabowski]: So we should hide the CRM ID from the form editor entirely when an integration is active.

[Erik Andersson]: Yes. If there's an active integration on the section, the CRM ID field should not be available for mapping in the form editor.

---

## Current Behavior and Uncertainties

### What Currently Happens with Mapped Attributes?

[Lukasz Grabowski]: Can customers currently overwrite CRM-mapped attributes through forms?

[Erik Andersson]: It depends on the form configuration. When you create a form, there's typically an option like "Update existing profile data" with choices like:
- Allow overwriting
- Don't override data
- Block if profile has CRM ID (this would be ideal, but I'm not sure it exists)

The problem is that the form tool may not distinguish between mapped and unmapped attributes—it might apply the same rule to everything.

### Acceptance Criteria Analysis

From the requirements document:

**Acceptance Criterion**: "CRM source attributes that differ from input are not overwritten in Apsis"

[Erik Andersson]: This depends on the form configuration option. If "allow override" is selected, they currently are overwritten. The product decision is whether this should be:
1. Always allowed (current behavior—introduces discrepancies)
2. Always blocked when a profile has a CRM ID
3. Blocked only for mapped fields, allowed for enrichment fields (most nuanced)

[Lukasz Grabowski]: I would say #3 is the right approach, but it requires the form tool to understand the concept of field mappings.

**Acceptance Criterion**: "An update request is sent to CRM for those attributes"

[Erik Andersson]: **This will never happen** with our current implementation. We do not support pushing profile attribute updates from Apsis to the CRM. This is a fundamental architectural constraint.

If the product team expects profile enrichment to sync back to the CRM (e.g., enriched attributes captured in a form appearing in the CRM system), **this is a completely new use case** that requires:
1. New development in the integration platform
2. New development in each CRM system's connector
3. Agreement on which attributes should sync back

This has never been a customer requirement, and the current architecture is designed with CRM as the single master of truth.

---

## Consent Synchronization and Pre-filled Forms

[Erik Andersson]: Consent is different from attributes. Consent synchronization is **bidirectional** and has legal/compliance reasons for this.

**Current behavior**: If a user modifies their consent in a form submission, and there's a consent mapping for that subscription to the CRM:
- We send the consent change to the CRM system
- The CRM updates the contact's consent status

[Lukasz Grabowski]: I noticed that in the pre-filled form demo, the consent checkboxes weren't being populated with the existing consent values.

[Erik Andersson]: That's a separate bug that needs to be fixed in the form tool. Once that's fixed, consent should work bidirectionally out-of-the-box because:

1. User modifies consent in form
2. Form service updates consent on the profile in Apsis
3. Integration's consent listener picks up this change
4. We send it to the CRM (if there's a consent mapping)

The integration side should already be listening for these changes via the consent mappings.

---

## Key Technical Decisions Needed

### 1. How Form Tools Locate Profiles with Active Integrations

**Investigation Required**: When a form is sent to a profile, and that profile exists only in the CRM key space:

- Option A: Form tool uses the profile key to find it (correct, no duplicate risk)
- Option B: Form tool searches by email, potentially finding a different profile (duplicate risk)
- Option C: Form tool checks if integration is active and searches the custom key space (ideal)

**Recommendation** [Erik Andersson]: The form tool should:

1. Check if an integration is active on the section
2. If yes, use the integration's key space for profile lookups
3. Only fall back to email key space if no CRM ID is present on the profile

### 2. Field-Level Protection Logic

**Decision**: Should the form tool:

- Prevent all data updates on profiles with CRM IDs?
- Use the Mappings Manager service to check which fields are mapped, and only protect those?

**Recommendation** [Erik Andersson]: 

```
To implement proper protection:

1. Make a request to Mappings Manager with:
   - account_id
   - section_id
   - integration_id

2. Response contains all field mappings and which Apsis attributes they correspond to

3. In form processing:
   - Mark mapped fields as read-only if profile has a CRM ID
   - Allow enrichment fields (not in mappings) to be written

4. Always hide the CRM ID field itself from the form editor
```

### 3. Hide CRM ID From Form Editor

**Acceptance Criterion**: Hide CRM ID from the form editor.

**Implementation**: If an integration is active on the section, the CRM ID should not appear as an available field to map in the form editor. This prevents accidental or malicious modification of the key that tracks the profile across systems.

---

## What We Know vs. What Needs Investigation

### Confirmed Working:
- ✅ Pre-filled form display (form shows existing profile data)
- ✅ Form submission on public profiles (creates new profile, syncs with CRM if configured)
- ✅ Consent bidirectional sync (once pre-fill bug is fixed)
- ✅ Profile merging when CRM responds with matching CRM ID

### To Be Investigated:
- ⚠️ **Duplicate prevention for CRM-synced-only profiles receiving pre-filled forms**
  - Test scenario: Profile exists only in CRM key space, receives pre-filled form link, submits with email
  - Required outcome: Should merge with existing profile, not create duplicate
  - Assigned to: Tomasz Kowalski
  
- ⚠️ **Form tool's profile location logic**
  - Does it respect integration key spaces?
  - Does it distinguish between different key spaces?
  - How does it handle profiles that exist in multiple key spaces?

- ⚠️ **Field-level protection in forms**
  - Can form tool distinguish between mapped and unmapped fields?
  - Does it support preventing overwrites on mapped fields only?
  - Current behavior vs. required behavior

- ⚠️ **Consent pre-fill on pre-filled forms**
  - Why aren't consent values being populated?
  - Is this a form tool bug or integration bug?

### Not Currently Supported (Requires New Development):
- ❌ Attribute updates pushed from Apsis back to CRM
- ❌ Attribute-level audit logging showing source (CRM vs. form vs. local)
- ❌ Audit trail of which fields came from CRM vs. which were updated locally

---

## Architecture Diagrams (Text Representation)

### Standard Form Submission Flow

```
Form Submission
    ↓
Audience → Audience Subscription Worker
    ↓
[Extract & Validate]
  - CRM ID (if present)
  - Email address
  - Phone number
  - Requires: CRM ID OR (Email OR Phone)
    ↓
Send to Kafka Queue
    ↓
Batching Worker
  - Groups submissions
  - Generates batch ID
    ↓
Outbound Worker
  - Sends batch to CRM
    ↓
CRM Response (3 options):
  1. No action
  2. matched_record { crm_id }
  3. new_records { crm_id }
    ↓
Merge Worker
  - Adds CRM ID to profile
  - Merges into integration key space
```

### Pre-filled Form Scenario (CRM-Synced Profile)

```
CRM System
    ↓
Integration Delta Sync
    ↓
Apsis Profile in FSC_Enterprise_12.1 key space
    ↓ (only exists here, not in email key space)
    ↓
Send Pre-filled Form Link (by email)
    ↓
[CRITICAL DECISION POINT]
Form Tool finds profile by:
  - Option A: Profile key ✅ (correct)
  - Option B: Email key space ❌ (creates duplicate)
  - Option C: Integration key space ✅ (correct)
    ↓
Form submission with enriched data
    ↓
[CRITICAL DECISION POINT]
Update mapped fields?
  - Option A: Yes, overwrite ❌ (discrepancy)
  - Option B: No, protect all ✅ (safe but restrictive)
  - Option C: No on mapped, Yes on enrichment ✅ (ideal)
```

---

## Key Takeaways

1. **CRM is the Master**: The integration architecture assumes the CRM is the single source of truth. Apsis is a replica. We sync FROM the CRM to Apsis, but not back. This must be protected in pre-filled forms.

2. **Multiple Key Spaces = Complex Profile Resolution**: When integrations are active, profiles can exist in multiple key spaces (CRM key space, email key space, SMS key space). The form tool must be aware of this and search the correct spaces to avoid duplicates.

3. **The Biggest Risk**: A pre-filled form sent to a CRM-synced-only profile (no email key space presence) could create a duplicate if the form tool searches by email and creates a new profile instead of merging with the CRM key space version.

4. **Mapped Fields Are Read-Only**: Fields that have CRM mappings should not be overwritable via forms when a profile has a CRM ID. This prevents data discrepancies between the CRM master and the Apsis replica.

5. **CRM ID Must Never Be User-Editable**: The CRM ID field should be completely hidden from form editors when an integration is active. Allowing users to change this would break the integration tracking.

6. **Enrichment is One-Way**: New attributes (not in CRM) can be collected via forms, but they will NOT sync back to the CRM. This is a fundamental limitation customers should understand.

7. **Consent is Different**: Consent is the only attribute type that syncs bidirectionally due to legal requirements. All other attribute syncing is CRM → Apsis only.

8. **Form Tool and Integration Must Coordinate**: The form tool needs visibility into:
   - Whether an integration is active
   - Which key spaces are involved
   - Which fields are mapped to the CRM
   - The profile key space to use for lookups

---

## Unresolved Questions and Action Items

### Investigation Required (Assigned):
1. **[Tomasz Kowalski]** Create scenario where a profile synced from CRM (only in CRM key space, no email key space presence) receives a pre-filled form link and test whether duplicates are created when submitted.
   - Test with a CRM contact that has an email
   - Sync the contact via CRM ID only
   - Send them a pre-filled form with their email
   - Check if one merged profile results or two separate profiles
   - Provide reproduction steps to Erik if issues found

### Follow-up Meeting:
- **With Henrik (Product)**: Next week (Monday), after Henrik's vacation
- **Attendees**: Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski, Henrik
- **Purpose**: Clarify product expectations on:
  - Whether attribute updates should flow back to CRM
  - How form tool should handle profile resolution with active integrations
  - Field-level protection strategy (all mapped fields or all fields with CRM ID?)
  - Whether audit logging is a requirement and where it should be stored

### Development Needs:
1. **Form Tool Enhancement**: Integrate with Mappings Manager to distinguish mapped vs. enrichment fields and protect mapped fields from overwrites when CRM ID is present.

2. **Form Tool Enhancement**: Check if integration is active on section before profile lookup; use correct key space.

3. **Form Tool Bug Fix**: Pre-fill consent checkboxes with existing consent values on pre-filled forms.

4. **Form Editor Fix**: Hide CRM ID field when integration is active on the section.

5. **Integration Documentation**: Need to clarify in technical documentation that attribute enrichment only happens in Apsis, not in the CRM system.

### Questions Remaining:
- How exactly does the form tool currently determine which key space to search when finding a profile?
- Does the form tool have visibility into the mappings for an integration?
- Is there a consent mapping list we can query, and does it include non-mapped subscription types?
- Where should audit logs be stored, and who has visibility into them?