---
source_file: Erik - Handling Sync Conditions in Efficy Enterprise.txt
domain: Apsis One Integrations
topics: [sync conditions, full sync optimization, contact filtering, consent sync, pagination, stateless vs stateful API design, webhook filtering, query parameter design]
speakers: ["Erik Andersson (Apsis, integration/backend engineer)", "Raziel Carvajal (Efficy Enterprise, integration engineer)", "Lukasz Grabowski (Efficy Enterprise, integration engineer/lead)"]
key_components: [Apsis One, Efficy Enterprise (FSC Enterprise), sync conditions, full sync, webhooks, get records endpoint, consent endpoint, Dynamics connector (Side Shop), Maxo connector]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson walks Raziel Carvajal and Lukasz Grabowski through a critical performance problem caused by how sync conditions are currently evaluated in the Apsis–Efficy Enterprise integration. At present, all contacts and consents must be downloaded from the CRM before sync conditions can be evaluated on the Apsis side — a design that breaks catastrophically at scale (8M+ contacts). Erik presents a solution already implemented for another CRM connector (Dynamics/"Side Shop") and proposes adapting it for Efficy Enterprise: pushing sync condition parameters into the `get records` and consent API requests so that Enterprise pre-filters results server-side before returning them. The session covers the tradeoffs between stateless (query parameter) and stateful (configuration/cache) approaches, how pagination interacts with filtered requests, and the scope of changes needed on both sides.

---

## Background: How Sync Conditions Currently Work

**Sync conditions** are rules configured inside Apsis that determine which contacts from a connected CRM should be synced into Apsis. Example: "only sync contacts where `birthplace = Sweden`."

- Sync conditions are evaluated on the Apsis side.
- This applies to both **full sync** and **real-time sync** (webhooks).
- On every sync — full or incremental — Apsis downloads **all** contacts from the CRM, evaluates the conditions, and discards non-matching contacts.
- This design was intentional from the start: because Apsis integrates with CRM systems it does not control, all filtering logic had to live inside Apsis.

> "This has worked quite well up until quite recently." — Erik Andersson

---

## The Scale Problem: 8 Million Contact Customer in France

### Scope of the Issue

A customer (in France) has approximately **8 million contacts** in their CRM and wants **22 installations** in Apsis to handle different sections/regions/groups.

- Target synced contacts per installation: ~100,000
- Contacts that must be downloaded just to evaluate conditions: 8,000,000
- Discarded contacts per sync: ~7,900,000
- Consents: at minimum 1 per contact → ~8,000,000 consents, but typically more (~16M at 2 per contact)
- Multiply by 22 installations: **~416 million consent records** downloaded per full sync cycle across all installations

### What Actually Happened

Erik confirmed the initial full sync was attempted:

- Contact download completed without the sync failing.
- Consent sync triggered a crash due to a memory optimization inside Apsis that cannot handle this volume.
- Even with that optimization removed, the **FSC Enterprise instance timed out after approximately 1.5 days** while the consent sync was ongoing.
- A **gateway timeout** was observed. There is also a proxy layer in the infrastructure that may terminate requests even earlier than the business logic timeout.

> "Even if it does go through, this sync will take approximately a day for one sync." — Erik Andersson

### Existing Impact on Smaller Customers

Raziel confirmed this is not limited to the 8M-contact case:

> "Even with 1 million it's been an issue — sometimes the disconnection takes place, clients report that the CRM is slow, and when you check the logs there's a sync conversation taking place." — Raziel Carvajal

Three customers are currently affected.

---

## Existing Solution: The Dynamics / "Side Shop" Connector Precedent

For the **Dynamics connector** ("Side Shop"), a different architecture was implemented:

- Sync conditions are still **configured inside Apsis** (keeping all config in one place).
- Apsis **sends the sync conditions to the CRM connector** as a configuration setting.
- The CRM connector stores these conditions and pre-filters data before returning it to Apsis — either via a pre-generated cache or on-demand query execution (Erik noted he doesn't know the exact internal mechanism).
- Result: Apsis only receives the ~100,000 matching contacts; full sync completes in **5–10 minutes**.
- For **webhooks**, Side Shop also filters outbound webhook events to only send records matching sync conditions — though Erik noted this is less urgent since Apsis can still filter incoming webhooks on its side.

A similar "preflight" pattern was also implemented for **Maxo** (expected ~5 million contact IDs sent in ~6 seconds): Apsis sent a preflight request to the CRM saying "we are about to sync, please prepare your data," and the CRM ran its SQL queries and had a cache ready.

---

## Proposed Solution for Efficy Enterprise: Sync Conditions as Query Parameters

### Core Proposal

Rather than storing sync conditions as configuration state on the Enterprise side, the proposed approach is to pass sync conditions as **query parameters on the existing `get records` API request**. This keeps the Enterprise connector stateless.

Example of what Apsis would send:

- The existing `page` and `page_size` parameters (unchanged)
- A new set of filter parameters, e.g.:

```
first_name=Erik&country=Sweden
```

Where the field names are the **FSC Enterprise attribute names** that Apsis already has access to.

Enterprise would apply these as filter conditions (equivalent to a `WHERE` clause) and return only matching records.

### How Pagination Still Works

- Apsis downloads pages in parallel (page 0, 1, 2, 3… across multiple threads).
- Apsis sends one more request than strictly necessary — when a page returns empty, the sync is considered complete.
- With filtering applied, the number of non-empty pages drops dramatically, but the pagination mechanism itself is unchanged.
- The `page` and `page_size` parameters must still be included in filtered requests because: (a) Apsis has no prior knowledge of how many matching records exist, and (b) the result set may still be large enough to require pagination.

### Stateless vs. Stateful: The Design Discussion

Raziel expressed a preference for keeping the Enterprise connector **stateless** — i.e., no config stored on the Enterprise side, consistent with how the connector has been designed so far:

> "We have treated all these requests as stateless. We don't keep any state. As soon as we receive the request, we just attend it and return the answer without keeping any cache — we might have concurrency issues or inconsistencies." — Raziel Carvajal

Erik acknowledged the tradeoff:

- **Stateful (config-based, like Side Shop):** Fewer total requests, but requires storing state on the Enterprise side.
- **Stateless (query params per request):** Same number of requests as today, but each request responds much faster because Enterprise only processes matching records. Net result is still a large reduction in total sync time.

Both agreed the stateless query-parameter approach is acceptable and simpler to implement without changing the architectural model of the connector.

### Sync Condition Logic: AND Only (No OR Support Yet)

Raziel asked whether conditions would be treated as a union (`OR`) or intersection (`AND`). Erik clarified:

> "We don't support OR today. Everything is AND. It's a weakness in the sync condition design." — Erik Andersson

Context: A **new version of sync conditions** supporting `OR` logic was developed and implemented on the Apsis backend, but was **never shipped to customers** because front-end resources were lost and then a company reorg occurred. This feature exists in code but is not active for any customer. For the purposes of this implementation, all conditions should be treated as `AND`.

---

## Consent Sync: Applying the Same Approach

The same query-parameter filter approach should be applied to the **consent endpoint**, not just the `get records` endpoint.

### Current State of the Consent Endpoint

- The consent endpoint currently does not accept field/filter parameters — it returns all consents.
- You cannot currently request consents for specific contact IDs.
- A list of contact IDs as a query parameter is not viable at scale (parameter would be millions of entries long).

### Proposed Change

Apply the **same sync condition query parameters** to the consent endpoint that are applied to `get records`. Enterprise would return only consents belonging to contacts that match the sync conditions.

This creates a **unified filtering flow** for both records and consents, which is strongly preferred from the Apsis side because Apsis processes everything as messages without separating contacts from consents internally.

> "It would be very preferable for us if we can do it that way — then we have one unified flow for both records and consents, and our sync process will reduce the number of messages radically." — Erik Andersson

### Webhooks and Consents

Raziel noted that the consent webhook issue is less acute:

- Webhooks are triggered by actual changes in the CRM, so payloads are small.
- The scenario where a CRM updates thousands of consents in minutes is unlikely.
- The full sync is the primary problem for consents, not webhooks.

Erik agreed: webhooks can continue to be filtered on the Apsis side for now. The immediate priority is the full sync.

---

## API Contract Changes Required

The following changes need to be designed and agreed upon:

1. **`get records` endpoint**: Add sync condition filter query parameters (field name + value pairs, AND logic). Keep existing `page` and `page_size` parameters.
2. **Consent endpoint**: Add the same sync condition filter query parameters.

### Who Designs the Contract

Erik proposed that **Apsis will define the query parameter names and format** (since Apsis controls the integration spec), and Efficy Enterprise will review and implement. Raziel agreed — this is standard practice: Apsis updates the API spec with examples, Efficy asks questions if needed, then implements.

### Known Constraints

- Apsis currently has a **maximum of 10 sync conditions** per section (Erik noted this from memory — ⚠️ should be verified).
- Typical customers use only 2–3 conditions.
- URL length limits are not expected to be an issue given the above, but should be confirmed.

---

## Current Sync Condition Feature Gap: OR Logic

- A v2 sync conditions feature supporting `OR`/`AND` grouping was built on the Apsis backend.
- It was **never released** due to loss of front-end resources and a company reorg.
- No customers currently use OR logic — all existing sync conditions are purely AND.
- This is a known weakness but does not affect the current implementation effort.

---

## Urgency and Prioritization

Erik stated that **three customers are currently suffering** from this issue and that he has already flagged it to Apsis product management.

Raziel offered a nuanced view on urgency:

> "If the client does the synchronization at night and it takes the whole night — in the morning they have synced data, so it's not rather urgent because the system works. It is urgent if the clients run it in the middle of the day when they are working and everything slows down." — Raziel Carvajal

Lukasz committed to writing an epic to document this improvement, using the session recording, and sending it to Erik for review. He noted the goal is to implement this before Erik's departure from the project.

---

## Key Takeaways

1. **The root cause** of sync performance problems is that Apsis currently must download 100% of CRM data just to evaluate sync conditions that should ideally be applied at the source.
2. **The proven solution** (already live with Dynamics/Side Shop) is to push sync conditions to the CRM connector so it pre-filters results. This reduced sync time from ~1.5 days to ~5–10 minutes in the Side Shop case.
3. **For Efficy Enterprise**, the agreed direction is a **stateless, query-parameter approach**: sync conditions are passed as filter parameters on each `get records` and consent API request, keeping the Enterprise connector stateless.
4. **Both the `get records` and consent endpoints** need to be updated with the same filter parameters to create a unified sync flow.
5. **Sync conditions are currently AND-only**; OR logic exists in Apsis backend code but has never been shipped to customers.
6. **Webhooks are lower priority** for this change — Apsis can continue filtering webhook payloads on its side. The full sync is the critical path.
7. **API contract design** is Apsis's responsibility; Efficy Enterprise will review and implement once the spec is updated.

---

## Unresolved Questions and Action Items

- [ ] **Lukasz**: Write an epic describing this improvement based on session recording; send to Erik for review.
- [ ] **Erik / Apsis**: Update the API spec with new query parameter names and format for sync condition filtering on `get records` and consent endpoints.
- [ ] **Efficy Enterprise team**: Implement filter support on `get records` and consent endpoints once spec is finalized.
- [ ] **⚠️ Verify**: Maximum number of sync conditions in Apsis (Erik said "10 if memory serves" — needs confirmation before finalizing URL parameter design).
- [ ] **Open question**: For consents specifically — is the query-parameter filter approach (filtering by contact-level sync conditions) sufficient, or is a different mechanism needed? The group agreed in principle but did not finalize the consent endpoint design.
- [ ] **Open question**: Should webhooks eventually also be filtered at the Enterprise side (sending only matching records)? Deferred as non-urgent — Apsis can handle webhook filtering internally for now.
- [ ] **Prioritization**: Three customers currently affected. Urgency depends on whether affected customers are running syncs during business hours. Lukasz to assess sprint prioritization.