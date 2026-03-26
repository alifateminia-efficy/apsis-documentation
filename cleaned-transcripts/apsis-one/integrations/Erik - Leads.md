---
source_file: Erik - Leads.txt
domain: Apsis One Integrations
topics: [lead creation, form submission flow, outbound mappings, CRM sync, profile merging, consent syncing, key spaces, batch processing]
speakers: ["Erik Andersson (subject matter expert)", "Lukasz Grabowski", "Michal Rosikiewicz"]
key_components: [Apsis One, Audience Subscription Worker, Batch Production Worker, Outbound Worker, Generic Connector, Kafka, CloudWatch, FCC Enterprise 12.1, CRM key space, email key space, export key space]
session_type: knowledge-transfer
subdomains: ["Lead creation"]
---

## Session Overview

Erik Andersson walks Lukasz Grabowski and Michal Rosikiewicz through the full lead-creation flow in Apsis One integrations, focusing on how form submissions are gathered, synced to a CRM system, and reconciled back into Apsis One. The session covers the outbound worker pipeline, the role of outbound mappings in enriching events, the profile merge process and key-space mechanics, and the consent synchronization that follows creation of a new CRM contact. The second half of the session is a live demo using FCC Enterprise 12.1, where Michal creates and submits a form while Erik narrates what is happening at each stage in the pipeline.

---

## Lead Creation: Conceptual Overview

**Lead creation** in the Apsis context means: a contact does not yet exist in the CRM system, but is collected via Apsis (primarily through a form submission) and then sent to the CRM by Apsis, resulting in a new record.

Everything related to lead gathering happens in the **outbound flow**.

Forms are the primary — and designed-for — mechanism for lead gathering, although lead creation can also be triggered by email submit events and other activity types (see caveat at the end).

---

## Outbound Flow: Step-by-Step Pipeline

The flow is triggered by a form submission event originating in Audience and proceeds through the following stages:

### 1. Audience Subscription Worker (Odd Sub Worker)

- Receives the form submit event from Audience via the **common SMS topic** (Kafka), which shuffles it to the **Odd Sub Worker queue** (a FIFO queue).
- Verifies that all required data is present for the sync.
- If the section has **outbound mappings** configured, the worker **enriches the event** with mapped attribute values at this stage. For example: if the profile has a `first_name` attribute and an outbound mapping is configured to send `first_name`, that value is included in the event payload before it moves downstream.

### 2. Batch Production Worker

- Batches events up to **200 KB per section**.
- Example: five form submissions from the same section are gathered into a single batch.

### 3. Outbound Worker

- Sends the batch to the CRM system via the generic connector endpoint.
- The request payload for each profile contains two distinct groups of fields:
  - **Event data fields**: the raw form field names and their submitted values (e.g., field named `motorbike`, `refrigerator`, `banana` — whatever the customer named them).
  - **Mapped fields**: the outbound mapping enrichment — structured fields such as `first_name`, `last_name`, `email` — which tell the CRM what attribute role each value plays, regardless of how the form field was named.

> "You didn't name your form field like email, first name, last name — it's like motorbike, refrigerator and banana. You would not really know what data should I put in the first name. The mapped field helps you with this."

- Not all CRM systems have implemented mapped fields, but the functionality exists in the generic connector.

---

## Outbound Mappings: Two-Stage Configuration

Outbound mappings are a **two-stage setup** — both stages must be in place for lead enrichment to work correctly:

1. **Form field → Apsis One attribute mapping**: Configured on the form itself. Maps each form field to an Apsis One attribute (e.g., the form field `email` → the `email` attribute on the profile).
2. **Apsis One attribute → CRM attribute mapping (outbound mapping)**: Configured on the integration page. Takes the value of an Apsis One attribute and maps it to the corresponding CRM field.

**Important distinction that causes common confusion:**
- **Field mappings** (on the integration page, inbound direction): attribute changes from the CRM system → Apsis One.
- **Outbound mappings** (on the integration page, outbound direction): enrichment of form submit events from Apsis One → CRM system.

> "The outbound mappings are NOT mappings for attribute changes. We don't sync attribute changes to the CRM. The field mappings are attribute changes from the CRM to Apsis, but the outbound mappings are the enrichment of the form submit events from Apsis to FCC Enterprise."

The **"CRM Sync" checkbox** must be selected on the form itself; without it, the outbound worker will not listen for submit events from that form.

---

## CRM Response and Profile Key Handling

After the outbound worker sends the batch, the CRM system responds with:
- **New records**: profiles for which a new contact/lead was created, along with the newly assigned **CRM ID**.
- **Matched records**: profiles that already existed in the CRM, along with their existing CRM ID.
- **No action**: if the CRM cannot identify a contact (e.g., no CRM ID, no email, no SMS provided), it may not return any data for that profile.

The CRM system itself decides whether an incoming profile matches an existing record, using the identifying data Apsis provides (CRM ID, email, etc.). Apsis does not control this deduplication logic.

---

## Profile Merge Process

After receiving the CRM response, Apsis triggers a **merge** to make the profile findable via the CRM key space:

- At form submission time, the profile exists in the **form key space** and most likely the **email key space**.
- After the CRM responds with a CRM ID, Apsis performs a merge specifying:
  - **Export key space** + profile key (from the form submission side)
  - **Custom CRM key space** + CRM ID (from the CRM response)

This makes the profile identifiable via the CRM key space, which is used for all future inbound syncs. This eliminates potential duplicates: any future updates from the CRM system will resolve to the same profile that was created from the form submission.

### CRM Data as Master After Merge

After the merge completes, Apsis downloads all contact data from the CRM system and applies it as an update to the Apsis profile. This is intentional:

> "We want the data from the CRM system to be the master data. That means that as soon as the merge is finished, we download all of the data from the CRM system for the contacts and make an update inside of Apsis One, just to make sure that regardless of the outcome of the merge, it is the data from the CRM system that is present on the contacts."

**Consequence observed in demo**: If a field (e.g., `middle_name`, `mobile_number`) has a mapping configured but the CRM returns an empty value for it, Apsis interprets this as the user having cleared that field in the CRM and will clear it in Apsis too. This caused the middle name and mobile number set during the form submission to be erased after the merge, because FCC Enterprise 12.1 was not storing those values properly.

---

## Consent Synchronization for New Leads

When the CRM responds with a **new record** (newly created contact/lead), Apsis immediately triggers a consent export for that contact. This is necessary because:

> "If we don't send over any consent for this new lead... you would not be able to do anything with this lead because you wouldn't be able to contact them — you couldn't use email, you couldn't call them or send a text because you don't have any consent."

Mechanism:
- Apsis exports all consents for which a subscription mapping is configured.
- Uses the **`PATCH consent` endpoint** on the generic connector to update the CRM contact with those consents.
- This is the **only bidirectional operation** in the sync: consent changes made in Apsis (e.g., an end user clicking unsubscribe in an email) are always sent to the CRM.

**Why consent updates must be bidirectional (legal requirement):**
> "The end users can modify their consents inside of Apsis when they click on unsubscribe in emails or in SMS, and that means we must send this change to the CRM system. Otherwise we will be in a situation where the customer says 'you are not allowed to contact me via email', but the next time there's a sync from the CRM system, we would overwrite their opt-out with the opt-in in the CRM system — and then we would be in all kinds of trouble."

The generic connector exposes a `PATCH consent` endpoint specifically for this purpose. Attributes on entities **cannot** be updated through the generic connector (only fetched), but consents can be updated.

---

## Form Publish: The One Synchronous Step

When a form is published in Apsis One with CRM sync enabled:

- Apsis synchronously calls the CRM system to create a corresponding campaign/form record there.
- This is the **only synchronous operation** in the entire lead creation flow.
- If the call to the CRM fails, the form publish is **blocked** — the customer will be notified immediately.

---

## Email Key Space Constraint

Because the form submission creates a profile in the email key space, that email address becomes **unique within Apsis**. This means:

- Multiple people cannot generate separate leads with the same email address through this flow.
- However, contacts created directly in the CRM system with a duplicate email address are not affected, because the normal inbound sync flow does not merge on the email key space.

---

## Live Demo Observations (FCC Enterprise 12.1)

The demo used the FCC Enterprise 12.1 sandbox. Key observations:

- CloudWatch logs on the **Audience Subscription Worker** showed the enriched submit event, including `mapped_fields` with `first_name` and `phone`.
- The **Outbound Worker** logs showed the CRM response confirming a new contact was created with a CRM ID.
- Searching for the profile in Apsis after the flow showed it present in both the **email key space** (from form submission) and the **FCC Enterprise 2 key space** (from the CRM ID returned after sync).
- **Known issue with FCC Enterprise 12.1**: This CRM system is not fully utilizing the `mapped_fields` sent in the request. The first name and phone number were included in the Apsis request payload, but FCC Enterprise did not store them correctly on the contact record. [Erik Andersson]: "I'm going to ping them about it, but the system is working as far as we are concerned — we are providing the data in the requests."

---

## Scope of Lead Creation: Forms vs. Other Activity Types

Currently, outbound mappings enrichment is only applied for **form submit** events:

> "The only time we add the mapped fields is for form submissions. If you would like to change this in the future, that is trivial to do — we just have a check: if it is a form submit, then we add the outbound mappings."

Other activity types (email submits, task notes, MA flow events) do trigger the same pipeline (Audience Subscription Worker → Batch Production Worker → Outbound Worker), and the CRM will respond with matched/new records and CRM IDs. However, **outbound mapping enrichment is not currently added for those event types**.

CRM systems confirmed to support form syncing / lead creation:
- FCC Enterprise 12.1
- Tribe (noted as having implemented outbound mappings properly)
- MaxMail (Max) — ⚠️ [speaker said "Max so is not" — ambiguous whether this means MaxMail does not support it or does not implement outbound mappings properly]
- E-Deal (supports lead generation)

---

## Key Takeaways

1. **Lead creation is entirely outbound**: a form submission event flows from Audience → Odd Sub Worker → Kafka → Batch Production Worker → Outbound Worker → CRM, with no user-facing synchronous steps except the initial form publish.
2. **Outbound mappings are a two-stage pipeline**: form field → Apsis attribute, then Apsis attribute → CRM field. Both must be configured.
3. **CRM data wins after merge**: after a new lead is created, Apsis downloads CRM data and applies it as master, overwriting anything set during the form submission (including potentially clearing fields the CRM didn't store).
4. **Consent sync is the only bidirectional operation** and is legally required. New contacts also get an initial consent push immediately after creation.
5. **The email key space creates uniqueness**: a given email address can only produce one lead profile through the form flow.
6. **The merge + CRM key space linkage prevents future duplicates**: after a lead is created, all future CRM syncs will find the same profile via the CRM key space.
7. **Mapped fields enrichment is currently form-submit-only** but is trivial to extend to other event types.

---

## Unresolved Questions / Action Items

- **Action (Erik)**: Ping FCC Enterprise 12.1 team regarding their failure to utilize `mapped_fields` from the request payload (first name and phone number not being stored on the contact).
- **Action (Erik / team)**: Add the outbound mappings diagram (including the outbound mappings step) to the architectural diagrams folder in the MA Integration repository. The existing PNG diagram does not include outbound mappings.
- **Ambiguous**: Whether the behavior of CRM data always overwriting Apsis data post-merge is intentional long-term. [Erik Andersson]: "If this should still work like this going forward, I don't know, but it would be quite easy to change if you would so want to."
- **Ambiguous**: MaxMail (Max) support status for lead generation / outbound mappings — the speaker's statement was unclear. ⚠️ Needs verification.
- **Recommendation (Erik)**: Team members should create a form and run through the full lead creation flow themselves on the sandbox to build hands-on familiarity.