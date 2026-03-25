---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One Integrations
topics:
  - Form submission and CRM synchronization flow
  - Pre-filled form functionality
  - Profile resolution and duplicate prevention
  - CRM master data protection
  - Attribute mapping and enrichment
  - Consent synchronization
  - Key space management and merging
speakers:
  - Erik Andersson (Integration Architecture/Lead)
  - Lukasz Grabowski (Product/Form Tools)
  - Michal Rosikiewicz (Product)
  - Tomasz Kowalski (QA/Testing)
key_components:
  - Audience service
  - Audience subscription worker
  - Form event worker
  - Outbound worker
  - Merge worker
  - Delta Sync worker
  - Mappings manager service
  - Form tool
  - CRM systems (Efficy Enterprise 12.0, 12.1)
session_type: architecture-review
subdomains:
  - Architecture
  - Outbound Flow
  - Efficy Enterprise 12.0
  - Efficy Enterprise 12.1
  - Lead creation
  - Duplicate profiles in Apsis
---

## Session Overview

This session combines knowledge from integration architecture and form tools teams to analyze requirements for pre-filled form functionality with CRM synchronization. The discussion covers how form submissions flow through the integration platform, profile resolution when forms are sent to existing CRM-synced profiles, and critical product decisions around protecting CRM master data from being overwritten by form inputs. The team identifies several risks around duplicate profile creation and CRM ID manipulation that require careful handling in the form tool implementation.

---

## Current Form Submission and CRM Sync Flow

### How Forms Today Sync with CRM Systems

When a form is created in a section with an enabled CRM integration that supports form syncing (such as Efficy Enterprise), the system offers a "CRM sync" option. When this option is selected, the form submission triggers the integration flow:

> When you select this sync, the form activity to whatever CRM you have, the tool in question will fire a request to us saying the user would like to sync this activity with this ID to this CRM system.

[Erik Andersson]: The integration platform registers event listeners based on the form type:
- For standard form tools: `open`, `submit`, `started`, `viewed` events
- For event tools: `attended`, `cancelled`, and other specialized events as specified in the architecture

### Submission Event Processing Pipeline

[Erik Andersson]: The processing follows this sequence:

1. **Audience Subscription Worker**: Receives the form submission event from the Audience service. This is the first step where the event enters the integration system.

2. **Event Conversion and Profile Field Extraction**: The worker converts the event to the integration format and extracts critical profile fields:
   - CRM ID (if available)
   - Email address
   - Phone number
   
   [Erik Andersson]: "If there is no CRM ID then we require there to be like either an SMS or an e-mail. Because that's the secondary like type of identification."
   
   If neither CRM ID nor email/SMS are present, the submission cannot be processed because there's no way to identify which profile to attach it to. First name and last name alone are insufficient for identification.

3. **Metadata Addition**: The worker adds required fields:
   - Activity ID
   - Event timestamp
   - Event type
   - Profile key

[Lukasz Grabowski raises an important point]: The profile key comes from Audience. For public forms with no existing profile, the key space depends on the data provided: email address goes into the email key space, phone number into the SMS key space.

4. **Kafka Queue**: The verified message is sent to Kafka (though there are no visible logs in Kafka itself).

5. **Form Events Worker**: Batches all form events received in the past time window and assigns them a batch ID.

6. **Outbound Worker**: Sends the batch to the CRM system via HTTP request.

### CRM System Response Handling

[Erik Andersson]: The CRM system has three possible response outcomes:

**Outcome 1 - Acknowledgment Only**: The CRM responds with an essentially empty response, indicating acceptance but taking no action. Nothing happens in the CRM.

**Outcome 2 - Profile Match Found**: The CRM matches the email address (or other identifier) to an existing contact and responds with the CRM ID for that contact. This is the key scenario for profile merging:

> When you submit the form, you have created a new profile in audience with the submit event. The CRM system will respond with yes, this e-mail address matches this existing contact in the CRM. We will then take this CRM ID and add that as a attribute to this form profile and then we do a merge between the contact that was created from the form and the and our bootstrap integration key space.

The merge process ensures that the profile created by the form submission is unified with the CRM's key space (e.g., "FSC Enterprise 12.1 key space" where CRM ID is the key).

**Outcome 3 - New Contact Created**: The CRM creates a new contact for the profile and responds with the new CRM ID. The merge process is identical to Outcome 2.

### Why Merging is Critical

[Erik Andersson]: The merge operation is essential for preventing duplicates in future real-time syncs:

> If you have if you would have just the profile submitted from the form with the e-mail and we had added the CRM ID, that profile would not have been found when we do the profile updates in the real time sync because we only look in our custom key space.

Each CRM integration has its own bootstrapped key space (e.g., "FSC Enterprise 12.1 key space", "Dynamics key space", "E-deal key space"). Real-time Delta Sync updates from the CRM only look in this custom key space to find profiles. If a profile isn't merged into this key space after a form submission, subsequent updates from the CRM will create duplicates.

### Critical Constraint: No Attribute Updates from Apsis

[Erik Andersson]: **This is foundational**: When a form is submitted, Apsis never updates attributes in the CRM system, regardless of what data the user entered in the form:

> We just let the CRM system know that this submit happened like we are not updating any profile like if they entered like a first name or last name. Apsis is never making any like actual attribute updates in the CRM system like we are not updating first names or last name or adding emails.

[Lukasz Grabowski clarifies]: This aligns with the fundamental rule that the CRM is the master. Outbound worker is consent-only—it notifies the CRM of events and updates consent where mapped, but does not modify contact attributes.

The CRM may create an internal entry or event record linked to the contact, but it does not update the contact's attributes based on form submissions.

---

## Pre-Filled Form Functionality and Profile Resolution

### What Pre-Filled Forms Do

[Lukasz Grabowski]: Pre-filled forms allow sending a form link to a specific profile in Apsis (currently behind a feature flag). When a recipient opens the form:

> Through this link, yeah, we know that this is this is my profile. Yeah, so this is my data. So this is pre-filled with e-mail and last name and other other attributes.

The form is populated with the profile's existing attribute values, providing a smoother user experience.

### The Critical Technical Challenge: Profile Resolution

[Erik Andersson identifies the core problem]: When a pre-filled form is submitted by a profile that was synced from CRM, there is ambiguity in which key space should be used for the submission event:

> Now the first important question is: How is this handled in the form service? Which which profile are you adding this to? How are you finding that profile?

The specific scenario that requires investigation:

1. A profile is synced from CRM (FSC Enterprise 12.1) into Apsis, existing only in the CRM's key space
2. The profile has a CRM ID but may not exist in the email key space
3. A pre-filled form link is sent to this profile and they submit it with their email address
4. **Unknown**: Does the form service find the profile via:
   - The CRM key space (using CRM ID)?
   - The email key space (creating a new profile)?
   - Both (merging them)?

[Lukasz Grabowski describes the test case needed]:

> So we have the same data, which it's not a good approach. If we have the case that we sync it from the CRM to Apsis where the e-mail will be mine. And then I send a form, I submit a form. The result should be here that still we will have one profile, right?

[Erik Andersson's expectation]:

> If you search for the e-mail and go to the profile, it should have the same ID as if you search for the CRM ID and go to the profile. If that is the case, everything is good. But if you search on the e-mail, you get one profile ID and then you search for the CRM ID and you get another ID and only the one with the e-mail has the submit event. Then we need to go back to the drawing board with that flow.

### The Duplicate Profile Risk

[Erik Andersson explains the danger]:

> This is probably the most important for my part because this is the technical part like how how is our key space handled? Which key space are you merging with? How if you don't, if we have not submitted a form before, will form key space or will the event be attached to the profile that is only in ours or will you create one new in the e-mail key space and then not do anything more? Because if you only create one in the e-mail key space and this is not merged with ours, then we have suddenly created a duplicate in Apsis.

If the form tool creates a new profile in the email key space without merging it with the CRM key space, the system ends up with:
- Profile A: CRM key space only, with CRM ID and form submission data
- Profile B: Email key space only, with email and form submission data
- These are the same person but treated as two separate profiles

This violates a core invariant: one person = one profile in Apsis.

---

## Product Decisions: Protecting CRM Master Data

### The Core Tension

Forms allow users to submit data (first name, last name, email, custom attributes, etc.). The integration system has a rule: **the CRM is the master source of truth**. This creates a product decision:

**Should form-submitted data be allowed to overwrite attributes that came from the CRM?**

### Why This Matters: Data Discrepancies

[Erik Andersson illustrates the problem]:

> Let's say that you sync the profile from the CRM to Apsis like I have. My first name is Eric, my last name is Frederick. I submit the form and now in the form I specify like yeah, my name is not Eric, it is like Henrik. If you accept the overwriting of this data, now suddenly in the CRM system my name will be Eric, but in Apsis this data will be Henrik.

When an email is sent to this profile, it will display "Henrik" (the form-submitted value from Apsis), but the contact in the CRM still has "Eric" (the master data). This introduces a discrepancy.

When the next Delta Sync update arrives from the CRM, it will overwrite the form-submitted "Henrik" back to "Eric" because Apsis does not support syncing attribute updates back to the CRM.

[Erik Andersson]: 

> What ever you change manually or in the form submissions like we will completely disregard and overwrite and that's why I I have always been of the opinion that like any data that you have set up the mapping for you should not be able to modify in Apsis because if you are to modify that data it should be modified in the CRM system because then it will be updated in Apsis as a consequence because of the the syncs.

### Erik's Recommended Approach

[Erik Andersson]: The integration side could support this by checking field mappings:

> The one I would suggest personally if I had done this, I would have looked at which mapping, which field mapping exists on this integration... then you use that as the integration ID parameter. Then you will get like the mappings we have and in the mappings object there will be a Apsis 1 field which contains the ID and all attributes that exists in those mappings, you should not be able to overwrite that data with data from the form because like all of that data will come from the CRM system.

Process:
1. Call mappings manager service with account, section, and integration ID
2. Retrieve the list of mapped attributes
3. Prevent form submissions from overwriting any attribute that appears in the mappings
4. Allow form submissions to create/update unmapped attributes (enrichment)

### Distinction: CRM-Mapped vs. Enrichment Attributes

[Erik Andersson]: There are two types of attributes:

**1. CRM-Mapped Attributes**: These are synced from the CRM (via field mappings). Examples: first name, last name, email, address. These should never be overwritten by form submissions.

> Any data that you have set up the mapping for you should not be able to modify in Apsis... if you are to modify that data it should be modified in the CRM system.

**2. Enrichment Attributes**: These are new attributes created in Apsis, not synced from the CRM. Example: "customer satisfaction rating", "preferred color". These can and should be created from form submissions.

[Lukasz Grabowski]: 

> So if there is new attributes so we we just not touch, don't touch it and that's it. I mean from the CRM perspective but of course it will be updated on the profile.

Critical caveat: These enrichment attributes will **never** be synced back to the CRM. They exist only in Apsis.

[Erik Andersson]:

> This data will not exist in the CRM system. It will only exist in APSIS because we don't support the sinking of attribute changes in Apsis to the CRM, just so Henrik is also aware of this and not like, yeah, yeah, we collect all cool data in Apsis and it is automatically added in the CRM.

### Current Form Behavior (Unclear)

[Erik Andersson]: The current form tool has an "update existing profile data" toggle, but its exact behavior is not fully understood:

> It it it completely depends on what you selected in the form. If you select selected that like don't override data then like you want to modify the attributes. If you select allow override data then anything can be updated.

The team notes that this needs investigation to understand the default behavior and which fields are actually protected.

### The Absolute Red Line: CRM ID Must Never Be Writable

[Erik Andersson raises a critical concern]:

> And there is, there is one absolute big no, no, which I think you might actually be able to do. Let's say now that you add a field and I don't know if you are able to add the mapping to CRM ID. Like let's say that you create a field called CRM ID and you map that to a the CRM ID field inside of Apsis if you for whatever reason would would be able to actually update your CRM ID. I can tell you that we will be in deep **** because this is one of the things you can under no circumstance do if you have an integration because we will lose complete track of that profile. Then you will definitely introduce duplicates.

If a form allows a user to change the CRM ID of a profile, the integration system loses its ability to track that profile across syncs, resulting in immediate and severe data corruption.

[Michal Rosikiewicz]: So we should hide CRM ID from the form editor.

[Erik Andersson confirms]: Yes, if you have an integration, you can never be allowed to do this.

### Proposed Solution: Hide CRM ID When Integration is Active

The team agrees that when a section has an active CRM integration, the CRM ID field should be hidden from the form editor entirely. 

[Lukasz Grabowski]: 

> So we should hide CRM ID from editor. I would say so hide CRM ID from editor if you have integration on a section.

This prevents accidental or intentional modification of the profile's key.

---

## Consent Synchronization

### Bidirectional Sync Requirement for Consent

[Erik Andersson]: Consent is fundamentally different from other attributes because it's bidirectional for legal reasons:

> Here there are legal reasons as to why we need to have a bidirectional sync. So like this, this should already work out-of-the-box.

When a user updates their consent in a pre-filled form (e.g., checking "I consent to marketing"), the system must send that consent change back to the CRM.

### Current Gap in Pre-Filled Forms

[Lukasz Grabowski observes]: Pre-filled forms currently don't show consent checkboxes:

> For prefil, it doesn't work right now because there is no checkbox here so we should have it filled somehow.

[Erik Andersson]: Once the form tool is fixed to display and accept consent changes, the integration side will automatically handle sending the updates:

> If an end user modifies their consent... and they say yes, I do want consent on this specific subscription that we will send to the CRM system if you have a mapping for it.

However, this only works for subscriptions that have consent mappings defined in the integration:

> If you add it for a subscription B, which does not exist in our consent mapping, we will not react on that because we don't have a listener registered for it.

---

## Key Acceptance Criteria for Pre-Filled Forms

Based on the discussion, the following acceptance criteria need to be validated:

### 1. Profile Data Pre-Population
- When a recipient opens a pre-filled form link, their profile data is pre-populated in the form fields
- Email field is locked (cannot be changed)
- **Status**: Confirmed working in demo

### 2. Email as Primary Identifier
- Email is treated as a locked primary identifier to prevent duplicate profile creation
- Only one profile should exist regardless of whether it's found via CRM ID or email key space

### 3. CRM ID Protection
- **CRM ID field must be hidden from form editor when integration is enabled on the section**
- Users should never be able to modify the CRM ID
- **Current status**: CRM ID can currently be mapped as a form field—this is a bug

### 4. Prevent Attribute Overwrites of CRM-Synced Data
- CRM source attributes (those with field mappings) are not overwritten by form submissions
- Only unmapped (enrichment) attributes should be updatable via forms
- **Current status**: Behavior is configurable but unclear whether it properly protects mapped fields

### 5. Ensure No Duplicate Profiles on Form Submission
- When a profile exists only in the CRM key space and a form is submitted via email, the profile must be merged into both key spaces
- If a profile is found via email, any existing CRM ID must be linked
- **Current status**: To be investigated—this is the critical test case

### 6. Enrichment of Non-CRM Attributes
- Attributes not present in the CRM field mappings can be added or updated via forms
- These enrichment attributes are stored only in Apsis and not synced back to CRM
- **Status**: Confirmed as design

### 7. Consent Changes Propagate to CRM
- When a user updates their consent status in a pre-filled form, the change is sent to the CRM
- Only subscriptions with consent mappings are synced
- **Current status**: Form tool needs to display consent checkboxes; integration side should auto-sync once fixed

### 8. Audit Trail (Unclear Requirement)
- The acceptance criteria mention "Audit log shows which fields came from CRM, what changes were suggested and what was stored locally"
- **Status**: Unclear what this means or how audit logs would be stored. Requires clarification with product.

---

## Key Technical Risks and Unknowns

### Risk 1: Duplicate Profile Creation (HIGH PRIORITY)

**Scenario**: Profile synced from CRM (only in CRM key space) receives pre-filled form, submits with email address.

**Unknown**: Will form tool create a new profile in email key space without merging with CRM key space?

**Impact**: Results in duplicate profiles for the same person.

**Mitigation**: Requires investigation and test case validation before form tool implementation.

[Tomasz Kowalski] is assigned to test this scenario.

### Risk 2: CRM ID as Form Field

**Current state**: CRM ID can apparently be added as a form field and mapped to the CRM ID attribute.

**Risk**: Users could accidentally or intentionally change CRM IDs, causing profile tracking to break.

**Resolution**: Hide CRM ID from form editor when integration is enabled.

### Risk 3: Data Discrepancies from Form Overwrites

**Current state**: Form tool allows "update existing profile data" toggle, unclear which fields are protected.

**Risk**: Mapped attributes overwritten by form submissions will be re-overwritten by next Delta Sync, creating confusion.

**Resolution**: Either:
- Option A: Prevent all overwrites of mapped attributes
- Option B: Prevent overwrites only if profile has CRM ID (but requires form tool to know mappings)
- Option C: Check mappings and prevent overwrites of mapped fields only (Erik's recommendation)

**Status**: Requires product decision and form tool implementation.

### Risk 4: Consent Checkbox Missing in Pre-Filled Forms

**Current state**: Consent subscriptions are not displayed in forms.

**Impact**: Users cannot update their consent via pre-filled forms.

**Resolution**: Form tool must display consent checkboxes for subscriptions. Integration will automatically sync changes.

---

## Recommended Implementation Path

### Phase 1: Protect Against Critical Bugs
1. **Hide CRM ID field from form editor when integration is enabled** — prevent accidental modification
2. **Validate duplicate profile prevention** — test case: sync profile from CRM, send pre-filled form with email, verify single profile exists

### Phase 2: Clarify Data Protection Strategy
1. Determine product decision on allowing overwrites of mapped attributes
2. If allowing selective overwrites, implement check against mappings manager service
3. Document behavior clearly

### Phase 3: Enable Consent Sync
1. Add consent checkboxes to pre-filled forms
2. Ensure consent changes trigger outbound sync to CRM

### Phase 4: Add Enrichment Support
1. Document which attributes can be added/modified (unmapped ones only)
2. Test that enrichment attributes are stored in Apsis and not sent to CRM

---

## Call Chain and Services Involved

```
Audience (form submission event)
  ↓
Audience Subscription Worker (validates, extracts profile fields, adds metadata)
  ↓
Kafka Queue
  ↓
Form Events Worker (batches events)
  ↓
Outbound Worker (sends batch to CRM)
  ↓
CRM Response
  ↓
Merge Worker (if CRM ID response received)
  ↓
Delta Sync Worker (real-time CRM updates use CRM key space to find profiles)
```

For field mapping checks:
```
Form Tool
  ↓
Mappings Manager Service (on section/account/integration_id)
  ↓
Returns mapped attribute list
```

---

## Key Takeaways

1. **CRM is the master**: All attribute values for CRM-synced fields originate in the CRM. Form submissions that overwrite these will be overwritten back on the next sync, creating confusion.

2. **Key space merging is critical**: Profiles synced from CRM must be merged into the CRM's custom key space (e.g., "FSC Enterprise 12.1 key space") to prevent duplicates in Delta Sync.

3. **Email is not enough**: Email address alone cannot reliably identify a profile that only exists in the CRM key space. The form tool must handle merging multiple key spaces when appropriate.

4. **Consent is bidirectional**: Unlike regular attributes, consent changes must propagate back to the CRM for legal compliance.

5. **CRM ID is untouchable**: Under no circumstances should users be able to modify a profile's CRM ID. This must be hidden from forms when integration is active.

6. **Enrichment attributes can be new**: New attributes not mapped to the CRM can be created and updated via forms. These will never be synced to the CRM.

7. **Three possible outcomes**: When a form is submitted to a CRM with an email match, the CRM can either:
   - Acknowledge but do nothing (empty response)
   - Match to existing contact and return CRM ID
   - Create new contact and return CRM ID
   - All three paths lead to a merge operation in Apsis

8. **Form tool needs mappings awareness**: To implement safe overwrites, the form tool needs to call the mappings manager service to determine which attributes should be protected.

---

## Unresolved Questions and Action Items

### Critical for Implementation
1. **[Tomasz Kowalski]**: Validate that pre-filled form submissions to CRM-synced profiles do not create duplicates. Specific test case:
   - Sync profile from FSC Enterprise 12.1 with CRM ID
   - Send pre-filled form link
   - Submit form with email address
   - Verify single unified profile exists (searchable by both CRM ID and email, returning same profile ID)

2. **[Form Tool Team]**: Determine current behavior of "update existing profile data" toggle in forms. Which fields are protected? Which can be overwritten?

3. **[Product Decision]**: Define policy on overwrites:
   - Option A: Never allow overwrites of any attributes on profiles with CRM ID
   - Option B: Allow overwrites only of unmapped (enrichment) attributes
   - Option C: Require form tool to check mappings and prevent overwrites selectively

4. **[Form Tool Team]**: Implement hiding of CRM ID field from editor when integration is enabled on section

5. **[Form Tool Team]**: Add consent checkboxes to pre-filled forms

### Lower Priority / Clarification
6. **[Product]**: Clarify audit log requirement in acceptance criteria—what should be tracked, where, and by whom?

7. **[Integration Team / Product]**: If customers need to send profile enrichment data from Apsis back to CRM, this is a new use case requiring roadmap discussion (currently unsupported)

### Follow-Up Meeting
- **When**: Next week, Monday (Henrik returns from vacation)
- **Who**: Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Product (Henrik)
- **Purpose**: Review findings, validate duplicate profile scenario, make product decisions on overwrites and CRM ID handling

---

## Technical Notes for Developers

### Key Space Naming
- Email key space: created automatically when profile has email
- SMS key space: created automatically when profile has phone
- CRM custom key spaces: one per integration (e.g., "FSC Enterprise 12.1 key space", "Dynamics key space", "E-deal key space")
- Form key space: (existence/usage unclear from discussion)

### Profile Lookup Logic
[Lukasz Grabowski]: Profile discovery depends on form context. For pre-filled forms coming from a specific profile ID, the key space must be determined. The audience service is called to find profiles, and the key space is "processed through a couple of services" (not detailed in this session).

### Outbound Worker Behavior (Consent-Only)
The outbound worker:
- ✅ Sends events to CRM (form submissions, consent changes)
- ✅ Updates consent on profiles in CRM (if mapped)
- ❌ Never updates attributes on existing contacts
- ❌ Never modifies non-consent fields based on form input

---

**Session ended**: Discussion concluded with follow-up meeting scheduled for next week with product stakeholder (Henrik) to validate findings and make final product decisions on duplicate prevention and data protection strategies.