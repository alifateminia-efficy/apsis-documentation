---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One Integrations
topics: [CRM form syncing, pre-filled forms, profile key spaces, profile merging, duplicate profile prevention, attribute overwrite protection, consent bidirectional sync, CRM master data principle]
speakers: ["Erik Andersson (Integration)", "Lukasz Grabowski (Forms)", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [Audience Subscription Worker, Outbound Worker, Merge Worker, Kafka, FSC Enterprise 12.1, Mappings Manager Service, CRM key space, Email key space, Form tool]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Duplicate profiles"]
---

## Session Overview

This session is a joint knowledge-transfer between the Integration team (Erik) and the Forms team (Lukasz), focused on understanding how the existing CRM integration handles form submission events, and what challenges arise when combining that with the new pre-filled forms feature. The session walks through the live flow of a form submission through the integration pipeline, then live-tests a pre-filled form sent to a CRM-synced profile. Key concerns discussed include: correct profile resolution to avoid duplicate profiles, protection of CRM-sourced attributes from being overwritten, the CRM-as-master-data principle, and the limitations of the current outbound sync (Apsis cannot push attribute updates to the CRM). Several open questions and action items were identified for a follow-up meeting with Henrik (product).

---

## CRM Form Sync: How It Works Today

### Enabling CRM Sync on a Form

When a form is created on a section that has a CRM system installed (e.g., FSC Enterprise 12.1), there is a **CRM sync option** in the form settings. Selecting this option tells the integration to listen to events from that form activity.

The integration registers listeners for the relevant events depending on the tool type:
- **Form tool**: listens to `opened`, `submitted`, `started`, `viewed`
- **Event tool**: listens to `registered`, `attended`, `cancelled`, etc.

### Event Flow: From Submission to Outbound Worker

When a public visitor submits a synced form, the following pipeline executes:

1. **Audience** receives the submission and sends the event to the **Audience Subscription Worker queue**.
2. The **Audience Subscription Worker** picks up the event and converts it into the integration's required format. It extracts:
   - `account` and `section` + `integration ID`
   - Profile fields: CRM ID (if present), email address, phone number
   - Activity ID, event timestamp, event type, profile key

   > "If there is no CRM ID then we require there to be either an SMS or an email. If there's no CRM ID, no email address, and no SMS, then this submission is useless — we can't attach it to any existing resource. First name or last name alone is not enough."

3. The processed message is sent to **Kafka**.
4. A **worker listening to form events** batches the message and sends it to the **Outbound Worker**.
5. The **Outbound Worker** sends the batch to the CRM system.

### What the Outbound Worker Actually Sends

**Critical point:** Apsis does NOT push attribute updates to the CRM. The outbound worker only notifies the CRM that a form submission event occurred.

> "We are just letting them know: 'yo, someone submitted this form — do what you want with it.' We are not updating first names, last names, or adding email addresses."

### Profile Key Space for Public Form Submissions

For public forms (no pre-existing profile), Audience creates a new profile before notifying integration:
- If the submitter provided an email → profile is created in the **email key space**
- If the submitter provided a phone number → profile is created in the **SMS key space**

---

## CRM Response Handling and the Merge Worker

After the Outbound Worker sends the form submission event to the CRM, the CRM can respond in one of **three ways**:

### Outcome 1: CRM Does Nothing
The CRM responds with an essentially empty response (no matched record, no new record). Integration takes no further action.

### Outcome 2: CRM Matches an Existing Contact
The CRM finds an existing contact matching the submitted email or phone number and responds with the corresponding **CRM ID** for that contact.

Integration then:
1. Adds the CRM ID as an attribute to the newly-created form profile in Apsis.
2. Triggers the **Merge Worker** to merge the form profile (in the email key space) with the corresponding profile in the **integration's custom CRM key space** (e.g., the FSC Enterprise key space where the CRM ID is the key).

> "In the future, if the CRM system sends us a real-time update in the Delta Sync worker flow, we don't create a duplicate in Apsis — because if you only had the form-submitted profile with the email and we had not added the CRM ID, that profile would not have been found when we do profile updates in the real-time sync, because we only look in our custom key space."

### Outcome 3: CRM Creates a New Contact
The CRM does not find a matching contact, so it creates a new one and responds with the new CRM ID. Integration flow is identical to Outcome 2: add CRM ID to the Apsis profile and trigger a merge.

### The CRM Does Not Update Attributes
When the CRM receives the form submission event, it typically creates an internal entry linking the event to the contact. It does **not** update contact attributes based on form field values. The CRM-as-master principle means data flows CRM → Apsis, not the reverse.

---

## Integration Key Spaces: Architecture and Importance

### Custom Key Space per Integration
Every integration installation bootstraps its own custom key space in Apsis:
- FSC Enterprise 12.1 → `FSC Enterprise 12.1` key space, where the **CRM ID is the key**
- Dynamics → Dynamics key space
- E-Deal → E-Deal key space

This is distinct from the email key space and the SMS key space.

### Why This Matters for Profile Lookups
When the integration's real-time sync (Delta Sync worker) processes an update from the CRM, it looks up profiles using the **custom CRM key space**, not the email or SMS key space. If a profile only exists in the email key space without a CRM ID attached and merged into the CRM key space, the real-time sync will not find it and will create a duplicate.

---

## Pre-Filled Forms and the Integration: Key Pitfalls

### What Pre-Filled Forms Are
A pre-filled form is sent to a **known profile** via a link (e.g., embedded in an email campaign). The form is pre-populated with the profile's existing attribute values. This is a new feature, currently behind a feature flag.

### Live Test Observations
The session included a live test: an email was sent to profiles from the CRM-synced list (FSC Enterprise 12.1), containing a link to a form with CRM sync enabled. A profile was opened that:
- Had a CRM ID
- Was in the FSC Enterprise 2 key space (the custom integration key space)
- Did NOT have a profile in the email key space (no email key space entry found on initial lookup)

After form submission, the profile was found via the identifier `179` → confirmed as `FSC Enterprise 2` key space. The merge had occurred, connecting the form submission to the correct profile.

**Observation:** First name was not updated on the profile after submission, possibly due to event deduplication logic in the forms service. Flagged for investigation.

### Critical Open Question: How Does Form Tool Find the Integration Key Space?

[Erik Andersson]: When a pre-filled form is submitted for a profile that only exists in the CRM custom key space (never had an email key space entry), how does the form tool find that profile?

- Is it using the profile export key?
- Is it using the email key space via the email address?
- Or is it somehow resolving via the integration key space?

> "If you only create one [profile] in the email key space and this is not merged with ours, then we have suddenly created a duplicate in Apsis."

This must be verified offline with a dedicated test: create a contact in the CRM with a test email, sync it to Apsis, send a form to it, and verify that only one profile exists in Apsis afterward (same profile ID whether looked up by email or by CRM ID).

**Action item:** Tomasz Kowalski to investigate and test this scenario. Erik offered to write up the exact steps.

---

## Duplicate Profile Risk with Pre-Filled Forms

This is identified as the most critical technical risk of the pre-filled form + integration combination.

### The Scenario That Creates Duplicates

1. CRM syncs a contact to Apsis → profile exists only in the **CRM custom key space**
2. An email is sent to that profile containing a form link
3. The recipient submits the form
4. If the form tool creates a new profile in the **email key space** without merging it into the CRM key space → **two separate profiles now exist** for the same person

### Why This Is Bad
- Real-time sync from the CRM will update the CRM key space profile, not the email key space profile
- The submit event is only on the email key space profile
- Data diverges; the person appears twice in Apsis

### The CRM ID Field: An Absolute No-No

[Erik Andersson]:
> "If for whatever reason you would be able to actually update the CRM ID [via a form field], I can tell you that we will be in deep shit. This is one of the things you can under no circumstance do if you have an integration — we will lose complete track of that profile. You will definitely introduce duplicates."

**Recommendation:** If an integration is installed on the section, the CRM ID field **must be hidden from the form editor**. Currently it is not hidden, which is a bug/gap.

[Michal Rosikiewicz]: The CRM ID field should only be hidden if an integration is installed on the section; if no integration is present, it can be used freely.

---

## Attribute Overwrite Protection: CRM Master Data Principle

### The Core Rule
Attributes that are mapped in the CRM integration should **not** be overwritable via form submission in Apsis, because:
1. Any change made in Apsis will be overwritten on the next CRM sync (CRM is always master)
2. It introduces a temporary discrepancy between what is shown in Apsis and what exists in the CRM

[Erik Andersson]:
> "I have always been of the opinion that any data you have set up the mapping for, you should not be able to modify in Apsis. If you want to modify that data, it should be modified in the CRM system — then it will be updated in Apsis as a consequence of the sync."

### How to Determine Which Fields Are CRM-Mapped
The **Mappings Manager Service** can be queried with the account, section, and integration ID to retrieve all field mappings. The `apsis_one_field` key in the mappings object identifies mapped attributes. These fields should be locked/read-only in the form editor when an integration is present.

### Allowed: Enrichment Attributes (Non-CRM Fields)
New attributes that are **not** in the CRM field mappings (e.g., `satisfactory_rating`, custom marketing attributes) **can and should** be writable via form. Integration will not touch them during syncs because there is no mapping for them.

**Important disclaimer:** These enrichment attributes will only exist in Apsis. They will **not** be pushed to the CRM system. Apsis today does not support syncing attribute changes from Apsis to the CRM.

### Current Behavior
Whether attributes are overwritten depends on the **"Update existing profile data"** toggle in the form settings. There is no current logic to distinguish between CRM-mapped fields (which should be protected) and custom enrichment fields (which should be writable).

**Proposed solution (Michal):** Add a third option: "Block update if profile has CRM ID." However, Erik noted the nuance — the block should only apply to CRM-mapped fields, not all fields.

---

## Consent: The Exception to the No-Outbound-Update Rule

Consent is **bidirectional** between Apsis and the CRM, unlike attribute data.

[Erik Andersson]:
> "There are legal reasons as to why we need to have a bidirectional sync [for consent]."

If a recipient submits a form and opts into a subscription that has a **consent mapping** configured in the integration, the outbound worker will send that consent change to the CRM.

- If subscription A is in the consent mapping → consent change is synced to CRM
- If subscription B is not in the consent mapping → integration does not react; no listener registered

**Current gap:** Pre-filled forms do not currently pre-fill consent/subscription checkboxes. This needs to be fixed in the form tool. Once fixed, the downstream consent sync to CRM should work automatically out-of-the-box via existing integration behavior.

---

## Requirement: Update Request Sent to CRM for Enrichment Attributes

One of Henrik's acceptance criteria states: "An update request is sent to CRM for [enrichment] attributes."

[Erik Andersson]:
> "This will never happen. We are never sending an update request to the CRM."

This requirement as written is **not achievable with current integration architecture**. Sending attribute updates from Apsis to the CRM is a completely new use case that has never been implemented. If this is a hard requirement, it needs to be raised with both the product team and the CRM-side development teams as new work.

---

## Acceptance Criteria Review (Henrik's Requirements)

| Criteria | Status | Notes |
|---|---|---|
| Recipient opens form link and profile data is pre-filled | ✅ Works | Pre-fill functionality demonstrated live |
| Email field is locked (read-only) | ✅ Works | Existing behavior |
| CRM source attributes that differ from input are not overwritten in Apsis | ⚠️ Partially | Depends on form "update existing data" toggle; no CRM-aware distinction currently |
| CRM ID not available/editable in form editor | ❌ Gap | Currently editable; must be hidden when integration is on the section |
| Update request sent to CRM for enrichment attributes | ❌ Not supported | Apsis does not send attribute updates to CRM; new development required |
| Non-CRM attributes stored in Apsis | ✅ Works | Standard behavior; CRM sync will not overwrite unmapped fields |
| Form submissions do not create duplicate profiles | ❓ Unknown | Requires investigation/testing (assigned to Tomasz) |
| Consent can be shown and updated | ⚠️ Partial | Consent sync to CRM works; pre-fill of consent checkboxes not yet implemented in form tool |
| Audit log showing field sources and changes | ❓ Unknown | No clear owner or mechanism identified; source tracking for attributes (unlike consent) not currently supported |

---

## Key Takeaways

1. **CRM is always master.** Apsis integration is strictly one-way for attribute data: CRM → Apsis. Apsis will never push attribute changes to the CRM. Any form-submitted data that conflicts with CRM data will be overwritten on the next sync.

2. **The CRM custom key space is the integration's backbone.** Every installation gets a dedicated key space where the CRM ID is the lookup key. Merging form-submitted profiles into this key space is what prevents duplicates and ensures real-time CRM updates reach the right profile.

3. **Pre-filled forms + integration = high duplicate risk.** If a profile only exists in the CRM key space (never submitted a form before) and submits a pre-filled form, there is a real risk of creating a second profile in the email key space. This is the top open question to resolve.

4. **CRM ID must be hidden from the form editor** when an integration is installed on the section. Allowing the CRM ID to be edited via a form field would cause immediate and unrecoverable profile tracking failures.

5. **Only enrichment attributes (non-CRM-mapped fields) should be writable** via form for profiles with a CRM ID. CRM-mapped fields should be read-only. The Mappings Manager Service can be used to determine which fields are mapped.

6. **Consent is the one exception** to the no-outbound-update rule — it is bidirectional for legal reasons, and changes made via form will be synced to the CRM if a consent mapping exists.

7. **Sending attribute updates from Apsis to CRM is not supported today** and represents significant new development if required.

---

## Unresolved Questions and Action Items

1. **[Tomasz Kowalski — Action]** Test the duplicate profile scenario: create a CRM contact with a personal test email, sync to Apsis, send a pre-filled form to that profile, submit it, and verify that only one profile exists in Apsis (same profile ID via email lookup and CRM ID lookup). Erik to provide exact reproduction steps.

2. **[Forms team — Investigate]** How does the form tool currently resolve which profile to attach the submit event to when using a pre-filled link? Does it use the profile export key, the email key space, or another mechanism? This is critical for understanding whether merges with the CRM key space happen correctly.

3. **[Forms team — Investigate]** Why was first name not updated on the profile during the live test submission? Possibly related to event deduplication logic in the forms service.

4. **[Product discussion — Henrik meeting]** Clarify requirement: "Update request sent to CRM for enrichment attributes." This is not currently supported. Is this a hard requirement? If so, it is new development for both integration and the CRM connectors.

5. **[Product discussion — Henrik meeting]** Confirm the intended behavior for CRM-mapped attribute protection in form submissions: should CRM-mapped fields be completely read-only, or only protected from overwrite (pre-filled but editable)?

6. **[Forms team — Implementation]** Hide CRM ID field from form editor when an integration is installed on the section.

7. **[Forms team — Implementation]** Pre-fill consent/subscription checkboxes in pre-filled forms (currently not working).

8. **[Open]** Audit log requirement: no current mechanism exists to track which fields came from CRM vs. were submitted via form. Ownership and feasibility unclear.

9. **[Erik + Lukasz — Offline test]** Create a profile in FSC Enterprise with a test email (e.g., `email+test@domain.com`), sync it, send a pre-filled form, and verify the integration key space behavior end-to-end.