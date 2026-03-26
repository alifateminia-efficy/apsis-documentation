---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One Integrations
topics: [Form syncing with CRM systems, Pre-filled forms, Profile resolution and merging, Key spaces, CRM data protection, Duplicate profile prevention, Consent management, Attribute mapping]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Audience service, Audience subscription worker, Kafka queue, Form tool, Outbound worker, Delta Sync worker, Merge worker, Consent mapping, CRM key spaces, Profile attributes]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Duplicate profiles"]
---

## Session Overview

This session covers the technical workflow of form submissions syncing with CRM systems in Apsis One, with a focus on pre-filled forms and the critical challenge of avoiding duplicate profiles. Erik Andersson walks through how form submissions are processed through the integration pipeline, while Lukasz Grabowski demonstrates the pre-filled form feature and explores the requirements for safely handling form submissions from profiles that originated in the CRM system. The team identifies key technical gaps around profile resolution when a CRM-synced profile receives a pre-filled form submission.

---

## Form Submission Workflow in Current Integration

### Overview of Form Syncing Architecture

When a form is created on a section with a CRM system that supports form syncing (similar to the event tool), there is a **CRM sync option** available. When selected, the tool fires a request to the integration service with the activity ID and the target CRM system.

[Erik Andersson]: Listeners are registered for the events available on that tool. For a form tool, this includes: `open`, `submitted`, `started`, and `viewed` events. For the form event tool, all events specified in the architecture are registered (`registered`, `attended`, `cancelled`, etc.).

### Initial Processing: Audience Subscription Worker

When a user submits a form, the event comes from the Audience service to the **audience subscription worker queue**. This worker:

1. Extracts the event and converts it to the integration format
2. Attempts to extract profile fields: CRM ID, email address, phone number
3. Adds ritual fields: activity ID, timestamp, event type, profile key

[Erik Andersson]: The profile key comes from Audience—they create a new profile with this key before sending the notification to integration. For public forms (no CRM context), the profile is created in a key space determined by the primary identifier: email key space if email is provided, SMS key space if phone is provided.

### Validation Rules

The audience subscription worker validates that sufficient identification data exists:
- If a CRM ID is present, the submission is processable
- If no CRM ID, either an email address or SMS phone number is **required**
- Without CRM ID, email, or SMS: the submission is rejected because there is no way to attach it to an existing resource

> First name and last name alone are not sufficient for profile identification.

### Batching and Outbound Transmission

After validation, the message is sent to Kafka to a worker queue. The next service **batches** form events and generates batch requests. The **outbound worker** then sends these batches to the CRM system.

[Erik Andersson]: The outbound worker is where actual sending occurs. It acts on the submit event and notifies the CRM system of the submission, but does NOT perform attribute updates on the CRM profile. It simply reports that a form submission occurred.

---

## CRM System Response Handling and Profile Merging

### Three Possible CRM Responses

When the outbound worker sends a form submission to the CRM system, three outcomes are possible:

#### 1. No Action Response
The CRM responds with an empty response, effectively saying "I accept this but I'm doing nothing with it." Nothing happens on the Apsis side.

#### 2. Matched Record Response
The CRM system recognizes the email address or other identifier and responds with:
```
{
  "matched_records": {
    "profile_key": <Apsis profile key>,
    "crm_id": <CRM ID>
  }
}
```

[Erik Andersson]: The CRM system says: "This email address matches an existing contact in my system. Here is the CRM ID." The merge worker then receives this response and performs a profile merge, adding the CRM ID to the form-created profile and merging it with the bootstrap integration key space.

#### 3. New Record Response
The CRM creates a new contact and returns:
```
{
  "new_records": {
    "profile_key": <Apsis profile key>,
    "crm_id": <CRM ID>
  }
}
```

The flow is identical to the matched record case: the CRM ID is added to the profile and a merge occurs.

### Critical: Bootstrap Key Spaces

[Erik Andersson]: When you install any CRM integration (e.g., FSC Enterprise, Dynamics, E-deal), Apsis creates a **bootstrap key space** named after the integration where the CRM ID is the key:
- `FSC_Enterprise_12.1` key space (for FSC Enterprise)
- `Dynamics` key space
- `E_deal` key space

These custom key spaces are distinct from the email key space and SMS key space. Form submissions create profiles in email/SMS key spaces, but profiles synced directly from the CRM exist only in the bootstrap key space **until a form submission triggers a merge**.

### Why Merging Matters

[Erik Andersson]: If a profile is not merged into the bootstrap key space, subsequent real-time updates from the CRM Delta Sync worker will not find it. The Delta Sync worker only searches in the bootstrap key space using the CRM ID as the lookup key. Without the merge, you have two separate profiles in Apsis:
- One in the email key space (from the form)
- One in the bootstrap key space (from CRM sync)

This creates a duplicate profile scenario.

---

## Rule: No Attribute Updates to CRM

A fundamental architectural rule: **Apsis integration never updates attributes on CRM profiles**.

[Erik Andersson]: When a form is submitted with additional data (e.g., first name, last name), the outbound worker does NOT send attribute update requests to the CRM. It only notifies the CRM that a form submission happened. The CRM may internally link this event to a contact, but no profile attributes are modified.

The reasoning is based on a uni-directional data governance model:

> The CRM is always the master. Any changes to profile data must originate in the CRM and flow into Apsis, never the reverse.

If a form submission contains a first name value that differs from the CRM's record, the CRM value takes precedence. On the next real-time sync, the Apsis profile will be overwritten with the authoritative CRM data.

---

## Pre-Filled Forms: Feature and Challenge

### What is Pre-Filled Forms?

[Lukasz Grabowski]: The pre-filled form feature allows sending a form link to a **specific profile** inside Apsis, rather than a generic public form. When a profile opens the form via the link, their existing profile data is automatically populated in the form fields.

> This is currently hidden behind a feature flag and will soon be enabled for customers.

### Demonstration Workflow

When a profile from a CRM-synced list receives a pre-filled form link:

1. The form opens with profile data pre-populated (e.g., email, last name)
2. The user may modify some fields and submit
3. The form submission should be attached to the **same profile** that opened the link

### The Core Technical Problem: Profile Resolution

[Erik Andersson]: The critical unresolved question: **Which key space is used to find the profile when a pre-filled form is submitted?**

Given a profile that was synced from the CRM:
- It exists in the bootstrap key space (e.g., `FSC_Enterprise_12.1`) with CRM ID as key
- It may or may not have been previously indexed in the email key space
- When the form is submitted, does the form service:
  - Find it via email key space?
  - Find it via the bootstrap key space?
  - Attempt both?

[Lukasz Grabowski]: If the form service only searches the email key space and the profile was only synced (never had a prior form submission), it would not be found. A new profile would be created in the email key space. The result: **two profiles for the same person**—one in the bootstrap key space, one in the email key space—creating a duplicate.

---

## Duplicate Profile Risk Scenarios

### Scenario: CRM-Synced Profile Without Prior Form Interaction

1. Profile is synced from FSC Enterprise to Apsis (`FSC_Enterprise_12.1` key space, no email key space entry)
2. A pre-filled form link is sent to this profile
3. User opens the form (data is pre-filled somehow, but mechanism unclear)
4. User modifies and submits
5. **Risk**: If form service only queries the email key space, it creates a new profile there
6. **Result**: Profile 179 (in CRM key space) and Profile X (in email key space, same email) exist separately
7. **Consequence**: Real-time CRM updates only update Profile 179; Profile X is orphaned

[Erik Andersson]: This is probably the most important use case to solve because it represents the most critical technical risk.

### Proposed Solution: Multi-Key Space Resolution

The form service should:

1. Check if an integration exists on the section using the **mappings manager service**
2. Look up mappings for the account and section
3. If mappings exist, retrieve the bootstrap key space name and integration ID
4. When processing a pre-filled form submission, search for the profile in:
   - First: the bootstrap key space (if integration exists)
   - Second: the email/SMS key space
5. If found in bootstrap key space, ensure any merge operations include that key space
6. If found in email key space but the profile has a CRM ID, check if it should be merged into the bootstrap key space

---

## CRM Master Data Protection

### The Risk of Overwriting CRM Attributes

[Erik Andersson]: If a profile is synced from CRM with `first_name = "Eric"` and `last_name = "Andersson"`, and the user submits a pre-filled form changing the first name to "Henrik", the following sequence occurs:

1. Form submission updates Apsis profile to `first_name = "Henrik"`
2. Next Delta Sync from CRM sends the authoritative data `first_name = "Eric"`
3. The profile is overwritten back to `first_name = "Eric"`

This introduces data discrepancies:
- Apsis shows "Henrik" immediately after form submission
- An email sent using this profile would display "Henrik"
- But the CRM still has "Eric"
- The discrepancy is resolved on next sync, creating confusion

### Attribute Categories and Update Rules

[Erik Andersson]: Attributes fall into two categories:

#### 1. Mapped Attributes (CRM-Synced)
These are defined in the integration's field mapping. They are owned by the CRM and should never be overwritten by form submissions.

**Proposal**: When a profile has a CRM ID and is configured with an integration:
- Mapped attributes should be **read-only** in forms
- The form editor should not allow editing these fields
- Submission data for mapped fields should be ignored

#### 2. Enrichment Attributes (Apsis-Only)
These are custom attributes not mapped to the CRM (e.g., "customer_satisfaction_rating").

**Proposal**: These can be freely updated by form submissions because:
- They don't exist in the CRM
- They are not subject to CRM sync overwriting
- They represent legitimate enrichment data collected in Apsis

#### Important Caveat: No Reverse Sync

[Erik Andersson]: Even enrichment attributes updated in Apsis are **never synced back to the CRM**. The integration does not support Apsis-to-CRM attribute updates. If customers want enrichment data in the CRM, it must be pushed to the CRM through separate mechanisms outside the form-integration flow.

> This is a fundamental architectural limitation that product stakeholders (Henrik) should be aware of upfront.

---

## Critical: Never Allow CRM ID Modification

[Erik Andersson]: There is an absolute prohibition: **The CRM ID attribute must never be modifiable by end users or forms.**

If a form field is mapped to the CRM ID and somehow allows modification, the consequences are catastrophic:
- The key in the bootstrap key space is changed
- The merge worker can no longer correlate the profile with future CRM updates
- Complete loss of tracking
- Guaranteed duplicate profile creation

**Technical Action Required**: The form editor must prevent mapping any form field to the CRM ID attribute when an integration exists on the section.

---

## Form Editor Configuration and Product Decisions

### Acceptance Criteria for Pre-Filled Forms

From the requirements document:

**Criterion 1**: When a recipient opens a form link, their profile data is pre-filled
- **Status**: Functionality demonstrated and working
- **Caveat**: Must be email-identified; CRM ID is not available for pre-fill without merging the profiles

**Criterion 2**: Data on existing CRM source that differs from input are not already in Apsis
- **Status**: Unclear / requires investigation
- **Issue**: If a CRM profile with email has never been indexed in the email key space, the pre-filled form mechanism must still find and use that profile

**Criterion 3**: Protect CRM master data from being overwritten
- **Status**: Currently not protected; depends on form configuration
- **Decision Required**: Should mapped attributes be locked, or should they be allowed to be updated (knowing they'll be overwritten on next sync)?

[Erik Andersson]: From integration perspective, nothing breaks if mapped attributes are updated—they're simply overwritten on next sync. But this introduces user-facing confusion. This is a **product decision**, not an integration limitation.

**Proposed Approach**: Block attribute updates for mapped fields if profile has CRM ID:
- Hide CRM ID field from form editor if integration is enabled on the section
- Allow updates to unmapped (enrichment) attributes freely
- Optionally display a warning: "This attribute comes from [CRM System]. Changes here will be overwritten on the next sync."

**Proposal Implementation**:

```
In form editor:
- Query mappings manager service early (first step, not last)
- If integration exists on section:
  - Hide CRM ID attribute entirely
  - Mark all mapped attributes with a "CRM-synced" label
  - Optionally prevent edits or add warning
```

### Consent Handling

[Erik Andersson]: Consent is the exception to the "no reverse sync" rule. Consent changes **are bidirectional**:

1. Profile receives pre-filled form
2. User changes consent checkbox (e.g., opts into a subscription)
3. Integration **sends this consent change to the CRM** if a consent mapping exists for that subscription

This is legally required (consent auditability).

[Lukasz Grabowski]: Currently, pre-filled forms do not show consent checkboxes. This needs to be fixed so that users can update consent, which will then be synced to the CRM automatically.

---

## Unresolved Questions and Action Items

### Investigation Required

1. **Profile resolution in form submissions**: How does the form service currently find profiles? Does it search only email key space? How should it handle profiles that exist only in the bootstrap key space?
   - **Owner**: Form tool team
   - **Depends on**: Spike / code review

2. **Duplicate profile scenario validation**: Create a test case where a CRM-synced profile (without prior form interaction) receives a pre-filled form submission. Verify whether a duplicate is created.
   - **Owner**: Tomasz Kowalski
   - **Steps**: Provided by Erik Andersson
   - **Timeline**: Before Monday meeting with Henrik

3. **Consent pre-fill in forms**: Why are consent checkboxes not currently shown in pre-filled forms? How can this be fixed to allow consent updates?
   - **Owner**: Form tool team
   - **Impact**: Consent syncing depends on this

4. **CRM ID field hiding logic**: In form editor, add conditional logic to hide CRM ID field if an integration is enabled on the section.
   - **Owner**: Form tool team
   - **Decision**: Should hiding be based on integration presence, or also on "sync to CRM" checkbox status?

[Erik Andersson]: Recommend hiding based on integration presence alone, regardless of sync option, because:
- If integration exists, CRM ID should never be modifiable
- Integration presence is determined early in the config flow
- Avoid relying on checkbox state set in later steps

5. **Attribute update semantics**: Does form editor have a "prevent override" option for mapped attributes? If so, how does it work? If not, should one be added?
   - **Owner**: Form tool team + Product
   - **Decision**: Product must decide if mapped field updates should be:
     - A) Blocked entirely when profile has CRM ID
     - B) Allowed but with warning
     - C) Allowed silently (current behavior, causes confusion)

### Technical Gaps

1. **Audit logging**: Acceptance criteria mentions audit logs showing which fields came from CRM and what changes were suggested. Currently no audit logging exists for attribute updates. This is a new capability request.
   - **Owner**: To be determined
   - **Scope**: Unclear

2. **Attribute source tracking**: System cannot currently track whether an attribute value came from the CRM or from Apsis enrichment. Would be needed for robust data governance.
   - **Owner**: To be determined

### Follow-Up Meeting

A meeting with Henrik (product owner) is scheduled for **next week (Monday)** to discuss:
- Pre-filled form requirements and acceptance criteria
- Data protection and overwrite semantics
- Whether enrichment attributes should sync back to CRM (likely: no, but confirm)
- Consent checkbox handling in pre-filled forms
- Audit logging needs

[Erik Andersson] will attend and provide integration constraints.

---

## Key Takeaways

1. **Form submissions trigger a multi-stage pipeline**: audience worker → Kafka → batch worker → outbound worker → CRM response → merge worker. Each stage has specific validation and transformation logic.

2. **Profile key spaces are critical**: Bootstrap key spaces (CRM ID key) are distinct from email/SMS key spaces. A profile synced from CRM exists only in the bootstrap space until merged, creating duplicate risk if a form submission is processed in the email space only.

3. **The merge is the solution**: When a form submission includes an email that matches a CRM contact, the CRM responds with the CRM ID, and the merge worker links the form-created profile to the bootstrap key space, preventing duplicates.

4. **CRM is the master**: No attribute updates are sent from Apsis to the CRM. The architecture is uni-directional: CRM → Apsis. Mapped attributes that are updated in Apsis are overwritten on next sync.

5. **Mapped vs. enrichment attributes**: Attributes defined in the integration's field mapping are owned by the CRM and should not be overwritable from forms. Enrichment attributes (unmapped) can be freely updated and exist only in Apsis.

6. **CRM ID is sacred**: The CRM ID attribute must never be modifiable, or the profile tracking is destroyed and duplicates are guaranteed. This must be prevented at the form editor level.

7. **Pre-filled form challenge**: The form service must resolve profiles using the bootstrap key space when an integration is enabled, not just the email key space. Current behavior may create duplicates for CRM-synced profiles that have never had a form submission.

8. **Consent is bidirectional**: Unlike regular attributes, consent changes from form submissions are synced back to the CRM. Pre-filled forms should display and allow consent updates.

9. **Enrichment doesn't reach CRM**: New attributes collected via forms are stored in Apsis only. They do not propagate to the CRM unless explicitly implemented as a new feature (not currently planned).

10. **Product and integration alignment required**: Several decisions are product-level (e.g., should mapped attributes be locked or editable with warnings?). Integration team has designed solutions but product decision is needed before implementation.