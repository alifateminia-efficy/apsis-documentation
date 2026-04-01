---
source_file: "Erik - Sync conditions filtering.txt"
domain: Apsis One Integrations
topics: [sync conditions filtering, FSC Enterprise integration, generic connector specification, full sync optimization, query parameters, installer options, delta sync, webhook handling]
speakers: ["Erik Andersson (appears to be architect/senior engineer)", "Lukasz Grabowski (engineer)", "Michal Rosikiewicz (engineer, minimal contribution)"]
key_components: [FSC Enterprise, Generic Connector, Delta Sync Manager, Full Sync Producer, Mock Service, Microsoft Dynamics, Generic Connector Spec]
session_type: knowledge-transfer
---

## Session Overview

This session covers a planned architectural change to push sync condition filtering upstream to FSC Enterprise, rather than evaluating conditions inside APSIS. The core problem is that APSIS currently downloads all contacts from CRM systems before filtering — a process that is crashing full syncs and CRM systems when customers have millions of contacts. The team discusses the implementation approach (query parameters vs. request body), the required changes to the Generic Connector spec and codebase, a new installer option flag, the correct release sequencing, and the boundary between full sync and real-time/delta sync behavior.

---

## Problem Statement: Sync Condition Evaluation Inside APSIS

Currently, **sync conditions** are evaluated inside APSIS. This requires downloading every contact from the CRM system first, then filtering locally.

This has become a critical issue with large customers:

> "We have French customers that have 8 million contacts in their CRM system and this is not working with the way it is set up because we need to download 8 million contacts and then at least 8 million consents and then throw away 99% of it. It's like it's ridiculous, and it is crashing APSIS full syncs and it is crashing the CRM systems."

The goal is that **APSIS should never receive contacts that don't match the sync conditions**. To achieve this, the CRM system must be made aware of the conditions at query time and filter server-side.

---

## Existing Solution in Microsoft Dynamics (Stateful Pattern) vs. FSC Enterprise (Stateless)

### Microsoft Dynamics / By-Site-Shop — Stateful Implementation (Working)

- The sync condition is stored locally on the connector side.
- APSIS sends the sync condition to the connector.
- When APSIS requests records during a full sync, the connector returns only matching records (pre-cached or paginated).
- Webhooks also only fire for contacts that match the sync condition.

### FSC Enterprise — Stateless API (Problem to Solve)

- FSC Enterprise does **not** have a stateful API.
- There is no stored sync condition on their side.
- The fix requires: on every `GET` request for records, include the sync conditions so FSC Enterprise can apply them as a server-side SQL-style filter and return only matching contacts.

> "We need to do some minor tweaks here. When we make a request to the CRM system, we should include the sync conditions as a query parameter or in the request body."

---

## Design Decision: Query Parameters vs. Request Body for Sync Conditions

### Option A: Query Parameters (Preferred)

Send sync conditions as query parameters on the `GET` request.

Example conceptual format:
```
GET /records?entity=contacts&fields=...&sync_condition=first_name=Erik&sync_condition=country=France
```

**Concerns raised:**
- URL length limit — standard is approximately 2000 characters.
- Data encoding required to make conditions URL-safe.
- Nested/complex syntax looks unusual but is syntactically legal.

**Why this is likely fine:**
- Typical sync conditions are only 2 conditions (e.g., `contact is active` and `country equals something`).
- The UI can be used to enforce limits on the number of conditions if needed.

### Option B: Raw Attribute Names as Separate Query Parameters

Pass the CRM's own attribute names directly as individual query parameters (e.g., `?first_name=Erik&last_name=Andersson`). This is more idiomatic REST but deviates from the **Generic Connector standard**.

> [Erik]: "This kind of goes a bit outside of the generic connector standard... this would be more generic-connector-y if we put it that way."

**Decision: Option A (sync_condition list as query parameter) was preferred** by Lukasz and Erik because it is more explicit and consistent with the Generic Connector abstraction.

### Request Body (Rejected)

Including filter parameters in the body of a `GET` request was explicitly rejected as non-RESTful and non-standard.

> [Lukasz]: "The request body — forget it, it's not RESTful, right?"

### Current Sync Conditions Assumption: AND-only Logic

> [Erik]: "We are doing this now with the old sync conditions in mind where everything is AND, so you can just create one big list of every sync condition because it's implicit that this is AND. OR doesn't exist in this world yet."

This simplifies the query parameter encoding for now.

---

## Generic Connector: Affected Endpoints

Two endpoints in the Generic Connector need to be extended to include sync conditions as query parameters:

1. **`get_records`** — used during full sync to retrieve contacts/entities.
2. **`get_consents`** — used to retrieve consent records.

[Erik]: "The records `get_records` is one and then the `get_consents` is the other."

These are the only two endpoints requiring changes on the APSIS side.

---

## New Installer Option Flag: `supports_sync_conditions_as_query_param`

A new **installer option** must be added to the Generic Connector configuration for FSC Enterprise. This flag gates the behavior so it is not accidentally applied to CRM systems that don't support it.

```yaml
supports_sync_conditions_as_query_param: true
```

**Behavior when `true`:**
1. Include sync conditions as query parameters on `get_records` and `get_consents` calls.
2. **Disable local condition filtering logic in the Full Sync Producer** — since the CRM is now responsible for filtering, APSIS does not need to re-verify conditions on received contacts during full sync.

**⚠️ Important caveat — real-time / webhook path:**
The local condition filtering logic must **not** be removed from the **Delta Sync Manager** (real-time sync path), at least in the first step.

> [Erik]: "For enterprise they will still send webhooks for everything. In the first step you need to keep that in the real-time sync. But in the full sync you don't need to have it, and it's in the full sync where you will have the savings."

Rationale: FSC Enterprise has no awareness of sync conditions at webhook-send time — they will still fire webhooks for all contact updates. APSIS must continue to validate those in the Delta Sync Manager.

---

## Risk: Silent Data Loss / Missing Contacts

Erik flagged this as a critical testing concern:

> "This is a very easy pitfall to miss out on contacts and APSIS will have no idea that this is happening because we will just get contact IDs. If things are missing from the CRM, we will be blind for it, so this needs to be heavily tested."

The specific risk is incorrect pagination or filtering on the FSC Enterprise side causing contacts to be silently omitted from the full sync. APSIS cannot detect this on its own.

Duplicate contacts (as was a known issue with Tribe integration) are also a risk pattern to watch for.

---

## Implementation Steps and Release Sequencing

The correct order of operations is:

1. **Extend the Generic Connector specification** — this is the shared source of truth for both APSIS and the CRM connector team (FSC Enterprise / Raisel's team). Changes to spec come first.

2. **Share spec with FSC Enterprise team (Raisel)** — they implement support for accepting sync conditions as query parameters on their endpoint, running them as SQL-style filters, and returning only matching records. Pagination must remain correct after filtering.

3. **Implement on APSIS side** — add sync conditions to `get_records` and `get_consents` calls; add `supports_sync_conditions_as_query_param` installer option; conditionally skip full sync condition verification when flag is true.

4. **Test using the Mock Service** — add support for sync condition query parameters in the mock service so the APSIS-side change can be tested independently on staging without requiring FSC Enterprise to release first.

5. **FSC Enterprise releases their change** — only after their production endpoint supports the parameters should the APSIS change be merged to master and released. Once released, APSIS will begin sending sync conditions and FSC Enterprise will filter server-side.

> [Erik]: "First always the general connector spec because that's the shared source of truth for both you and the CRM. You implement it on your side, you test it with the mock service, they enable it, you add this and release it."

**Note on early release:** Lukasz raised the possibility of releasing the APSIS-side change before FSC Enterprise is ready (since sending extra query parameters they ignore would be harmless). However, this was noted as only acceptable if the local filtering logic is **not** simultaneously removed — otherwise contacts could be missed during the transition window.

---

## Estimated Effort

> [Erik]: "The absolute heaviest work will be on the Enterprise side. The APSIS-side work is not that much — I think honestly you can solve this in like 2 days tops. Then it is a collaborative task to make sure that you are still getting the correct amount of profiles."

---

## Key Takeaways

1. **Root cause of full sync crashes:** Sync conditions evaluated inside APSIS force downloading all CRM contacts (up to 8M) before filtering — this must move server-side.
2. **FSC Enterprise is stateless:** Unlike Microsoft Dynamics, it cannot store sync conditions. They must be passed on every request as query parameters.
3. **Chosen approach:** Sync conditions sent as a query parameter list (AND logic only for now) on `get_records` and `get_consents` calls.
4. **New installer flag** `supports_sync_conditions_as_query_param` gates this behavior per connector.
5. **Full sync filtering logic can be disabled** when the flag is true; **real-time/webhook path (Delta Sync Manager) must retain filtering logic** in step one, because FSC Enterprise webhooks are not condition-aware.
6. **Silent data loss is the primary testing risk** — if FSC Enterprise pagination breaks under filtering, APSIS will not detect missing contacts.
7. **Release order is strict:** Spec → FSC Enterprise implementation → APSIS implementation + mock service testing → FSC Enterprise production release → APSIS production release.

---

## Unresolved Questions / Action Items

- [ ] **Erik** to send notes/steps from VS Code chat to Lukasz and Michal as reference.
- [ ] **Lukasz** to create stories based on the steps discussed, referencing Erik's notes.
- [ ] **Lukasz/team** to extend the Generic Connector specification to include sync conditions query parameter contract.
- [ ] **Specification to be shared with Raisel (FSC Enterprise team)** for their implementation.
- [ ] **Mock service** needs to be updated to accept and handle sync condition query parameters for staging testing.
- [ ] **Clarify the exact query parameter format/encoding** to be defined in the spec — the session discussed the concept but did not finalize exact parameter naming or encoding scheme.
- [ ] **OR logic support** is out of scope for now but will need revisiting when the sync condition model is extended beyond AND-only.
- [ ] **Webhook / real-time filtering removal** from Delta Sync Manager is explicitly deferred to a later step — needs a follow-up decision on when/how to handle it.