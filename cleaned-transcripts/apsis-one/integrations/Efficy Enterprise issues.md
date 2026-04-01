---
source_file: Efficy Enterprise issues.txt
domain: Apsis One Integrations
topics: [full sync performance, sync conditions, consent synchronization, on-premise installations, generic connector, CRM-side filtering, memory optimization, support process]
speakers: ["Erik Andersson (Senior Developer/Integration SME)", "Lukasz Grabowski", "Tomasz Kowalski", "Michal Rosikiewicz"]
key_components: [FSC Enterprise, Generic Connector, Apsis One, SQS, Audience export, sync conditions endpoint, consent mappings, squid proxy, Dynamics/SiteShop connector]
session_type: knowledge-transfer
---

## Session Overview

This session was initiated to investigate why full syncs for large Efficy/FSC Enterprise customers are taking excessive time or failing outright. Erik Andersson walks through the root causes affecting customers with very large datasets (up to 8 million contacts), focusing on two compounding problems: sync conditions being enforced on the Apsis side rather than the CRM side, and consent pagination causing gateway timeouts from the CRM system. The session also covers an existing architectural solution (CRM-side sync condition filtering, already implemented for Dynamics/SiteShop) that needs to be extended to FSC Enterprise. A secondary discussion covers an in-memory audience export optimization that will become a memory risk at large scale and should be removed.

---

## Identifying Which Integration a Customer Uses

To determine which integration a specific customer has installed:

1. Take the customer's display name and look up their account ID in **Back Office** (the display name alone is not sufficient).
2. Query the **`installations` table** in the database — this is the canonical source of truth for which integration is installed on which section/account. This is also the data source used by the integration report store.

```sql
SELECT * FROM installations WHERE id = '<account_id>';
```

In the example discussed, the customer was found to have an **FSC Enterprise 2** installation.

> "In installations we have a complete repository of which integration is installed on which section and account."

---

## On-Premise FSC Enterprise Installations: Known Limitations and Access Requirements

From the integration side, there is **no way to tell** whether a customer's FSC Enterprise environment is on-premise or cloud-managed. This distinction does not affect integration behavior technically, but on-premise setups frequently cause issues in practice.

**Hard requirement:** Apsis integrations require **public internet access** to the CRM environment. VPN tunnels are not supported.

> "We need to have public access to the environments. If you are to use Apsis, we don't support VPN tunnel like we do in Pro, because we don't have anyone to maintain or support them."

Common on-premise issues:
- IP whitelisting blocking Apsis requests (whitelisting is handled at installation time via the squid proxy)
- Heavy customer customizations causing unrelated failures

**Diagnostic approach for on-premise issues:** The FSC Enterprise team took a copy of the customer's database, ran it against a managed instance, and the connector worked fine — confirming the issue was environmental/customization-related, not an Apsis integration bug.

---

## Full Sync Architecture: How Sync Conditions Work Today (and the Core Problem)

### How Sync Conditions Are Currently Enforced

**Sync conditions** allow configuring a filter so that only contacts matching a condition (e.g., email equals a specific value) are synced to Apsis. These conditions are configured inside the Apsis One UI under the integration settings.

**Critical design implication:** Even when sync conditions are configured, the filtering happens **on the Apsis side**. Apsis must download the entire dataset from the CRM and then discard records that don't match.

For a customer with 8 million contacts where only 1.2 million should be synced across several sections (each capped at ~100,000 contacts):

- The full sync still downloads all **8 million contact records** from the CRM
- Then downloads all **8 million consent entries** (assuming ~1 consent per contact; customers can have up to 20 consent lists mapped)
- Total data processed: **~16 million+ records**, of which the vast majority are discarded

> "We have to download all of the data and while it seems to be working quite OK for the contact data, it is for sure not working for the consent."

### Why the System Was Designed This Way

The decision to keep all integration configuration (field mappings, sync conditions, consent mappings) inside Apsis One was deliberate — everything regarding the integration is configured within Apsis. During the development of Maxo, there was significant back-and-forth; the ideal would have been to configure field mappings and sync conditions inside the CRM (where richer relational data is available), but no one prioritized implementing the UI on the CRM side. The current architecture is a compromise: **configuration lives in Apsis, enforcement is intended to move to the CRM side** (but this has not yet been implemented for FSC Enterprise).

### Limitations of Apsis-Side Sync Conditions

Because Apsis only has access to the flat contact data it receives, some filtering use cases are **impossible** to implement via Apsis sync conditions:

- "Only sync contacts belonging to a company with revenue > 500,000 SEK/year" — requires related entity (organizational) data not available in Apsis
- "Only sync contacts that have at least one consent" — a very common customer request, also not possible in Apsis today

Both of these would be trivially implementable as CRM-side filters.

---

## The Consent Timeout Problem (Immediate Breaking Issue)

### Symptoms

For the affected customer (8 million contacts):
- Contact data downloads completed (paginated, ~16,000 pages fetched) — slow but functional
- **Consent downloads are failing with HTTP 504 Gateway Timeout** responses
- The 504 is returned as an HTML response, indicating it originates from a **proxy server** that FSC has in front of their business logic
- A manual test request for 500 consent entries took **22 seconds**

### Scale of the Problem

```
8,000,000 consents / 500 per page × 22 seconds = ~97 hours to download all consents
```

The sync ran for **over 8 hours** and was still hitting timeouts, having made no progress past the consent download phase. Contacts are visible in Apsis One but **consents are not added** because the CRM keeps timing out.

[Erik Andersson]: "They have some weird implementation on the pagination, so they are taking an incredibly long time to give a reply."

### What Apsis Can and Cannot Do

Apsis **cannot fix** the gateway timeouts — the CRM system's pagination performance is the bottleneck. What Apsis can do:
- Surface the evidence (504 responses, timing data) to set expectations
- Raise the issue with the FSC Enterprise team for pagination/caching optimization
- Implement the CRM-side sync condition filtering feature for FSC Enterprise (see below)

---

## Existing Solution: CRM-Side Sync Condition Filtering (Already Built for Dynamics/SiteShop)

### How It Works (Dynamics/SiteShop Reference Implementation)

For the **Dynamics/SiteShop** integration (which uses the **Generic Connector**), a feature was built where:

1. When sync conditions are configured in Apsis, Apsis makes a request to a dedicated endpoint on the CRM side, passing the sync conditions.
2. The CRM stores these conditions and applies them server-side.
3. When Apsis requests contacts during a full sync, the CRM returns **only the pre-filtered contacts** — Apsis never sees the records that don't match.

Result: Instead of downloading 1,000,000 contacts to get 1, only 1 contact is returned. Full sync performance improvement is dramatic.

The same filtered logic must also apply to **webhook requests** — the CRM should only send webhooks for contacts that fulfill the sync conditions.

### Applying This to FSC Enterprise

**FSC Enterprise 12.1 uses the Generic Connector.** The Apsis-side change required is minimal:

In the Generic Connector installer config, set:
```
connector_supports_sync_condition: true
```

This is already built and working. However, **it will not function unless the CRM exposes the required endpoint**.

### The Sync Condition Endpoint Contract

The endpoint receives a payload structured as:

```json
{
  "section": "<section_id>",
  "entity": "<entity_type e.g. contact>",
  "send_delete_if_unmatched": true/false,
  "conditions": [
    {
      "crm_field_name": "<field name in CRM e.g. first_name>",
      "operator": "<condition operator>",
      "value": "<value e.g. Elsa>"
    },
    {
      "crm_field_name": "sync_to_apsis",
      "operator": "equals",
      "value": "true"
    }
  ]
}
```

- `send_delete_if_unmatched`: toggleable in Back Office — determines whether the CRM should send a delete request to Apsis when a contact no longer matches the sync conditions
- Conditions are registered **per section**
- Typically only applicable for the main entity (contact) in FSC Enterprise
- Must also apply to consents: the CRM should only return consent entries belonging to contacts that fulfill the sync conditions

> "They need to expose an endpoint so we can send it to them, and they also need to implement the filtering logic — when we make the full sync request, they should only give us contacts that we should have, and they should only make webhook requests for contacts that fulfill the sync conditions."

[Erik Andersson on implementation complexity for the CRM side]: "Handling this in an efficient way is not trivial because you need some kind of ready cache, or however you handle it — you can't really do this in real time. Maybe incorporate it into an SQL statement. I don't know how they implement it best, but it can be complex."

---

## Secondary Issue: In-Memory Audience Export Optimization (Future Memory Risk)

### Current Optimization

As one step in the full sync, Apsis performs an **export from Audience** of all existing subscriptions that have a consent mapping. This export is held **in memory** and used for comparison:

- If a consent entry from the CRM differs from the current Apsis state → generate a consent update message → put it on **SQS** for the full sync consumer
- If it matches → skip (no update needed)

This is an optimization to avoid unnecessary consent update processing.

### Why This Will Become a Problem

For a customer with 7–8 million profiles each having multiple consent entries, loading the full Audience export into memory will likely **crash the sync with an out-of-memory error**.

Note: For the current affected customer this is **not the breaking issue right now** (the consent download fails before this step is reached, so the existing consent matrix is empty). But if the CRM-side pagination is fixed and a subsequent full sync runs with actual existing consent data in Apsis, this memory issue will surface.

### Recommended Fix

Remove the in-memory Audience export comparison optimization and **process all consent data streamed from the CRM directly**. The CRM data is already handled in a streaming fashion (downloaded pages are put on SQS immediately rather than held in memory); the Audience export comparison is the outlier.

> "I probably think we should remove it and just process everything from the CRM system."

This is a **proactive fix** — not urgent for the current broken state, but should be addressed before a customer of this scale successfully completes a full sync and triggers the memory issue.

---

## Scale Testing History and Current Gap

Previous load testing was conducted with **2 million contacts**. At the time, the largest customer had approximately 1 million entries — syncs were slow but completed successfully. The current affected customer has **8 million contacts** — four times the tested limit. No load testing has been performed at this scale.

> "Obviously things will get very interesting from this."

**⚠ Warning:** The 2-million-contact load test results should not be used to make guarantees about larger datasets. Documentation of platform limits should be updated (currently not documented anywhere).

---

## Affected Customers and Triage Status

Three customers were flagged in the product help channel as affected by slow/failing full syncs:

1. **On-premise customer** — Issues are primarily due to their custom on-premise setup, not Apsis. The FSC Enterprise team is working with them. Do not investigate from the integration side until the team confirms it is an integration issue.
2. **Large customer (~8 million contacts)** — Root cause identified as described above (consent pagination timeouts + no CRM-side filtering). Active issue.
3. **Third customer** — Also related to consents; a pattern was identified connecting this to the above issue.

---

## Support Process and Escalation Path

[Erik Andersson - strong recommendation]:

> "No developer is expected to sit and follow the product help channel. Only handle things that come through the proper support channel (Jira/helpdesk). Otherwise, essentially ignore it — we can't track stories from product help channels, internal chat, emails, and the help channel simultaneously."

- Do **not** respond to direct messages or product help channel posts as a substitute for support tickets
- All support queries should be routed through the official help channel/support system
- R&D responding ad hoc sets a precedent and leads to escalation abuse
- If this creates friction, that friction is intentional — it will surface the need for a properly scaled support process

**Recommended escalation path for this specific issue:**
1. Erik to present the issue to PS (customer success / account management side) with supporting evidence (504 logs, timing data, scale numbers)
2. PS raises with product team for a cross-organizational meeting
3. Product team to drive the conversation with the FSC Enterprise CRM team to prioritize the sync condition endpoint implementation
4. This is above R&D's pay grade to drive directly — it requires organizational pressure around cost and stability

**Framing for CRM team negotiation:**
- The CRM system itself is being harmed by the current approach (it is generating the timeouts)
- Implementing CRM-side filtering reduces stress on **their** system, reduces sync time for customers, and reduces Apsis compute/data transfer costs
- A full sync that currently takes 1.5+ days could complete in ~15 minutes for 100,000 filtered contacts

---

## Key Takeaways

1. **Sync conditions are enforced on Apsis side today** — Apsis downloads the full CRM dataset and filters locally. For customers with millions of contacts, this is unsustainable.

2. **The immediate breaking issue** is FSC Enterprise's consent pagination performance — 22 seconds per 500 records causes gateway timeouts at scale. This is a CRM-side problem.

3. **A working solution already exists** in the Generic Connector (CRM-side sync condition filtering, as implemented for Dynamics/SiteShop). FSC Enterprise needs to implement the corresponding endpoint and filtering logic. The Apsis config change is a one-liner (`connector_supports_sync_condition: true`).

4. **A secondary future risk**: the in-memory Audience export comparison in the full sync consent step will likely cause memory crashes at 7–8 million consent scale. The optimization should be removed.

5. **3 customers are currently affected**. The trend is toward larger datasets. The current architecture will not scale without the CRM-side filtering fix.

6. **Load testing was done up to 2 million contacts only.** Platform limits are undocumented and should be formally recorded.

7. **Support must flow through official channels.** R&D should not be responding to product help channel posts or direct messages as a primary support mechanism.

---

## Unresolved Questions / Action Items

- [ ] **Erik** to present the issue to PS with evidence, including the 504 logs and timing analysis
- [ ] **Erik** to raise the issue in the FSC Enterprise channel to see if quick pagination/caching optimizations are feasible on the CRM side
- [ ] **Lukasz** to be included in the PS discussion if desired
- [ ] **Product team** to organize cross-organizational meeting with FSC Enterprise CRM team to prioritize the sync condition endpoint implementation
- [ ] **R&D** to create a ticket to remove the in-memory Audience export comparison optimization from the full sync consent step (not urgent, but should be done before a large-scale customer successfully imports consents)
- [ ] **Documentation**: Platform scale limits (previously tested up to 2M contacts) should be formally documented — location TBD (Docs or GitHub limitations page per Lukasz's suggestion)
- [ ] **Clarification needed**: Prem (not present) is described as the "mastermind" behind the sync condition endpoint design — further details on the endpoint contract should be confirmed with him if implementing this for FSC Enterprise