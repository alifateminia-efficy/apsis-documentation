---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One - Integrations
topics:
  - CRM Form Syncing Architecture
  - Profile Resolution and Key Space Management
  - Pre-filled Form Workflows
  - Data Protection and Duplication Prevention
  - Attribute Mapping and Overwrite Behavior
  - Consent Synchronization
speakers:
  - Erik Andersson (Integration Architecture)
  - Lukasz Grabowski (Form Tools & Integration)
  - Michal Rosikiewicz (Product/Technical Lead)
  - Tomasz Kowalski (Testing/QA)
key_components:
  - Audience Subscription Worker
  - Kafka Message Queue
  - Form Event Batching
  - Outbound Worker
  - CRM Integration Key Spaces
  - Profile Merge Worker
  - Consent Mapping
  - Mappings Manager Service
session_type: knowledge-transfer
---

## Session Overview

This session combined knowledge from form tools and integration architecture to analyze requirements for pre-filled CRM forms. The discussion covered how form submissions currently sync to CRM systems, the critical importance of maintaining correct profile resolution across multiple key spaces to prevent duplicates, and the technical challenges of extending this to pre-filled forms where profiles already exist in the CRM key space. Key topics included data protection rules, attribute mapping behavior, and the distinction between what can be enriched locally in Apsis versus what must remain protected from CRM sources.

---

## Current Form-to-CRM Sync Flow

### How Form Syncing Works Today

When a form is created on a section with a CRM system that supports form syncing, users can enable a **CRM sync option** to synchronize form activity to their connected CRM system. [Erik Andersson]

The process initiates when the form tool fires a request to Integration, specifying the user wants to sync an activity with a particular ID to a specific CRM system. Integration then registers event listeners based on the tool type:

- **Form Tool**: Registers `open`, `submitted`, `started`, and `viewed` events
- **Form Event Tool**: Registers `attended`, `cancelled`, and other events specified in the architecture

### Event Processing Pipeline

When a form submission occurs, the event flows through multiple workers:

1. **Audience Subscription Worker** (first step)
   - Receives the submit event from Audience
   - Converts the event to the format needed by Integration
   - Extracts profile fields: CRM ID, email address, phone number
   - Adds ritual fields: activity ID, timestamp, event type, profile key
   - [Erik Andersson]: "If there is no CRM ID then we require there to be like either an SMS or an e-mail. Because that's the secondary like type of identification."
   - **Critical validation**: If no CRM ID, email, or SMS exists, the submission is considered useless and cannot be attached to any resource. First name and last name alone are insufficient for identification.

2. **Kafka Queue**
   - Message is batched and sent through Kafka (no logging available here)

3. **Form Event Batching Worker**
   - Listens to form events from the past 5 minutes
   - Generates a batch with the submit event and adds the activity ID

4. **Outbound Worker** (actual sending)
   - Sends the batch to the CRM system
   - Handles retries on failure (e.g., 504 errors from CRM outages)
   - **Important distinction**: Integration does NOT update any profile attributes in the CRM (first names, last names, email addresses). It only notifies the CRM that a form submission occurred.
   - [Erik Andersson]: "We are just letting them know like yo someone submitted this uh form like do what you want with it."

### Three Possible CRM Responses

When Integration sends a form submission event, the CRM system can respond in three ways:

1. **Ignore the event** - CRM responds with an empty response, indicating acceptance but taking no action

2. **Match to existing contact** - CRM recognizes the email address (or phone number) matches an existing contact and responds with the CRM ID for that contact. Integration then:
   - Takes the CRM ID and adds it as an attribute to the form profile
   - Performs a merge between the form-created profile and the bootstrapped integration key space (FSC Enterprise, Dynamics, E-deal, etc.)
   - [Lukasz Grabowski]: "So we receive this, I can merge this this profile with the existing profile with CRM Key."
   - [Erik Andersson]: "The merge worker which reacts on these responses"

3. **Create new contact** - CRM creates a new contact and responds with the new CRM ID. The merge flow is identical to scenario 2.

### Key Space Architecture for CRM Integrations

When you install a CRM integration (e.g., FSC Enterprise, Dynamics, E-deal), Apsis creates a **bootstrapped key space** specific to that integration where the CRM ID is the key:

- **FSC Enterprise 12.1**: Has its own key space (`FSC_Enterprise_12.1`) where CRM ID is the primary key
- **Dynamics**: Has `dynamics` key space
- **E-deal**: Has `e_deal` key space

[Erik Andersson]: "If you do it for dynamics, you have a dynamics key space. If you do it for E deal, you have a E deal the key space and this is going to be very important when we come to the prefield. Prefilled forms later on."

Public forms, by contrast, create profiles in the **email key space** or **SMS key space** depending on what identifier is provided, because they have no CRM context initially.

---

## Pre-Filled Form Functionality

### What Pre-Filled Forms Are

Pre-filled forms are currently behind a **feature flag** and will be enabled for customers soon. Unlike public forms, pre-filled forms are sent to specific profiles already in Apsis.

[Lukasz Grabowski]: "If I send e-mail to myself with link to the form which I created so if I open it... through this link, yeah, we know that this is this is my profile."

When a recipient opens the pre-filled form link:
- The system knows which profile submitted the form
- Form fields are pre-populated with that profile's data (email, last name, other attributes)
- The user can modify these fields and submit

### The Critical Pre-Fill Problem: Profile Resolution Across Key Spaces

The core challenge Erik identified is ensuring that pre-filled forms submitted by profiles that were synced from the CRM do not create duplicate profiles in Apsis. This is the **most important technical use case** for the integration side. [Erik Andersson]

**Scenario that needs investigation:**

1. A contact exists in the CRM system
2. Contact is synced to Apsis via the FSC Enterprise 12.1 integration → profile exists only in the `FSC_Enterprise_12.1` key space (identified by CRM ID)
3. That contact has an email address, and a pre-filled form is sent to them via email
4. User opens the form and submits it with (potentially modified) data
5. **Critical question**: Which key space does the form tool use to find and update this profile?

**Current uncertainty:** [Erik Andersson]: "Are you using it through the export key for with the profile key? Are you doing it with via e-mail key space with the e-mail address? Or how how how are you finding the profile you should add the event to?"

[Lukasz Grabowski]: "For sure there is no export, there is a call to to audience for the profile and it depends on the key space which we find using this ID. Which is somehow processed through the couple of services... But I'm not sure how it is done in forms because in forms and there is difference between forms and event tool functionality."

**The duplicate profile risk:**

If the form tool only creates/updates a profile in the **email key space** and does not merge with the **FSC_Enterprise_12.1 key space**, the result is two separate profiles in Apsis representing the same person:
- One in the CRM key space (with CRM ID)
- One in the email key space (with form submission event)

If only the email key space profile receives the form submission event and these are not merged, then:
- The CRM key space profile remains unchanged
- Future updates from the CRM (Delta Sync) would still go to the CRM key space profile, not the email profile
- Customer would see inconsistent data and lost form submission history

[Erik Andersson]: "If you search on the e-mail, you get one's profile ID and then you search for the CRM ID and you get another ID and only the one with the e-mail has to submit event. Then we need to go back to the drawing board with that flow."

---

## Data Protection and Attribute Mapping Strategy

### The Master-Detail Rule: CRM is the Master

Integration operates under a fundamental principle: **the CRM system is the master data source**. Apsis does not push attribute updates back to the CRM system.

[Erik Andersson]: "The master is CRM, right? So we are not, we don't touch the data. I mean they don't touch data which are coming from from us. In terms of attributes in this case."

The outbound worker only sends **events** (form submissions, opens, etc.) to the CRM, not attribute updates. The CRM decides what to do with these events internally.

### Why Overwriting CRM-Synced Data Creates Problems

If a profile's attribute (e.g., first name) comes from the CRM and is then overwritten in Apsis via a form submission, it creates a **discrepancy**:

**Example given by Erik:**
- CRM has: First Name = "Eric", Last Name = "Frederick"
- Profile synced to Apsis with these values
- User submits pre-filled form and changes: First Name = "Henrik"
- If Apsis accepts this overwrite:
  - CRM still shows: Eric Frederick
  - Apsis shows: Henrik Frederick
  - Any email sent would display Henrik (incorrect data)
  - Next sync from CRM overwrites back to Eric, clearing the form change
  - **Result**: Data discrepancy and lost user input

[Erik Andersson]: "We will then override whatever you might have submitted in the form will be cleared out and we will re add Eric because of the reason that we don't support updating the data from Appsys to the CRM like what whatever you change manually or in the form submissions like we will completely disregard and and overwrite."

### Recommended Approach: Protection via Field Mapping

Erik's recommendation is to use the **Mappings Manager Service** to determine which attributes are protected:

1. Look up the integration for the account and section
2. Use the Mappings Manager to get all mappings for that integration
3. For any attribute that appears in the mappings (indicating it's synced from CRM):
   - **Do not allow overwriting** of that attribute from the form
   - Prevent the field from being edited in the form if the profile has a CRM ID

[Erik Andersson]: "All attributes that exists in those mappings, you should not be able to overwrite that data with data from the form because like all of that data will come from the CRM system and if that data is to be changed, it should be changed from the CRM system instead."

**Product decision needed**: Should the form tool:
- **Option A**: Allow overwriting mapped attributes but accept that they'll be reset on next sync (confusing UX)
- **Option B**: Prevent overwriting of mapped attributes entirely (protects data integrity)
- **Option C**: Block overwriting only if the profile has a CRM ID (more nuanced)

### Exception: Enrichment Attributes (New Data)

Attributes that are **not in the CRM mappings** can and should be updated/enriched via forms:

**Example:** A form field "Customer Satisfaction Rating" mapped to a new Apsis attribute "satisfaction_rating" that doesn't exist in CRM.

[Erik Andersson]: "If you introduce. Use a new form field called satisfactory rating which you have mapped to a completely standalone attribute inside of apps is called like satisfactory rating. Then of course you can add that new data to the profile in Apsis because we will not touch it when we do the sinks because there is no field mapping."

**Critical limitation**: This enriched data will **only exist in Apsis**, not in the CRM, because Apsis does not support pushing attribute updates to CRM. [Erik Andersson]: "This data will not exist in the CRM system. It will only exist in APSIS because we don't support the sinking of attribute changes in Apsis to the CRM."

### Absolute Prohibition: Modifying CRM ID

**Never allow a form to update the CRM ID itself.** This would break profile tracking entirely.

[Erik Andersson]: "Let's say that you create a field called CRM ID and you map that to a the CRM ID field inside of Apsis if you for whatever reason would would be able to actually update your CRM ID. I can tell you that we will be in deep **** because this is one of the things you can under no circumstance do if you have an integration because we will lose complete track of that profile. Then you will definitely introduce duplicates."

**Current vulnerability**: It may be possible to add a field that maps to CRM ID in the form tool.

**Mitigation required**: If an integration exists on the section, the CRM ID attribute must be hidden from the form editor.

---

## CRM ID Field Visibility and Form Editor Controls

### Current Behavior

The form tool's "CRM sync" checkbox appears in the last or second-to-last step of form creation, making it difficult to conditionally show/hide fields based on whether integration is enabled.

### Proposed Controls

**[Michal Rosikiewicz]**: "Hide CRM ID from editor" if an integration is enabled on the section.

The logic should be:
1. Check if the section has an active CRM integration
2. If YES: Hide the CRM ID attribute from the form editor (make it unavailable for mapping)
3. If NO: Allow CRM ID to be used in the form

**Implementation consideration**: Erik suggests making a single request to the Mappings Manager Service early in the form creation flow and caching the result. [Erik Andersson]: "You should do it like in the first time you need it and then preserve that data until the flow is done."

The form tool can do this request:
- In the first step when the section is selected, or
- Whenever the CRM sync checkbox is toggled

---

## Consent Synchronization in Pre-Filled Forms

### Bidirectional Consent Sync

Consent is **bidirectional** and must sync between Apsis and CRM due to legal/compliance requirements.

[Erik Andersson]: "Here there are legal reasons as to why we need to have a bidirectional sync."

When a user modifies consent via a pre-filled form:
1. They change their subscription consent status in the form
2. Apsis updates the profile's consent attribute
3. Integration's outbound worker **automatically sends this consent change to the CRM** if there is a consent mapping for that subscription

[Erik Andersson]: "If an end user modifies their consent... and then they say yes, I do want consent on this specific subscription that we will send to the CRM system if you have a mapping for it."

### Current Pre-Filled Form Gap

Pre-filled forms currently **do not show consent checkboxes**, so users cannot update their consent via the form. [Lukasz Grabowski]: "Because it doesn't work right now for prefil as you can see there is no checkbox here."

**Fix required**: The form service needs to pre-fill and display consent checkboxes for subscriptions that have consent mappings.

### Integration's Role in Consent Changes

Integration only reacts to consent changes for subscriptions that are in the **consent mapping**:

[Erik Andersson]: "We are only reacting on subscriptions that are in our consent mapping if you have like consent mapping a which is synced with the CRM. We will send those changes to the CRM. If you add it for a subscription B, which does not exist in our consent mapping, we will not react on that because we don't have a listener registered for it."

---

## Acceptance Criteria and Requirements Validation

The following acceptance criteria were discussed against the current architecture:

### 1. Recipient Profile Data Pre-Fills ✓ (Verified)
When a recipient opens a form link, their profile data is pre-populated in the form fields.

**Status**: Verified working in the demo. Email and other attributes were pre-filled correctly.

### 2. CRM ID Field Protection ⚠️ (Requires Implementation)
The CRM ID attribute must not be available for editing or mapping when an integration is enabled on the section.

**Status**: Currently possible to add CRM ID mapping; needs to be hidden from editor.

**Implementation**: Hide CRM ID field if integration is enabled on the section.

### 3. CRM Source Attributes Not Overwritten ⚠️ (Depends on Form Tool Implementation)
Attributes that come from the CRM (mapped via Mappings Manager) should not be overwritten by form submissions when the profile has a CRM ID.

**Status**: Form tool has an "update existing profile data" / "allow override" option, but it's unclear which approach is default or recommended. Needs investigation and product decision.

**Current behavior**: Depends on form configuration options selected during form creation.

**Recommended approach**: Either prevent updates entirely or provide granular control (allow enrichment of non-mapped attributes only).

### 4. Profile Submissions Don't Create Duplicates ⚠️ (To Be Investigated)
When a pre-filled form is submitted by a profile that exists in the CRM key space, no duplicate profile should be created in the email key space.

**Status**: UNKNOWN - needs testing and verification.

**Action**: [Tomasz Kowalski] to create and test the exact scenario:
- Sync a contact from CRM to Apsis
- Send a pre-filled form to that contact's email
- Submit the form
- Verify that only one profile exists (in the CRM key space, merged appropriately)

**Risk**: If the form tool finds/creates the profile using only the email key space and doesn't merge with the CRM key space, duplicates will be created.

### 5. Attribute Update Requests Sent to CRM ✗ (Not Supported)
The acceptance criteria includes "update request is sent to CRM for those attributes."

**Status**: This is NOT currently supported and was flagged as a significant limitation.

[Erik Andersson]: "We are never sending an update request to the CRM and this is where I wonder like if they expect these like enrichment attributes to be added in the CRM... we can't because we don't support this today."

**Current architecture**: Integration only sends **events** to the CRM, not attribute value updates.

**If needed**: This would require new development on both the Integration side and coordination with CRM systems to define how they accept attribute updates.

### 6. Audit Logs ❓ (Unclear Requirement)
Acceptance criteria mentions "Audit log shows which fields came from CRM, what changes were suggested and what was stored locally."

**Status**: Unclear what this means or where audit logs should be stored.

[Erik Andersson]: "Where would those audit logs be stored? Who would check it and who adds them?"

**Current limitation**: Apsis currently stores source information only for consent changes, not for all attribute updates.

**Action**: Requires clarification on what audit logging is needed and where it should be visible.

### 7. Consent Display and Update ✓ / ⚠️ (Partially Implemented)
Consent should be shown and updatable in pre-filled forms.

**Status**: 
- Bidirectional sync of consent with CRM is supported
- Pre-filled forms don't currently show consent checkboxes
- **Fix required in form service**: Pre-fill consent checkboxes based on the profile's current consent status

---

## Known Pitfalls and Edge Cases

### Pitfall 1: Email-Only Identification for CRM-Synced Profiles

[Erik Andersson]: "So this is the biggest pitfall of them all."

**Scenario**: A profile is synced from CRM (exists in FSC_Enterprise_12.1 key space with CRM ID). A pre-filled form is sent via email. The form tool needs to find the profile to add the form submission event.

**Problem**: If the form tool searches only by email address, it finds a profile in the email key space (or creates a new one), not the profile in the CRM key space.

**Why it matters**: The CRM key space profile has the CRM ID and is connected to the real contact in the CRM. The email key space profile is disconnected from that chain. Future CRM syncs won't update the email-only profile.

**Investigation needed**: Verify how the form tool actually finds profiles when processing pre-filled form submissions.

### Pitfall 2: Form Tool's Key Space Lookup Mechanism

[Lukasz Grabowski]: "For sure there is no export, there is a call to to audience for the profile and it depends on the key space which we find using this ID."

The form tool makes a call to Audience to find the profile, but it's unclear:
- How it determines which key space to use
- Whether it checks multiple key spaces if the first lookup fails
- Whether it merges across key spaces appropriately

**Difference from Event Tool**: Event tool has its own path for profile lookup; forms use a different path. The differences need to be documented.

### Pitfall 3: Attributes That Are "Protected" vs. "Enriched"

The form tool must distinguish between:
- **CRM-sourced attributes** (protected, should not be overwritten if profile has CRM ID)
- **Enrichment attributes** (new data not in CRM, can be freely updated)

Currently, the form tool may not make this distinction. This requires integration with the Mappings Manager Service to determine which attributes are mapped from the CRM.

### Pitfall 4: Form Submissions Only Send Events, Not Attribute Updates

The acceptance criteria seemed to expect attribute updates to reach the CRM:

[Erik Andersson]: "We are never sending an update request to the CRM... If they want the profile enrichment to happen in the CRM too. This is something that needs to be discussed with product on the CRM side as well."

This is a fundamental architectural limitation. Apsis sends **events** to CRM, but CRM systems must choose how to handle them. Apsis does not push attribute changes back to CRM.

---

## Technical Validation and Testing

### Live Demo Observations

[Lukasz Grabowski] demonstrated the pre-filled form functionality with a real profile synced from FSC Enterprise 12.1:

1. **Profile identification**: Successfully identified that the profile exists in the `FSC_Enterprise_12.1` key space (CRM ID = 179)
2. **Pre-fill verification**: Email and last name were pre-filled correctly
3. **Form submission**: Submitted the form with modified data
4. **Form event recording**: Form submission event was recorded in the profile's form interaction history
5. **Key space merge status**: Profile also appears in email key space after submission, but **first name update was missing**

[Michal Rosikiewicz]: "We are missing first name update and this is most probably because it was there before we started updating and it didn't merge this data."

### Outstanding Testing

**To be completed by [Tomasz Kowalski] before follow-up meeting with Henrik:**

1. Test the exact duplicate prevention scenario:
   - Create a contact in CRM
   - Sync to Apsis (appears in CRM key space only)
   - Send pre-filled form to that email address
   - Submit form with changes
   - Verify no duplicate profile is created
   - Verify form submission event is on the original CRM key space profile

2. Test attribute overwriting behavior:
   - Determine current form tool behavior for mapped vs. non-mapped fields
   - Clarify the "update existing profile data" / "allow override" option functionality

3. Test consent pre-filling:
   - Verify consent status is fetched and displayed in pre-filled forms
   - Verify consent changes sync back to CRM

---

## Product Decisions Required

Several items require product-level decisions:

### 1. Attribute Overwriting Policy
**Question**: Should pre-filled forms allow end users to overwrite attributes that came from the CRM?

**Options**:
- A. Block all overwrites if profile has CRM ID
- B. Allow overwrites but accept they'll be reset on next sync
- C. Allow overwrites of non-mapped attributes only (requires Mappings Manager integration)

**Erik's position**: "I I I don't see a use case as to why you would allow this because it will only introduce confusion but... I I see this as a product decision and then of course the technical solution will need to support whatever that decision is."

### 2. Enrichment Data in CRM
**Question**: Should enriched attributes (data collected in Apsis but not from CRM) be pushed back to the CRM?

**Current answer**: Not supported today.

**Implication**: If this is desired, it requires new development in both Integration and CRM connector services.

### 3. Audit Logging Requirements
**Question**: What audit information should be captured for form submissions to CRM-synced profiles?

- Which fields came from CRM?
- What changes did the user suggest?
- What was actually stored locally?

**Note**: This requirement in the acceptance criteria is unclear and may need refinement.

### 4. CRM ID Field Visibility
**Decision made**: Hide CRM ID attribute from form editor if an integration is enabled on the section.

---

## Key Takeaways

1. **Form syncing to CRM works today**, but only for form submission events. The CRM system decides how to handle these events; Apsis does not update CRM attributes.

2. **Pre-filled forms introduce a critical technical challenge**: ensuring that form submissions from profiles synced from the CRM don't create duplicate profiles in Apsis. The profile resolution mechanism across multiple key spaces (CRM key space, email key space) needs to be verified and potentially modified.

3. **CRM is the master data source**. Any attributes mapped from the CRM should not be overwritten by form submissions. This requires integration with the Mappings Manager Service in the form tool to identify protected vs. enrichable attributes.

4. **Enrichment is supported locally, but not bidirectionally**. New attributes (not from CRM) can be added via forms and stored in Apsis, but Apsis does not push these back to the CRM system.

5. **Consent is bidirectional**. Consent changes made via forms will be sent to the CRM if a consent mapping exists, due to legal requirements.

6. **CRM ID must be protected**. If an integration is enabled on a section, the CRM ID attribute must be hidden from the form editor to prevent accidental modification.

7. **Multiple key spaces complicate profile resolution**. Integration installations create bootstrapped key spaces (FSC_Enterprise_12.1, dynamics, etc.) where the CRM ID is the key. Form submissions need to be aware of these key spaces and perform appropriate merges to avoid duplicates.

8. **The form tool and event tool use different paths**. Profile lookup and merging may behave differently between these two tools; the form tool's exact mechanism needs to be documented.

---

## Unresolved Questions

1. **Profile Resolution in Form Tool**: How exactly does the form tool find the profile to update when a pre-filled form is submitted? Does it check multiple key spaces? Does it perform merges?

2. **Duplicate Prevention Test**: Will a pre-filled form submission by a profile that exists only in the CRM key space create a duplicate in the email key space, or will it properly merge?

3. **Current Override Behavior**: What is the current behavior of the "allow override data" option in forms when a profile has mapped attributes from a CRM integration?

4. **Consent Pre-fill Implementation**: How should consent status be fetched and pre-filled in form checkboxes? Is this currently implemented for pre-filled forms?

5. **First Name Update Issue**: Why was the first name not updated in the live demo when other attributes were? Is this a merge issue or a form submission issue?

6. **Audit Logging Scope**: What exactly should audit logs capture, where should they be stored, and who should have access to view them?

7. **Enrichment Data Sync**: Is there a future requirement to push enriched attributes (collected in Apsis) back to the CRM system? This is not currently supported.

---

## Action Items

| Item | Owner | Timeline | Notes |
|------|-------|----------|-------|
| Test duplicate prevention scenario | Tomasz Kowalski | Before follow-up with Henrik | Create test profile in CRM, sync, send pre-filled form, verify no duplicate |
| Test attribute overwrite behavior | Tomasz Kowalski | Before follow-up with Henrik | Clarify form tool's current behavior with mapped vs. unmapped fields |
| Investigate consent pre-fill | Form Tool Team | Before follow-up with Henrik | Verify consent checkboxes are pre-filled and can be updated |
| Hide CRM ID from form editor | Form Tool Team | Implementation | Hide CRM ID field if integration is enabled on section |
| Follow-up meeting with Henrik | Erik Andersson, Lukasz Grabowski | Next week (Monday) after Henrik returns from vacation | Present findings and product decisions needed |
| Investigate form tool profile resolution | Form Tool Team | Ongoing | Document exactly how form tool finds profiles and performs merges across key spaces |
| Document form vs. event tool differences | Form Tool Team & Integration | Ongoing | Clarify differences in profile lookup and merge between these two tools |