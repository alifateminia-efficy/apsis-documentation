---
source_file: Shreevidhya - Efficy Enterprise 12.1.txt
domain: Apsis One Integrations
topics: [Efficy Enterprise 12.1 installation, generic connector setup, delta sync, full sync, sync conditions, field mappings, subscription mappings, consent bidirectionality, queries and profiles (tagging), CloudWatch log navigation, multi-account installation constraints, generic connector specification]
speakers: [Shreevidhya Ganesan (domain expert/KT lead), Lukasz Grabowski (learner, screen-sharer), Michal Rosikiewicz (learner), Tomasz Kowalski (learner)]
key_components: [Efficy Enterprise 12.1 (FEC Enterprise), generic connector, Delta Sync Manager, Delta Sync Worker, CloudWatch log groups, Apsis One audience/profiles, subscription mappings, sync conditions, queries and profiles (tagging feature), LastPass (credential storage), Postman collection]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

This session is a live walkthrough of installing and testing the **Efficy Enterprise 12.1 (FEC Enterprise 12.1)** integration using the **generic connector** in an Apsis One staging environment. Shreevidhya guides the team through creating a new section, installing the connector, configuring field mappings and subscription mappings, observing delta sync and full sync behavior, and using sync conditions to filter which profiles are synced. The session also covers how to navigate CloudWatch logs to debug sync issues, how the queries-and-profiles tagging feature works, and the constraints around multi-section installations for different connector types.

---

## Initial Setup: Creating a Section and Installing FEC Enterprise 12.1

- The team created a **new section** in Apsis One specifically for this installation. The naming convention recommended was `EE 12.1` so it is easy to identify which CRM is installed in which section.
- **Key constraint (discussed the previous day):** Only one CRM integration can be installed per section. This is a known limitation.
- Shreevidhya recommended having multiple sections so each can host a different CRM installation for exploration purposes.
- The installation environment (URL + API key) for FEC Enterprise 12.1 was provided by the CRM team. The team does not currently have permissions to create new API keys from within the CRM UI.

> [Michal Rosikiewicz]: "Can you show us where and how we can create this API key in FEC 12.1 CRM?"
> [Shreevidhya Ganesan]: "No, it was like the environment which was provided by the CRM, so we don't have those permissions to create a new integration."

- If needed, they can ask someone from the Enterprise team to clarify how API keys are created in the CRM.

---

## Field Mappings and Subscription Mappings

### Field Mappings
- After installation, the team navigated to **field mappings** and used the **auto-mapping** feature, then saved.
- Rather than doing a full sync first, the team opted to directly create a record in the external system to test delta sync behavior.

### Subscription Mappings
- A new **folder** (`EE 12.1`) and a **subscription** (`Profile QA`) were created in Apsis One for the purpose of mapping.
- In the integration's subscription mapping, `Profile QA` was mapped to the **email channel**.
- Important: a folder must exist before a subscription can be assigned to it; creating a folder alone is not sufficient — a subscription must be created and placed in that folder.

---

## Delta Sync: How It Works

### Trigger Mechanism
- When an installation is performed, the integration **registers a callback URL** (webhook) in the external CRM system.
- Whenever a change occurs in the CRM for a mapped field, the CRM triggers the callback URL with the **ID of the changed profile** as the payload.
- The fields sent during registration are exactly the fields that have been mapped in Apsis One — so the CRM knows which field changes should trigger the webhook.

> [Shreevidhya Ganesan]: "Whenever we do a registration, we will be sending the fields whatever we have mapped in our system to the external system, so that the external system will know that these are the fields which Apsis has mapped for. Any changes in that field, the webhook will be triggered."

- The webhook endpoint is handled by the **Delta Sync Manager** service.

### Timing
- Delta sync starts automatically as soon as the installation is done — there is no need to trigger it manually.
- For **Dynamics**, changes are reflected almost immediately. For **FEC Enterprise**, there is a delay of a few minutes because the CRM batches changes before sending them.

### Full Sync vs. Delta Sync
- **Full sync** is needed for the first-time pull of all existing records from the CRM (records created before the integration was installed, where no delta change has occurred).
- From that point on, delta sync keeps both systems in sync for ongoing changes.

---

## Inbound vs. Outbound Sync and Consent Bidirectionality

- **Inbound**: Changes in the CRM are picked up by delta sync and reflected in Apsis One profiles. This applies to **contact attributes**.
- **Outbound**: Changes made in Apsis One (e.g., a profile opts out) are pushed back to the external CRM system.
- **Attribute sync is one-directional (inbound only).** If a field value is changed in Apsis One directly, it will be **overwritten** by the next delta sync from the CRM.

> [Shreevidhya Ganesan]: "The attributes are not sent back... Only consent is bidirectional."

- **Consent sync is bidirectional:**
  - If a profile opts out in Apsis One → the opted-out status is pushed to the CRM.
  - If a profile is then re-opted-in from the CRM → Apsis One profile is updated to opted-in.
- The team demonstrated this live: unsubscribing a profile in Apsis One immediately updated the CRM record's opt-out status; re-opting-in from the CRM eventually updated the Apsis One consent timeline (with a short delay).

---

## CloudWatch Log Navigation

### Delta Sync Manager Logs
- Location: **CloudWatch → Log Groups → Delta Sync Manager**
- Contains logs up to the point where messages are produced and formatted for the worker to consume.
- Useful for confirming that a webhook was received and what payload fields were included (e.g., `webhook request parameters fields`).

### Delta Sync Worker Logs
- Location: **CloudWatch → Log Groups → Delta Sync Worker**
- Contains the actual processing logic, including validation errors.
- Example observed during the session: a phone number failed validation because the number entered was too short (`tried to validate and format phone number`). The field was skipped silently.

### How to Filter Logs
- Logs can be filtered by **integration key**, which contains:
  - Account ID
  - Section ID
  - Integration name (human-readable)
- The **message group ID** in the log context also encodes `account/section/integration_id` and includes the CRM profile ID (e.g., `179`).

> [Shreevidhya Ganesan]: "The integration key is something what you can always look in for when you look into the logs — you know which account, which section, and then followed by the message group ID, which gives you the integration ID along with the CRM ID."

---

## Sync Conditions

- Sync conditions allow filtering which profiles from the CRM are synced into Apsis One.
- Conditions are evaluated **on the Apsis side**, not in the CRM. The CRM still sends all changes; Apsis decides whether to process them.
- Conditions are evaluated as **AND logic** — all conditions must be satisfied for a profile to be synced.
- Applies to both **delta sync** and **full sync**.

### Demo
1. A sync condition was added: `email equals [lukasz's email]`.
2. A full sync was triggered. Result: **1 profile synced, 1 skipped** (the other profile did not match the condition).
3. Shreevidhya explained that adding a second condition (e.g., `name equals Mikael`) would cause even the matching profile to fail if the name didn't also match `Mikael`.

> [Shreevidhya Ganesan]: "For a profile to be synced, all the conditions should satisfy."

- **Note on "contains" operator:** A `contains` condition type exists but is behind a **feature flag** — it requires the external system to support the feature before it can be released.
- The sync condition was removed afterward to allow full sync to proceed without filtering.

---

## Queries and Profiles (Tagging Feature)

- **Queries and Profiles** is a feature distinct from full sync and delta sync.
- A **query** is a named list of profiles defined in the external CRM system (e.g., "profiles with VIP status").
- When this query is imported into Apsis One via the Queries and Profiles UI, the matching profiles receive a **tag** in Apsis One corresponding to the query name.
- This tag can then be used in **segmentation** within Apsis One.

> [Shreevidhya Ganesan]: "Say there will be a query in the external system where you can say the profiles which are in VIP status. When you import those profiles, you will have those profiles tagged as VIP, so that in segmentation it can become handy to filter out the profiles which have the tag VIP."

### Recurring Import
- If **recurring** is not checked, the import runs once.
- If **recurring** is checked, a **cron job** runs every morning to keep the tagged profile list in sync with the CRM query.

### Demo
- The `Apsis Test` query was used (an existing query in the CRM environment).
- Import was triggered immediately using **Start Import**.
- Result: 5 profiles processed; a new tag (`FS Enterprise`) was created in Apsis One. Profiles opened in Apsis One audience showed the tag but were missing first/last name because the CRM record used a single `name` field rather than separate first/last name fields.
- After removing the sync condition and running a full sync, the profiles were visible in Apsis One audience with email populated.

> **Note:** Shreevidhya mentioned she has not built queries from scratch herself — she has only used pre-existing queries in the test environment. Eric is expected to cover this in more depth in a following session.

---

## Multi-Section and Multi-Account Installation Constraints

- **FEC Enterprise (generic connector):** Can be installed in **multiple sections** across multiple accounts. There is no restriction on concurrent installations.
- **Maxo, Dynamics, Sideshop (legacy/specific connectors):** Can only be installed in **one section** — a single instance cannot be shared across multiple sections.

> [Shreevidhya Ganesan]: "If you see enterprise, you can have it on multiple sections, the same installation. But Maxo and Dynamics and Sideshop — we cannot have multiple integrations. One instance can be installed in only one section."

- The reason for the single-section restriction on those connectors is related to the **deep linking** feature: when you select a profile in Apsis One, it directly navigates to the corresponding profile in the external CRM. This requires a 1:1 mapping between section and CRM instance.

---

## Test Credentials and Environment Access

- Test environment API keys and URLs for most integrations (FEC Corporate, Tribe, etc.) are stored in **LastPass**.
- To find them, search LastPass by integration name (e.g., `FEC corporate`, `tribe`).
- For some environments, no frontend CRM link is available — only the API is accessible.
- Since all generic connectors share the same service code, testing one installation is sufficient to validate generic connector behavior.

---

## Generic Connector: Path for Adding New External Systems

- The team asked about the process for onboarding a new external CRM system.
- [Shreevidhya Ganesan]: The generic connector approach means any external system that wants to integrate must adhere to a **standard specification** (interface/API contract) defined by the Apsis integrations team.
- A **Postman collection** exists that covers all endpoints of the generic connector. New team members can use it to test features by supplying only:
  - The API key
  - The prefix URL of the external CRM

> [Shreevidhya Ganesan]: "The generic specification whatever we have developed is more standardized and straightforward. There is a Postman collection also — you can try all those endpoints by yourself, making a simple Postman request which needs only the API key and the prefixing URL of the external CRM."

- A deeper workshop on this topic (new connector onboarding workflow) was proposed for a future session, after the team has more familiarity with the architecture.

---

## Key Takeaways

1. **Generic connector (FEC Enterprise) vs. legacy connectors:** Generic connectors can be installed in multiple sections; legacy connectors (Maxo, Dynamics, Sideshop) are restricted to one section per instance due to deep linking.
2. **Delta sync is always running** from the moment of installation via a registered callback URL in the CRM. Full sync is only needed to backfill pre-existing records.
3. **Attribute sync is inbound only.** Any manual edits to attributes in Apsis One will be overwritten by the next delta sync from the CRM. Only **consent** is bidirectional.
4. **Sync conditions are AND-logic, Apsis-side filters.** The CRM sends all changes; Apsis decides whether to process each one based on whether all conditions are met.
5. **Phone number validation** can silently skip a field if the value does not meet the expected format (e.g., too short). Check **Delta Sync Worker** logs to find these skipped-field warnings.
6. **Queries and Profiles** is a tagging mechanism: CRM-defined lists of profiles get imported as tagged profiles in Apsis One, enabling segmentation by CRM-side criteria.
7. **Log navigation:** Use **integration key** (account/section/integration) to filter in CloudWatch. Delta Sync Manager logs cover message production; Delta Sync Worker logs cover processing and validation.
8. **Test credentials** for all integrations are in LastPass; search by integration name.
9. A **Postman collection** exists for the generic connector specification — recommended as a self-service exploration tool for new developers.

---

## Unresolved Questions / Action Items

- **API key creation in FEC 12.1 CRM UI:** The team does not have permissions to create API keys. Follow up with the Enterprise CRM team to understand the process.
- **Query creation from scratch in FEC Enterprise:** Shreevidhya has only used pre-existing queries. Eric is expected to cover query creation in the next session.
- **Deep dive on Delta Sync Manager and Worker internals:** Noted as Eric's topic for the following session.
- **Workshop on onboarding a new external CRM:** Requested by Lukasz. Proposed as a future session after the team has completed the architecture review with Eric.
- **"Contains" sync condition operator:** Currently behind a feature flag pending external system support. Status of rollout is unresolved.
- **Consent sync update (re-opt-in from CRM):** During the demo, the consent timeline in Apsis One had not yet updated by the end of the session. Assumed to be a timing/batching delay — not confirmed as working within the session.