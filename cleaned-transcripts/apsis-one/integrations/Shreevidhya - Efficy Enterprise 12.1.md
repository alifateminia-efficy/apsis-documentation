---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One Integrations
topics: [Efficy Enterprise 12.1 installation, delta sync, full sync, sync conditions, queries and profiles, subscription mappings, field mappings, bidirectional consent sync, CloudWatch logging, generic connector, multi-section installation constraints]
speakers: ["Shreevidhya Ganesan (Subject Matter Expert)", "Lukasz Grabowski (Team Lead/Presenter)", "Michal Rosikiewicz (Developer)", "Tomasz Kowalski (Developer)"]
key_components: [Efficy Enterprise 12.1, Delta Sync Manager, Delta Sync Worker, CloudWatch, Generic Connector, Full Sync, Subscription Mappings, Sync Conditions, Queries and Profiles, LastPass]
session_type: knowledge-transfer
---

## Session Overview

This session is a hands-on knowledge transfer covering the installation and core functionality of the **Efficy Enterprise (FEC) 12.1** integration within the Apsis One platform. Shreevidhya Ganesan walks Lukasz Grabowski and team through a live installation in a new account section, demonstrating delta sync, full sync, subscription mappings, sync conditions, and the queries-and-profiles (tagging) feature. The session also covers where to find logs in CloudWatch, how the generic connector differs from the legacy connector, and constraints around multi-section installation for different CRM types. The team acknowledges that a deeper technical dive (including the delta sync webhook registration mechanism and the full architecture) is planned for a separate session with Eric.

---

## Installation Setup: Efficy Enterprise 12.1 in a New Section

### Section and Integration Naming

- Each section can have **one CRM integration installed per section**. This is a known limitation discussed prior to this session.
- Best practice: name the section or integration clearly (e.g., `EE 12.1`) so it is easy to identify which CRM installation lives in which section.
- Two URLs are required for setup: one for the CRM system itself and one for the Apsis One installation endpoint.

### API Key Creation

[Michal Rosikiewicz]: "Can you show us where and how we can create this API key in FEC 12.1 CRM?"

[Shreevidhya Ganesan]: The API key was provided as part of the environment provisioned by the CRM vendor. The team did not have permissions to create new integrations/API keys directly in that environment. If this is needed, the recommendation is to ask someone from the enterprise team.

> "I have not explored so much about it — we had the environment and API key and then just used it."

---

## Field Mappings and Auto-Mapping

- After installation, navigate to **field mappings**.
- Use **auto-mapping** to automatically map available fields, then save.
- In this session, `mobile` was mapped as a field. It was later confirmed in logs that a mobile number with an invalid format (too short, not matching `+46700...` format) was skipped due to **phone number validation logic** inside the processing pipeline.

> **Warning:** Data validation for fields like mobile numbers can cause silent skips. If a field value fails validation, the attribute update is skipped but processing continues for other fields.

---

## Delta Sync: How It Works

### Mechanism

[Shreevidhya Ganesan]: As part of the installation process, the integration **registers a callback URL** (webhook) in the external CRM system.

- Whenever a change occurs in the CRM for a mapped field, the CRM sends a notification to the integration's callback URL, including the **ID of the profile** that changed.
- The fields sent during registration tell the CRM which fields to watch — only changes to those mapped fields trigger the webhook.

> "When we do a registration we will be sending the fields whatever we have mapped in our system to the external system, so that the external system will know that these are the fields which have been mapped. And any changes in that field, the webhook will be triggered."

- The callback URL is handled by the **Delta Sync Manager** service.

### Timing Behavior

- For **Microsoft Dynamics**: delta sync is essentially immediate.
- For **FEC Enterprise**: there is a delay of a couple of minutes because FEC batches change notifications before dispatching them.

> "They try to collect the changes and try to send it in a short burst, I guess. So that's why this delay."

### Relationship to Full Sync

- Once installation is complete, **delta sync starts running automatically in real time** — no manual trigger required.
- **Full sync** is needed for the initial import of existing data. After that, delta sync keeps records in sync going forward.

> "Full sync is required for the first time because you want to pull all the records from the external system. From there onwards, delta sync will try to keep the contacts in CRM and Apsis profiles in sync."

---

## Bidirectional Sync: Consent vs. Attributes

This is a critical behavioral distinction:

| Direction | Attributes | Consent |
|-----------|-----------|---------|
| CRM → Apsis | ✅ Yes (inbound) | ✅ Yes (inbound) |
| Apsis → CRM | ❌ No | ✅ Yes (outbound) |

- **Attributes are NOT bidirectional.** Changes made directly to a profile in Apsis will be overwritten by the next delta sync from the CRM if those fields are mapped.

> "If you change something in Apsis and then go and change something in the CRM, the Apsis value will be overwritten by data from the CRM. That's very important."

- **Consent IS bidirectional.** The live demonstration showed:
  1. Profile opted out in Apsis → reflected in FEC Enterprise CRM within minutes.
  2. Profile opted back in from the CRM → reflected in Apsis profile via delta sync.

- The **inbound** process: CRM → Apsis (record and consent changes from the CRM).
- The **outbound** process: Apsis → CRM (consent changes originating in Apsis).

---

## Subscription Mappings

- Navigate to the integration's **subscription mappings** section.
- Create a **subscription** (e.g., `Profile QA`) and associate it with a **folder** (e.g., folder named `EE 12.1`).
- Map the subscription to an **e-mail channel**.
- Once mapped, the consent state for that subscription is kept in sync bidirectionally.

In the demo, profile ID `179` in the CRM was confirmed to have been synced to Apsis and was visible in the **Audience** section. The profile's **CRM ID** (`179`) is stored as a separate attribute on the Apsis profile, distinct from the Apsis-internal profile key visible in the URL.

---

## CloudWatch Logging: Where to Find Delta Sync Logs

### Log Groups to Check

Two separate log groups are relevant:

1. **Delta Sync Manager** logs — shows the incoming webhook request payload (fields received, message production). This is the entry point.
2. **Delta Sync Worker** logs — shows the actual processing, including validation failures and attribute updates.

> "Delta Sync Manager is where you produce the messages in the format that the worker can start processing with."

### How to Search Logs

- Use **CloudWatch** on the staging environment (AWS).
- Filter by time window (e.g., last 10 minutes).
- Key identifiers to search for in log lines:
  - **Integration key** — encodes `account_id / section_id / integration_name`
  - **Message Group ID** — encodes `account / section / integration_id / CRM_ID`
  - **CRM ID** — the external system's profile identifier (e.g., `179`)

Example log structure observed:
```
integration_key: <account_id>/<section_id>/<integration_name>
message_group_id: <account>/<section>/<integration_id>/<crm_id>
```

### Example: Identifying a Validation Failure

In the session, a log entry was found in **Delta Sync Worker** showing:
```
updating attribute mobile with delete status — failed validation
```
Root cause: the phone number entered (`+46700...` style but too short) failed format validation, causing the `mobile` attribute update to be silently skipped.

---

## Sync Conditions

### What They Are

**Sync conditions** are filters that restrict which CRM profiles get synced into Apsis. They are evaluated **on the Apsis side**, not sent to the CRM as query filters.

> [Michal Rosikiewicz]: "So this is filtered on our side, not communicated to the CRM?"
> [Shreevidhya Ganesan]: "Yes, on our side. That's why we see 90 profiles come in but only one matching."

### How They Work

- Conditions are defined as attribute-value pairs (e.g., `email = lukasz@example.com`).
- **All conditions must be satisfied** — it is a logical AND.
- Applies to both **full sync** and **real-time delta sync**: if an update arrives for a profile that doesn't satisfy the conditions, it is skipped/discarded.

### Demo Result

- One sync condition set: `email = <Lukasz's email>`.
- Full sync result: **1 profile synced, remaining profiles skipped**.
- Confirmed in the **full sync report** in the integrations UI.

### Feature Flag Note

A `contains` operator for sync conditions exists as an enhanced feature but is **behind a feature flag** — it requires the external CRM system to support it before it can be released.

---

## Queries and Profiles (Tag Import)

### What This Feature Does

**Queries and Profiles** allows importing a list of profiles from a pre-existing **query** defined in the external CRM system, and **tagging** the matching Apsis profiles.

Use case example: a CRM has a query for "VIP status" profiles. Running the import tags those Apsis profiles as VIP, enabling segmentation in Apsis campaigns.

> "It tags the profile for a particular query result, so in segmentation it can become handy to just filter out profiles which have the tag VIP."

### Recurring Option

- If **recurring** is NOT checked: import runs once.
- If **recurring** IS checked: a **daily cron job** keeps the tag list in sync with the CRM query results.

### Demo Result

- Used existing query `apsis test` from the FEC Enterprise environment.
- Ran "Start Import" immediately (does not require waiting for cron).
- Result: **5 profiles processed**, new tag `FS Enterprise` created and visible on profiles in Apsis Audience.
- After a subsequent full sync (with sync conditions removed), profile data (e-mail, CRM ID `87`) was visible on the imported profiles.

### Important Note

Shreevidhya has only used **existing queries** from the external system for testing and has not created new queries herself. Eric is expected to cover query creation in a follow-up session.

---

## Multi-Section / Multi-Account Installation Constraints

This is a significant architectural constraint that varies by connector type:

| Connector | Multiple Sections? |
|-----------|-------------------|
| **FEC Enterprise** | ✅ Yes — same CRM instance can be installed in multiple sections |
| **Maxo** | ❌ No — one instance, one section only |
| **Microsoft Dynamics** | ❌ No — one instance, one section only |
| **Side Shop** | ❌ No — one instance, one section only |

The reason for the restriction on some connectors is related to a feature called **deep linking** — where selecting a profile in Apsis takes you directly to that profile in the external CRM. This requires a unique 1:1 relationship between section and CRM instance.

> "There's a feature called deep linking where you can just select the profile and it directly takes you to the profile in the external system. So [that's why the restriction exists]."

### Testing Credentials

- CRM API keys and URLs for test instances are stored in **LastPass**.
- Search by connector name (e.g., `FEC Corporate`, `Tribe`) to find the relevant credentials.
- For some connectors, there is no front-end UI for the CRM — testing is purely via the integration layer.

---

## Generic Connector vs. Legacy Connector

- The FEC Enterprise 12.1 integration uses the **Generic Connector**, not the legacy connector.
- The generic connector is described as faster and better-performing than the legacy approach.
- The strategic direction (per Benjamin's session the previous day) is that **all new external systems should use the generic connector approach**.
- New CRM systems wanting to integrate must adhere to a **standardized specification**.
- A **Postman collection** exists for testing generic connector endpoints. It requires only:
  - The API key
  - The prefixed URL of the external CRM

> "The generic specification we have developed is more standardized and straightforward. You can try all those endpoints by yourself using a simple Postman request."

---

## Key Takeaways

1. **Delta sync is webhook-driven**: the integration registers a callback URL in the CRM at installation time; the CRM calls this URL for any changes to mapped fields.
2. **Attributes are one-directional (CRM → Apsis); consent is bidirectional** — this is a fundamental design principle.
3. **Sync conditions are evaluated on the Apsis side** — all CRM records are fetched, then filtered locally.
4. **FEC Enterprise allows multi-section installs; Maxo/Dynamics/Side Shop do not** — due to deep linking requirements.
5. **Phone number validation is silent** — invalid phone formats cause field updates to be skipped without a hard error; check Delta Sync Worker logs.
6. **Logging path**: CloudWatch → Delta Sync Manager (message production) → Delta Sync Worker (processing, validation, errors). Use integration key and CRM ID to filter.
7. **Full sync is required only for initial data import**; delta sync handles ongoing changes automatically.
8. **A Postman collection exists** for testing all generic connector endpoints — key resource for new developers.
9. **Test credentials are in LastPass** — search by connector name.
10. The team (Shreevidhya) noted: once you understand the architecture, adding new external systems via the generic connector is straightforward.

---

## Unresolved Questions / Action Items

- [ ] **Eric's session (next day)**: Deep dive into delta sync webhook registration internals, CloudWatch log navigation, query creation in FEC Enterprise, and complete architecture walkthrough.
- [ ] **Workshop request**: Lukasz requested a future workshop covering the scenario "new external system integration request — what do you do end to end?" — Shreevidhya agreed this would be valuable after the architecture session.
- [ ] **API key creation in FEC 12.1**: The team does not currently have permissions to create API keys/integrations in the FEC environment. If needed, contact the enterprise team.
- [ ] **`contains` sync condition operator**: Currently behind a feature flag pending external CRM support — status/timeline unclear.
- [ ] **Query creation in FEC Enterprise**: Not covered in this session; Shreevidhya has only used pre-existing queries. Eric to cover in follow-up.
- [ ] ⚠️ **Ambiguity**: The consent sync back to FEC Enterprise (opt-in from CRM → Apsis) was triggered during the session but the result in the Apsis consent timeline was not confirmed visible before the session moved on. It was expected to work based on prior testing but was not verified live.