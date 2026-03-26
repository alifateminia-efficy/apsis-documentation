---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One Integrations
topics: [legacy plugins, open-source connectors, third-party integrations, Zapier integration, Lime CRM plugin, lead creation flow, Tribe CRM virtual lead entity bug, webhook registration, duplicate profiles, generic connector, silhouette IDs, key spaces]
speakers: ["Erik Andersson (senior/outgoing developer)", "Michal Rosikiewicz (developer)", "Tomasz Kowalski (developer)"]
key_components: [Apsis One, generic connector, Tribe CRM, E-deal, Dynamics (by Sideshop), iAM Loyalty, Join CX (Intermail), Zapier, Lime CRM plugin, Sleeknote, WooCommerce, FSC Enterprise, One API, ECS/load balancer, AWS Lambda/API Gateway, Postman collection]
session_type: knowledge-transfer
subdomains: ["Lead creation", "Different Types of Connectors", "Duplicate profiles"]
---

## Session Overview

Erik Andersson walks Michal Rosikiewicz and Tomasz Kowalski through the history and current status of the various Apsis One integration repositories, including legacy plugins (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely), the Zapier app, and the abandoned Lime CRM plugin. He explains the business model shift away from APSIS-owned plugins toward third-party and generic connector partnerships. The latter half of the session focuses on a live demonstration of lead creation via form submission to the E-deal generic connector, and then a detailed technical deep-dive into a Tribe CRM bug where a virtual "lead" entity caused incorrect webhook registration, resulting in duplicate empty profiles being created in Apsis One for every real contact update received from Tribe.

---

## Legacy Plugins: History, Business Model Shift, and Current Status

### Original Strategy: APSIS-Owned Plugins

The original integration strategy was that for any CRM system requiring an installed plugin (e.g., Microsoft Dynamics, Presta Shop, Episerver/Optimizely), APSIS paid external companies to develop those plugins. APSIS owned the resulting code, and therefore also owned all support responsibilities. When a customer had an issue, they contacted APSIS, who then had to investigate and potentially schedule a paid engagement with the external development company — expensive if the root cause was on the APSIS side.

Plugins built under this model (all now in the `apsis-integrations` repository namespace with corresponding repo prefixes):
- Presta Shop
- Adobe Commerce (formerly Magento)
- Sitecore
- Drupal
- Optimizely

### Business Model Shift: Third-Party Partnership Model

[Erik Andersson]: When Felix became head of APSIS, the strategy changed. APSIS stopped paying external companies to build and own plugins. Instead, the external company (e.g., Sideshop for Dynamics) builds and owns the plugin, charges their own customers directly, and handles support. APSIS benefits from the high volume of profiles being synced.

Under the new model:
- The external developer owns customer relationships and support for the plugin.
- APSIS only provides support when the external developer can demonstrate their side is sending correct data but APSIS is returning errors.
- When APSIS adds new platform features (e.g., event tool syncing), it is up to the development partner to enable those features in their plugin — which is in their interest as it expands their product offering.

### Current Status of Legacy Plugins: Open Source, Unsupported

All legacy plugins are now considered **open source and unsupported**. Customers can theoretically download and install them, but APSIS provides zero support — including for existing customers still using them.

> [Erik Andersson]: "You should never need to sit [debugging] like this customer tried to install Drupal, it's not working. Why? And then you just say: not supported, their problem."

- **Presta Shop**: No active customers. Plugin is in `pre-launch` state — not even visible to customers in the UI.
- **Adobe Commerce**: No active customers.
- **Sitecore**: No active customers.
- **Drupal**: One old customer known to be using a customized version, but no support is offered even to them.
- **Optimizely**: No active customers.

[Erik Andersson] confirmed that most of these plugins use the **One API** for the majority of their functionality, similar to third-party integrations.

### Why the Repositories Still Exist

The decision was made not to delete the repositories because it was considered a waste of effort to remove them entirely. They were migrated into the new repository structure. However, Erik stated his personal preference would be to remove them entirely.

> [Erik Andersson]: "Essentially consider them dead connectors."

---

## Zapier Integration

The Apsis-Zapier integration is an app built by APSIS and uploaded to the Zapier platform. The source code lives in the `apsis-integrations-zapier` repository. APSIS owns and maintains it.

Key technical detail: **The Zapier integration uses One API 100%.** Everything that happens in Zapier flows through One API. There is no custom backend logic beyond that.

Current status: Active, with a meaningful number of customers using it. Not expected to require much development going forward.

**Practical use case**: Facebook Lead Ads (which lost its dedicated connector when the "what team" was let go) can now be handled via Zapier, as Zapier supports real-time triggers for new Facebook leads.

---

## Lime CRM Plugin: History, Purpose, and Preservation Rationale

### What It Was

APSIS attempted to build a custom plugin for Lime CRM approximately four years ago. The goal was to replicate a feature that existed in Dynamics: displaying all Apsis One email events and website events for a specific contact directly inside the CRM UI. This required:
- A custom UI element inside Lime
- Custom API endpoints inside Lime that retrieved data from APSIS via One API

The plugin was built before the **generic connector** concept existed.

### Why It Failed

[Erik Andersson]: There were significant complications with database persistence. The core problem was scoping data per customer/account: Lime instances can be shared across multiple accounts, and it was very easy to accidentally persist data for all customers on the environment rather than for a single customer if the wrong library or configuration was used. The team was never able to resolve this reliably.

Additionally, interest in the feature declined rapidly, so the project was scrapped.

### Why the Repository Was Preserved

Even though the plugin is non-functional and ancient, the repository was kept as a **reference for how to build a Lime plugin** — specifically:
- How to structure components
- How to name and generate them
- The overall plugin framework

> [Erik Andersson]: "The plugin and the implementation is absolutely ancient by now. I don't think you'll be able to run it, but the process for it still holds true."

### Future Relevance: Generic Connector for Lime

If there is ever a demand for a proper Lime ↔ APSIS integration, it would be implemented as a **generic connector** by building a Lime plugin that exposes the custom endpoints the generic connector protocol requires. **This should be built and owned by someone outside APSIS** (e.g., an external partner), not by the integration team, for the same support-ownership reasons as all other plugins.

[Erik Andersson]: Lime is accessible today as a CRM, just without the email/website events panel. That feature exists only in Dynamics, because it previously existed in Apsis Pro and therefore had to be replicated in Apsis One.

---

## Active Third-Party Integrations: Architecture and Responsibilities

### Fully One API-Based Third-Party Connectors (Sleeknote, WooCommerce, Playable)

For these connectors, APSIS's responsibility is minimal:

1. When a customer clicks **Connect** in the Apsis One UI, APSIS:
   - Generates a **One API key** for the customer
   - Creates a **key space** for that integration (e.g., a Sleeknote key space)
2. From that point, everything happens via One API inside the third-party system. The customer copies their credentials into the third-party platform's configuration.

> [Erik Andersson]: "The only thing we really do in integration for these third party ones is bootstrap the key space, generate a One API key, and then we are done."

### iAM Loyalty (Special Case)

iAM Loyalty uses One API for everything **from their side to APSIS**. The exception: they want **consent changes** to flow from APSIS back to iAM Loyalty. Since One API does not support that, APSIS handles this via **consent mappings** — APSIS listens to consent changes inside Apsis One and sends them back to iAM Loyalty.

Current trajectory: iAM Loyalty is being migrated to **Join CX** (same parent company, Intermail). iAM Loyalty will eventually be decommissioned, reducing the number of connectors to maintain.

### Join CX / Join Loyalty (Generic Connector Implementation)

Although classified as a third-party connector, Join CX has implemented the **full generic connector protocol**, including field mappings, full syncs, and all standard functionality. It is "third-party" in the sense that APSIS does not control their product roadmap. Intermail/Join CX has a close partnership with APSIS (APSIS handles marketing; they handle loyalty programs, points, offers).

---

## Policy on Building New Custom Plugins

[Erik Andersson]: Strong advice against the integration team ever volunteering to build new custom plugins.

> "I don't care if they pay me like 500,000 — we will lose money on this from R&D perspective."

If product or management demands it, the team should raise the following ramifications explicitly:
- Engineering hours will silently disappear into building, debugging, and researching the target system.
- The team will own all maintenance, support, and bug fixes forever.
- It needs to be a deliberate prioritization decision with full awareness of the cost.

Preferred alternative: Find an external company (e.g., Silverlord Consulting was mentioned) to build and own the plugin.

### Shopify: A Cautionary Tale

APSIS built a complete Shopify connector, including overtime work due to rushed timeline. After completion, it was discovered that Shopify's licensing terms would require APSIS to pay Shopify **12% of annual Apsis revenue**. The connector was scrapped entirely and deleted from the codebase.

---

## Lead Creation Flow: E-deal Demonstration

### Standard Lead Creation via Generic Connector

[Erik Andersson] demonstrated a live form submission to the E-deal generic connector on staging.

Flow:
1. A form submission in Apsis One triggers a batch message.
2. The **out-one-worker** processes the batch (type: `form`).
3. The form data is submitted to E-deal.
4. E-deal responds with their **lead entity** (a silhouette), including a `recordId`.
5. APSIS creates a profile in the Apsis One audience with a **silhouette ID** (not a CRM ID) in the appropriate key space.

Key points:
- Lead creation flows **only from Apsis → CRM**. APSIS never downloads leads from the CRM.
- The only time APSIS creates profiles with silhouette IDs is when the CRM responds with them during a form submission or full sync.
- The CRM may send a **delete webhook** for a silhouette if they decide not to proceed with a lead. APSIS supports this via the same webhook mechanism used for profile updates/deletes.
- APSIS does **not** expect silhouette create or update webhooks from the CRM — only deletes.

### Webhook Registration During Installation

During installation, APSIS registers webhooks for all entities it has a configuration for. The rationale: APSIS doesn't know in advance which entity types the CRM might send delete events for. This general approach was correct for most connectors but caused the Tribe bug described below.

---

## Tribe CRM Bug: Virtual Lead Entity Causing Duplicate Profiles

### Background: Tribe's Dynamic Entity Selection

Tribe CRM has a configuration setting where the customer selects **which entity type should be created in Tribe when an Apsis form is submitted**. This is a dropdown in the Tribe admin UI — options include "contacts," "potential customer," or any custom entity. The selection is fully dynamic.

APSIS introduced a **virtual "lead" entity** in its Tribe configuration to handle this dynamic selection. When APSIS asks Tribe for the mappable fields for "lead," Tribe returns the fields for whichever entity is currently selected in that dropdown. This virtual entity is not a real entity in Tribe — it is an abstraction over the dropdown selection.

### The Bug

When APSIS registered webhooks during Tribe installation, it included the virtual "lead" entity in the list of entities to register webhooks for. This was an error, because **Tribe will never send webhook events for an entity literally called "lead"** — it will always use the actual selected entity (e.g., "contacts").

The unintended side effect:
1. Tribe sent a real-time delta sync for a **contact** to APSIS.
2. Because APSIS had also registered a webhook endpoint for "lead," Tribe incorrectly sent the **same contact data to the lead webhook endpoint as well**.
3. APSIS interpreted this as a new lead creation event.
4. APSIS looked up the unique identifier field (ID) from the request — which happened to be the same field name used for contacts — and extracted a value.
5. However, the **"lead" key space is completely separate from the "contact" key space** in Apsis One.
6. Result: APSIS created a new, empty profile in the **lead key space** containing only the silhouette ID. No other attributes were mapped, because no outbound mappings existed for this virtual entity.

[Erik Andersson]: "We checked here — we did get a request. The lead entity has this unique identifier ID which happens to be the same as contact. So we could find an ID for this lead in the request to us. But the lead for us has a completely different key space. So because the ID field from the contact is the same as this ID field here, we took the ID for the contact but created a new empty profile in the lead key space."

This produced **duplicate/ghost profiles** in Apsis One: one real contact profile and one empty lead profile, both sharing the same underlying record ID from Tribe.

### Fix Applied

[Erik Andersson] added a parameter to the webhook registration logic: `do not create webhooks` flag for the Tribe lead entity. During installation, when iterating over the list of entities to register webhooks for, the code now checks this flag and skips registration for the virtual lead entity.

The fix applies to **new installations only**. For the affected existing customer, Erik manually deleted the incorrect lead profiles and the erroneous webhook registration using the generic connector Postman collection (see tooling note below).

### Remaining Risk: Other Affected Customers

There may be other customers who installed the Tribe connector after this bug was introduced and who are also experiencing empty ghost profiles. The SOC/support team has been briefed.

**Signal to watch for**: Empty profiles from Tribe with only a silhouette ID and no other attributes → almost certainly caused by this bug on an installation where the lead webhook was incorrectly registered.

[Erik Andersson]: Emphasized that support questions about this must go through the SOC and support channel, not directly to the dev team via account managers, in order to preserve the knowledge-sharing and avoid repeated explanations.

### Desired Long-Term Fix

Erik's preferred resolution is to work with Tribe (via Rose, who is now a product owner at Tribe and provides a "backdoor" relationship) to always respond to form submissions with a **static, predictable entity** — ideally always "contacts." This would allow APSIS to:
- Remove the virtual lead entity entirely from the Tribe configuration
- Simplify outbound mappings, field mappings, and consent mappings to only reference the contact entity
- Eliminate the fragility of the current flow, where a customer changing the Tribe dropdown retroactively breaks all configured mappings

> [Erik Andersson]: "Right now, everyone selects contacts as the entity, and it just so happens that the configuration we have for contacts will be exactly the same as for leads — but that's just pure sheer luck."

---

## Tooling: Generic Connector Postman Collection

The generic connector includes a Postman collection that allows direct interaction with a customer's CRM environment, given their API key and environment URL. Capabilities include:
- `webhook.record.delete` — delete a specific webhook registration
- List webhooks
- Create webhooks
- Get schema / field mappings (useful for debugging "why can't I see any mappings?" issues)

This collection is the primary tool for manually cleaning up incorrect webhook registrations or debugging customer-specific connector issues.

---

## Infrastructure Note: Dynamics Cold Start Issue (Historical)

[Erik Andersson]: Microsoft Dynamics instances that have not been used for a period of time enter a **cold storage** state. The first API request to such an instance can take an obscenely long time (over a minute).

**Historical problem**: When the integration services ran as AWS Lambda functions behind API Gateway, the cold Dynamics instance would cause the installation process to time out at the API Gateway layer, making the first installation attempt always fail.

**Resolution**: Migrated to **ECS tasks + load balancer**, which eliminated the timeout problem because the connection is long-lived and not subject to API Gateway timeout constraints.

---

## UI/Catalog Cleanup: Deprecated Integrations Still Shown

Several integrations that no longer exist in the backend are still showing as cards in the Apsis One integrations catalog UI. The integration backend controls whether a "Connect" button is shown (active) or a "Read more" link is shown (inactive), but the cards themselves are rendered by the frontend.

Integrations flagged for removal:
- Shopify (already deleted from codebase)
- Maxo (discontinued)
- Power BI
- E-commerce plugin
- Facebook Lead Ads
- Tassitura, Ambraco, Google Data Studio (already deleted from backend)
- Presta Shop (still shows as pre-launch, not visible to customers but visible to admins)

Action: Erik will create a cleanup story. Backend cleanup is already partially done; frontend card removal will require frontend team (Chris) involvement, as it may touch data models beyond simple component deletion.

---

## Key Takeaways

1. **Legacy plugins (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely) are open source and fully unsupported** — no support even for existing customers. Do not volunteer to fix, extend, or debug them.

2. **The business model shift** moved plugin ownership to external partners. APSIS now only provides support when the partner can demonstrate APSIS is at fault. The integration team should never volunteer to build new plugins; doing so permanently transfers maintenance and support burden to APSIS R&D.

3. **Third-party integrations** (Sleeknote, WooCommerce, Playable) require almost no APSIS backend involvement: APSIS bootstraps a key space and One API key; everything else goes through One API on the partner side.

4. **Lead creation** flows only from Apsis → CRM. APSIS creates profiles with silhouette IDs when a CRM responds to form submissions with a lead entity. APSIS never downloads leads from the CRM; it only accepts delete webhooks for silhouettes.

5. **The Tribe virtual lead entity bug** caused duplicate empty profiles in Apsis One. Root cause: APSIS registered a webhook for a virtual entity ("lead") that Tribe would never actually emit events for, but Tribe incorrectly forwarded real contact events to that endpoint anyway. Fixed for new installations via a `do not create webhooks` flag. Existing installations may need manual cleanup via the Postman collection.

6. **The Lime CRM plugin repository** is preserved purely as reference material for how to build a Lime plugin. It is non-functional and should not be used directly.

7. **Support questions must flow through SOC/support**, not directly to the dev team via account managers — especially for known bugs like the Tribe duplicate profile issue, to preserve institutional knowledge.

---

## Unresolved Questions / Action Items

- **[Erik]** Create a cleanup story to remove deprecated integration entries from the backend integration list and coordinate with frontend team (Chris) to remove the corresponding UI cards.
- **[Erik]** Submit a PR for the `do not create webhooks` flag fix for the Tribe lead entity.
- **[SOC Team]** Monitor for additional Tribe customers with empty silhouette-ID-only profiles indicating they are affected by the same webhook bug.
- **[Erik / Rose backdoor]** Pursue alignment with Tribe product team to always respond to form submissions with the static "contacts" entity, enabling removal of the virtual lead entity from APSIS's Tribe configuration entirely.
- **⚠️ Ambiguity**: It is unclear which specific Tribe connector version introduced the webhook bug, making it uncertain how many customers may be affected. SOC has been informed but the scope is not yet determined.
- **⚠️ Ambiguity**: The Drupal plugin reportedly has one existing customer using a customized version. No support is offered, but the repository should not be deleted in case it is needed for reference.