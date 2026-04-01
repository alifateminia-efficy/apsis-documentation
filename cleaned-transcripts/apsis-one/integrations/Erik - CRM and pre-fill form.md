---
source_file: Erik - CRM and pre-fill form.txt
domain: Apsis One Integrations
topics: [CRM form sync, pre-filled forms, profile key spaces, profile merging, duplicate prevention, CRM master data protection, consent sync, attribute enrichment, outbound worker flow]
speakers: ["Erik Andersson (Integration Engineer)", "Lukasz Grabowski (Form Tools Engineer)", "Michal Rosikiewicz", "Tomasz Kowalski"]
key_components: [Audience Subscription Worker, Outbound Worker, Kafka, Mappings Manager Service, FSC Enterprise key space, Delta Sync Worker, Merge Worker, Form Tool]
session_type: knowledge-transfer
---

## Session Overview

This session is a joint knowledge-transfer between the Integration team (Erik) and the Form Tools team (Lukasz, Michal, Tomasz) to analyze requirements defined by Henrik for CRM-integrated pre-filled forms. Erik walks through how the existing CRM form sync integration works end-to-end — from event capture through to CRM notification and profile merging. Lukasz then demonstrates the pre-fill form feature (currently behind a feature flag). The session identifies several critical technical risks and open product decisions around profile key space resolution, duplicate prevention, CRM master data protection, and attribute enrichment. The session concludes with a list of acceptance criteria from Henrik's requirements, reviewed against actual system behavior.

---

## Existing CRM Form Sync: End-to-End Flow

### Enabling CRM Sync on a Form

When a form is created on a section that has a CRM system installed (e.g., FSC Enterprise 12.1), the form settings expose a **CRM sync option**. When enabled, the tool fires a request to Integration to register event listeners for that activity.

- For a **form tool**: listeners are registered for `opened`, `submitted`, `started`, and `viewed` events.
- For a **form event tool**: all event types are registered (e.g., `registered`, `attended`, `cancelled`, etc.), consistent with the Apsis One architecture specification.

### Event Flow: Form Submission to Integration

When an end user submits a synced form:

1. **Audience** receives the submission and sends an event to the **Audience Subscription Worker queue**.
2. The **Audience Subscription Worker** picks up the event and:
   - Extracts the `account`, `section`, and `integration ID`.
   - Attempts to extract profile fields: **CRM ID**, **email address**, **phone number**.
   - If no email is found, the email field is omitted. CRM ID will typically be absent for public forms since end users don't supply it.
   - Appends required metadata: activity ID, event timestamp, event type, and **profile key**.
3. The enriched event is published to **Kafka**.

> [Erik Andersson]: "If there is no CRM ID, then we require there to be either an SMS or an email — because that's the secondary type of identification. If there's no CRM ID, no email, and no SMS, then this submission is useless. We can't attach it to any existing resource."

4. A **form event worker** (listening on the Kafka topic) batches the event and assigns it a batch ID.
5. The batch is forwarded to the **Outbound Worker**, which performs the actual send to the CRM system.

### What the Outbound Worker Sends — and What It Does NOT Do

The Outbound Worker **notifies the CRM that a form submission occurred**. It does **not** update any profile attributes in the CRM (no first name, last name, or email address updates).

> [Erik Andersson]: "We are just letting them know: 'yo, someone submitted this form — do what you want with it.' Apsis is never making any actual attribute updates in the CRM system."

The Outbound Worker also handles retries on failure (e.g., HTTP 504 responses from CRM outages).

---

## CRM Response Handling and the Three Outcomes

After the Outbound Worker sends the form submission event, the CRM system can respond in one of **three ways**:

### Outcome 1: CRM Does Nothing
The CRM responds with an empty/acknowledgement response. No record is created or matched. Integration takes no further action.

### Outcome 2: CRM Matches an Existing Contact
The CRM finds a contact matching the submitted email address (or other identifier) and responds with the corresponding **CRM ID** for the Apsis profile key.

- Integration takes this CRM ID and **adds it as an attribute** to the form-submitted profile in Apsis.
- The **Merge Worker** then merges this profile with the profile in the **bootstrapped CRM key space** (e.g., `FSC Enterprise 12.1` key space, where the CRM ID is the key).
- This merge ensures that future Delta Sync updates from the CRM will find and update the correct profile rather than creating a duplicate.

### Outcome 3: CRM Creates a New Contact
The CRM does not find a matching contact, so it creates a new one and responds with the new CRM ID.

- The flow is identical to Outcome 2: Integration adds the CRM ID to the Apsis profile and the Merge Worker merges it into the CRM key space.

> [Erik Andersson]: "In the future, if the CRM system sends us a real-time update in the Delta Sync worker flow, we don't create a duplicate in Apsis — because if you would have just the form-submitted profile with only the email and no CRM ID, that profile would not have been found when we do the profile updates in the real-time sync, because we only look in our custom key space."

---

## Bootstrapped CRM Key Spaces

Each CRM integration installation creates its own **dedicated key space** in Apsis, where the **CRM ID is the key**:

| CRM System | Key Space Name |
|---|---|
| FSC Enterprise | `FSC Enterprise <version>` (e.g., `FSC Enterprise 12.1`) |
| Dynamics | `Dynamics` key space |
| E-deal | `E-deal` key space |

When a profile is synced from a CRM to Apsis, it exists **only** in this CRM key space — it does **not** automatically exist in the email key space or the form key space.

> ⚠️ **This is a critical point for pre-filled forms** — see the section below.

---

## Pre-Filled Forms: Feature Overview

**Pre-filled forms** are currently behind a **feature flag** (to be enabled for customers soon). The feature works by sending a form link to a specific known profile in Apsis. When that profile opens the link, their stored attributes (email, first name, last name, etc.) are pre-populated in the form fields.

---

## Pre-Filled Forms + CRM Integration: Key Technical Risks

### Risk 1: Profile Key Space Resolution — Potential for Duplicates

This is identified as the **most critical technical concern** of the session.

**The problem:** When a profile has been synced from a CRM to Apsis, it exists **only in the CRM key space** — not in the email key space. If a pre-filled form is submitted and the form tool resolves or creates the profile using the **email key space**, and that merge is not also performed against the **CRM key space**, the result is **two separate profiles in Apsis** with the same underlying person.

> [Erik Andersson]: "If you only create one in the e-mail key space and this is not merged with ours, then we have suddenly created a duplicate in Apsis. We have the same data — that's not a good approach."

**The expected correct behavior:** Searching for the profile by email address should return the **same profile ID** as searching by CRM ID. If these return different IDs, the flow is broken.

**What needs investigation (assigned to Tomasz Kowalski):**
- Create a profile in FSC Enterprise 12.1 using a known email address (e.g., a personal test email with `+` suffix).
- Sync that profile to Apsis.
- Submit the pre-filled form as that profile.
- Verify: does searching by email return the same profile ID as searching by CRM ID?

**Erik's proposed correct approach for the form tool:**

> [Erik Andersson]: "I would look at which field mappings exist on this integration — you can use our **Mappings Manager Service**, do a lookup on the account and section, and you'll know if an installation exists. You use that as the `integration ID` parameter. You'll get the mappings object back, which will contain the Apsis One field ID and all mapped attributes. All attributes that exist in those mappings — you should not be able to overwrite that data with data from the form."

If a merge is performed with the email key space, the form tool must also check:
- Does this profile have a CRM ID?
- Is there an active integration on this section?
- If yes, what is the CRM key space — and must a merge also happen there?

### Risk 2: CRM ID Must Never Be Editable in a Form

> [Erik Andersson]: "If you for whatever reason would be able to actually update the CRM ID — I can tell you that we will be in deep **** because this is one of the things you can under no circumstance do if you have an integration. We will lose complete track of that profile and you will definitely introduce duplicates."

**Current state:** It is currently possible to add a CRM ID field to a form and map it to the CRM ID attribute. This **must be blocked**.

**Proposed fix:** If an integration is enabled on the section, the CRM ID attribute must be **hidden from the form editor** entirely.

[Michal Rosikiewicz]: The CRM ID should only be hideable/available in the editor if there is **no integration on the section**. If there is an integration, hide it regardless of other settings.

### Risk 3: CRM-Mapped Attributes Must Not Be Overwritten

**The problem:** Apsis does not support syncing attribute changes back to the CRM. If an end user modifies a CRM-mapped attribute (e.g., first name) via the form, the data in Apsis will differ from the CRM. On the next sync from the CRM, Apsis will overwrite the user's input with the CRM value, silently discarding what was entered.

> [Erik Andersson]: "Whatever you change manually or in the form submissions — we will completely disregard and overwrite. That's why I have always been of the opinion that any data you have set up the field mapping for, you should not be able to modify in Apsis. If you're going to modify that data, it should be modified in the CRM system — then it will be updated in Apsis as a consequence of the sync."

**Current behavior:** Whether mapped attributes can be overwritten depends on the form setting **"Update existing profile data"** — if this is enabled, any attribute can be changed.

**Proposed resolution (product decision required):**

[Michal Rosikiewicz] proposed a third option: **"Block update if profile has CRM ID"** — but Erik clarified this is too broad. The requirement is more granular:

- **Block** overwriting attributes that are in the CRM field mapping.
- **Allow** overwriting or adding attributes that are **not** in the CRM field mapping (enrichment attributes).

This requires the form tool to call the Mappings Manager Service to determine which attributes are CRM-mapped and apply selective blocking.

---

## Attribute Enrichment: What Is and Is Not Supported

### What IS Supported
Adding **new attributes** to a profile in Apsis that have no corresponding CRM field mapping (e.g., a `satisfactory_rating` or `favorite_color` field). These can be collected via the form and stored in Apsis without any conflict with the CRM sync.

> [Erik Andersson]: "If you introduce a new form field called 'satisfactory rating' mapped to a standalone attribute in Apsis called 'satisfactory_rating' — that is completely fine. We will not touch it during syncs because there is no field mapping for it."

### What is NOT Supported (and is a common misunderstanding)
Enrichment attributes collected via form **will not be sent to the CRM system**. They exist only in Apsis.

> [Erik Andersson]: "Just so Henrik is also aware of this and not assuming that 'yeah, we collect all cool data in Apsis and it is automatically added in the CRM' — we support enriching them in Apsis, but we do not support enriching them in the CRM."

### New Development Required for CRM-Side Enrichment
If the requirement is that new enrichment attributes collected via form should also be written back to the CRM, this is **not currently supported** and would require new development in **both Integration and the CRM system**. This was flagged as a likely misunderstanding in Henrik's requirements and needs to be clarified.

> [Erik Andersson]: "I can see the use case that you would want to proliferate this data from Apsis to the CRM system, but this is a completely new use case that hasn't been on the roadmap — the customers have not wanted to sync profile data from Apsis to the CRM system."

---

## Consent Sync: Bidirectional by Design

Unlike attribute updates, **consent is bidirectional** for legal reasons.

- If an end user modifies their consent on a form (e.g., opts into a subscription they previously had no consent for), Integration **will** send this change to the CRM system — **if there is a consent mapping configured for that subscription**.
- If the subscription is not in the consent mapping, Integration does not have a listener registered for it and will not react.

> [Erik Andersson]: "Here there are legal reasons as to why we need to have a bidirectional sync."

**Current issue with pre-filled forms:** Consent/subscription checkboxes are not currently being pre-filled correctly in the form. This is a bug in the form tool to be fixed as part of the pre-fill feature work. Once fixed, the consent sync to CRM should work out-of-the-box via the existing integration flow.

---

## Acceptance Criteria Review (from Henrik's Requirements)

The team reviewed Henrik's acceptance criteria against current system behavior:

| Criterion | Status / Notes |
|---|---|
| When a recipient opens a form link, their profile data is pre-filled | ✅ Works for known profiles |
| Email field is locked (read-only) | ✅ Confirmed |
| CRM-sourced attributes that differ from input are not overwritten in Apsis | ⚠️ Currently depends on "Update existing profile data" form setting — needs selective blocking by mapping |
| CRM ID is not available to modify in the form editor | ❌ Currently possible — must be hidden when integration is on the section |
| Update request is sent to CRM for enriched attributes | ❌ **Not supported** — this will never happen today; needs product/CRM-side discussion |
| CRM remains master and decides whether to update | ✅ Correct by design — but this means no update request is sent to CRM |
| Non-CRM attributes are stored in Apsis | ✅ Will work as normal, but data remains Apsis-only |
| Submissions do not create duplicate profiles | ⚠️ **Unknown — to be investigated by Tomasz** |
| Audit log shows which fields came from CRM, what changes were suggested, what was stored locally | ❓ **Unknown** — no audit log mechanism identified. Currently, source cannot be specified on attribute updates (only on consent updates). Where these logs would be stored and who would write/read them is unclear. |
| Consent can be shown and updated | ⚠️ Pre-fill of consent fields is broken currently — fix is in scope for form tool; sync to CRM should then work out-of-the-box |

---

## Key Takeaways

1. **The CRM key space is isolated** — profiles synced from CRM exist only in the bootstrapped CRM key space, not in email/SMS/form key spaces. Any form submission flow must account for this or risk creating duplicates.

2. **Merge on form submission is the critical linking step** — when a submitted profile is matched to an existing CRM contact, the Merge Worker links the email key space profile to the CRM key space, preventing future duplicates from Delta Sync.

3. **CRM ID must be completely hidden from the form editor** when an integration is installed on the section — modifying the CRM ID would cause Integration to lose track of the profile entirely.

4. **Attribute overwrite protection must be granular** — CRM-mapped attributes must be blocked from overwrite; non-mapped enrichment attributes must remain writable.

5. **Enrichment data does not flow back to the CRM** — this is a hard architectural limitation today. If Henrik's requirements assume CRM-side enrichment, this needs a new development track and must be explicitly discussed.

6. **Consent sync is bidirectional by design and for legal reasons** — fixing the pre-fill bug for consent checkboxes is sufficient; the downstream sync to CRM will work automatically for mapped subscriptions.

7. **The Mappings Manager Service** can be queried by account + section + integration ID to determine whether an integration exists and which attributes are CRM-mapped — the form tool should use this to drive both the "hide CRM ID" and "block overwrite" behaviors.

---

## Unresolved Questions and Action Items

| Item | Owner | Notes |
|---|---|---|
| Investigate profile key space behavior on pre-filled form submit: does submitting create a duplicate or correctly merge with CRM key space? | **Tomasz Kowalski** | Erik to provide exact reproduction steps. Use FSC Enterprise 12.1. Create test contact in CRM with known email, sync to Apsis, submit pre-filled form, compare profile IDs. |
| Clarify with Henrik whether "update request sent to CRM for enriched attributes" is a hard requirement or a misunderstanding | **Lukasz / Erik** (meeting with Henrik scheduled for next Monday) | If it is a real requirement, this is new development for both Integration and the CRM system. |
| Determine what "audit log" means in Henrik's acceptance criteria — where it would be stored, who writes it, who reads it | **To be discussed with Henrik** | No existing audit log mechanism identified for attribute updates. |
| Product decision: implement selective attribute blocking in form tool (block CRM-mapped fields, allow enrichment fields) | **Product + Form Tools team** | Requires integration with Mappings Manager Service. |
| Hide CRM ID attribute from form editor when integration is enabled on the section | **Form Tools team** | Must be conditioned on integration presence on section, not a global flag. |
| Fix pre-fill of consent/subscription checkboxes in the form | **Form Tools team** | Once fixed, downstream consent sync to CRM should work out-of-the-box. |
| Verify that Tomasz has access to FSC Enterprise 12.1 for testing | **Tomasz Kowalski** | Confirmed access — added to search folder. |