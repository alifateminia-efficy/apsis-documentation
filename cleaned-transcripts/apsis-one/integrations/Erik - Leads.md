---
source_file: Erik - Leads.txt
domain: Apsis One Integrations
topics: [lead gathering, outbound flow, form syncing, CRM integration, outbound mappings, profile merging, consent sync, batch processing, key spaces]
speakers: ["Erik Andersson (subject matter expert)", "Lukasz Grabowski (attendee)", "Michal Rosikiewicz (attendee)"]
key_components: [audience subscription worker, batch production worker, outbound worker, generic connector, FCC Enterprise 12.1, Kafka, CloudWatch, Apsis One forms, CRM key space, outbound mappings]
session_type: knowledge-transfer
---

## Session Overview

This session covers the complete lead-gathering flow in Apsis One integrations, from a user submitting a form through to a new contact being created and synced in a CRM system. Erik Andersson walks through the architecture of the outbound flow — covering the audience subscription worker, Kafka batching, the outbound worker, CRM response handling, profile merging, and consent synchronization. The session includes a live demonstration using FCC Enterprise 12.1 as the target CRM, with Michal sharing his screen to create and submit a test form. Key subtleties around outbound mappings, CRM master data, key spaces, and duplicate prevention are discussed in depth.

---

## Overview of the Lead Gathering Outbound Flow

**Lead gathering** in Apsis One happens entirely within the **outbound flow**. The primary mechanism for gathering leads is via **forms**, though as noted in a prior session, it can also occur via emails.

A **lead** in this context is something that does not yet exist in the CRM system but is collected through Apsis and then sent to the CRM by Apsis.

### Step-by-Step Flow

1. **Form submit event** — A user submits a form configured to sync to the CRM system. This generates an event from Audience to the integrations layer.

2. **Audience Subscription Worker (odd sub worker)** — The first step in the outbound flow. It:
   - Verifies that all required data is present to perform the sync.
   - If the event is a form submit and outbound mappings are configured for the section, enriches the event with mapped attribute data (e.g., first name, last name, email).

3. **Kafka** — After a brief stop in Kafka, the event is passed to the Batch Production Worker.

4. **Batch Production Worker** — Gathers events up to **200 kilobytes** per section. For example, if five people submitted the same form, those five form submit events are aggregated into one batch and passed to the Outbound Worker.

5. **Outbound Worker** — Sends the batch request to the CRM system via the **generic connector**.

> The audience sends events to our common SMS topic. The SMS topic shuffles things over to the odd sub worker queue, which is a FIFO queue. The odd sub worker picks this up.

---

## The Generic Connector: Batch Request Format and CRM Response

### Batch Request Structure

The outbound worker sends the batch to a dedicated endpoint in the **generic connector**. For a form submit event, the payload contains:

- **Profile identity** (the Apsis profile key)
- **Pure submit event data** — the raw field names from the form (e.g., field names might be arbitrary like "motorbike", "refrigerator", "banana")
- **Mapped fields** (outbound mappings) — if configured, these translate Apsis attribute values to CRM-recognizable fields such as first name, last name, and email

[Erik Andersson]: "The outbound mappings say: if you create a new lead from this submission, then the lead should have this first name, this last name, and this email. The mapped field helps you with this — because you didn't name your form fields 'email', 'first name', 'last name', you wouldn't know what data to put where."

Not all CRM systems have implemented the mapped fields functionality, but the generic connector supports it.

### CRM Response

The CRM system responds with:
- **New records** — profiles that were newly created in the CRM, including the generated **CRM ID**
- **Matched records** — profiles that already existed in the CRM, with their CRM IDs

The CRM decides whether an incoming profile is new or existing based on the identifying fields provided (e.g., CRM ID, email, SMS number). If none of these match anything and none are present, the CRM may ignore the profile entirely and return nothing in either `new_records` or `matched_records`.

---

## Profile Merging After CRM Response

Once the CRM responds with a CRM ID, Apsis needs to make the submitted profile identifiable via the **CRM key space**.

The submitted profile typically exists in:
- The **form key space**
- The **email key space**

A **merge** is triggered, specifying:
- The export key space + profile key (from the form submission)
- The custom **CRM key space** + CRM ID (from the CRM response)

### CRM as Master Data

In a standard Apsis merge, the attribute with the latest timestamp wins. However, the design decision here is that **CRM data is the master data**. To enforce this regardless of merge outcome:

> After the merge completes, Apsis downloads all data from the CRM system for the contact and performs an update in Apsis One — ensuring that CRM data is always what's present on the contact.

[Erik Andersson]: "If this should still work like this going forward, I don't know, but it would be quite easy to change if you would so want to."

### Side Effect: Empty Field Overwrite

If a field has a mapping configured (e.g., middle name, mobile number) and the CRM returns an empty value for it, Apsis interprets this as the user having cleared the field in the CRM, and **clears it in Apsis as well**. This was observed live during the demo when the first name and phone number were cleared from the profile after the post-merge CRM data download.

---

## Consent Synchronization for New Leads

When the CRM responds with a **new record** (a newly created contact), Apsis immediately triggers a consent export. This is critical because:

> "If we don't send over any consent for this new lead, you wouldn't be able to contact them via email, call them, or send texts because you don't have any consent."

The process:
1. Detect new record in CRM response.
2. Export all consents in Apsis for which a subscription mapping exists.
3. Send those consents to the CRM system using the **patch consent endpoint** on the generic connector, now that the CRM ID is known.

### Why Consents Are the Only Bidirectional Sync

Attribute updates are **one-directional** (CRM → Apsis). The generic connector has no endpoint to update entity attributes from Apsis to CRM. However, **consent updates are bidirectional** because of legal requirements:

> "The end user can modify their consents inside Apsis when they click unsubscribe in emails or SMS, and we must send this change to the CRM. Otherwise, the next full sync from the CRM would overwrite their opt-out with an opt-in in the CRM, and we would be in all kinds of trouble."

The `patch consent` endpoint requires knowing the entity and its CRM ID, which is available at this point in the flow.

---

## Form Configuration Requirements for Lead Sync

For the lead sync flow to work end-to-end, two independent mapping configurations must both be in place:

1. **Form field → Apsis attribute mapping** — maps form input fields to attributes in Apsis One. This is configured on the form itself.
2. **Outbound mappings** — maps Apsis One attributes to CRM fields. Configured on the integration page. These are the fields used to enrich form submit events sent to the CRM.

[Erik Andersson]: "Outbound mappings work on the attribute level in Apsis One. We take the value from the Apsis attributes and map that to the attributes in the CRM. It's: form fields → Apsis attributes → outbound mappings. Both need to be in place for this to fully work."

**Important distinction that causes confusion:**
- **Field mappings** = attribute changes flowing **from CRM to Apsis** (inbound)
- **Outbound mappings** = enrichment of form submit events flowing **from Apsis to CRM** (outbound)
- Outbound mappings are **not** attribute sync mappings; they are event enrichment.

Additionally, when creating the form, the **"CRM sync" checkbox must be selected**. Without it, Apsis will not listen to submit events from that form.

---

## Form Publish: Synchronous CRM Call

When a customer publishes a form configured for CRM sync, Apsis makes a **synchronous call** to the CRM system to create a corresponding campaign/form entity in the CRM.

> "This is the only thing in the whole lead creation flow that happens synchronously. When you click publish, we directly call the CRM. We tell you immediately if that failed or not. You cannot publish the form if our call to the CRM fails — that is a blocked operation."

---

## Key Space and Duplicate Prevention

After the merge, the profile exists in both:
- The **email key space** (from form submission)
- The **FCC Enterprise key space** (using the CRM ID)

[Erik Andersson]: "For any future updates from the CRM system, this will not lead to a duplicate in Apsis. We always utilize our FCC Enterprise key space when we do updates — meaning we will find the same profile as was created from the form submission. We have eliminated any potential duplicates."

This applies to:
- Consent updates from CRM
- Contact attribute updates
- Full sync updates via generic connector

### Caveat: Email Address Uniqueness Constraint

Because profiles are created in the email key space during form submission, **the email address becomes unique within that flow**:

> "Multiple people cannot submit the same email address to generate multiple leads in this flow."

However, contacts created directly from the CRM system with the same email address **can** coexist, because the normal sync flow does not merge on the email key space.

[Michal Rosikiewicz]: "But when we submit the form again with the same email, will we receive any events on this profile?"
[Erik Andersson]: "You would not create a new profile — you would use the same one because you are utilizing the email key space. And we would get a matched record from the CRM instead of a new record."

---

## CRM Systems Supporting Lead/Form Sync

The following CRM systems support the lead gathering flow (form syncing):
- **FCC Enterprise 12.1** ✓
- **Tribe** ✓ — also properly implements outbound mappings
- **Max** — does not fully implement outbound mappings ⚠️
- **eDeal** ✓ — supports lead generation

### Known Issue: FCC Enterprise 12.1 Not Fully Utilizing Outbound Mappings

During the live demo, it was observed that FCC Enterprise 12.1 did not apply the mapped fields (first name, phone number) to the created contact record in the CRM, even though the data was correctly provided in the request.

[Erik Andersson]: "The system is working as far as we are concerned — we are providing the data in the requests. If they are not actually fully utilizing them, that's bad. I'm going to ping them about it."

---

## Scope of the Outbound Mappings — Form Submits Only (For Now)

Currently, outbound mappings (mapped fields enrichment) are only applied to **form submit events**. This is not an architectural limitation — it is simply because lead gathering was only requested for form submits.

```
// Pseudocode of the current check in the outbound worker
if (event.type === "form_submit") {
    addOutboundMappings(event);
}
```

[Erik Andersson]: "The generic connector does support it for anything. We can add outbound mappings data for email submits, task notes, MA flow things, or whatever. We just have a check: if it is a form submit, then we add the outbound mappings. The only reason we have that check is because it was only requested for form submits."

**Changing this is described as trivial.**

The same flow (odd sub worker → batch production worker → outbound worker → CRM response + CRM ID → merge) applies to **activities synced from Apsis**, except that outbound mappings are not currently enriched for activity events.

---

## Key Takeaways

1. **Lead gathering is entirely outbound**: Form submit → odd sub worker → Kafka → batch production worker (up to 200KB) → outbound worker → CRM.
2. **Two mapping configs are both required**: Form field→Apsis attribute mappings AND outbound mappings. They serve different directions and are configured in different places.
3. **CRM is master data**: After merge, Apsis always downloads CRM data and overwrites Apsis attributes to enforce CRM as the source of truth. Empty CRM values will clear Apsis attributes.
4. **Consent is the only bidirectional sync**: Legal requirement. Opt-outs in Apsis must propagate to CRM to prevent them being overwritten on next sync.
5. **New leads get consent immediately**: When the CRM returns a new record, Apsis automatically exports all mapped consents to the CRM.
6. **Form publish is the only synchronous CRM call** in the entire flow. It will block the publish if it fails.
7. **Post-merge, the profile exists in both email and CRM key spaces**, preventing duplicates on future CRM-initiated syncs.
8. **Email address uniqueness caveat**: The same email cannot generate multiple leads via the form flow. This is an inherent side effect of using the email key space.
9. **Outbound mappings enrichment is trivially extensible** to event types beyond form submits — it is only a conditional check.
10. **FCC Enterprise 12.1 is not fully consuming outbound mapped fields** — this is a known issue to be raised with the FCC Enterprise team.

---

## Unresolved Questions / Action Items

- [ ] **Erik**: Ping the FCC Enterprise 12.1 team about their failure to use the mapped fields (first name, phone number) that are correctly provided in the outbound request.
- [ ] **Erik / Michal**: Upload the outbound flow architectural diagram to the integrations architectural diagrams folder (add outbound mappings to the existing diagram or create a new one alongside it).
- [ ] **Open design question**: Should CRM always be master data going forward? Erik noted this could be easily changed if desired, but no decision was made.
- [ ] **Clarification needed**: ⚠️ The exact behavior when re-submitting a form with an existing email address and whether a `matched_record` response triggers the same post-merge CRM data download as a `new_record` was not fully confirmed in the session.
- [ ] **Recommended action for Michal/Lukasz**: Independently recreate the form submit flow end-to-end in the sandbox to build familiarity with the sequence.