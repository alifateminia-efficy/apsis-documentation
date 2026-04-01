---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One Integrations
topics: [plugin repositories, open source connectors, deprecated integrations, third-party integrations, Zapier integration, Lime CRM plugin, Shopify connector history, WooCommerce, iAM Loyalty, Join CX, Tribe virtual lead entity bug, generic connector webhooks, lead entity creation, silhouette ID, key space bootstrapping]
speakers: ["Erik Andersson (Integration Tech Lead / Senior Developer)", "Michal Rosikiewicz (Team Lead / Developer)", "Tomasz Kowalski (Developer)"]
key_components: [Apsis One, Generic Connector, Zapier App, Lime CRM Plugin, Microsoft Dynamics, Tribe CRM, E-deal, iAM Loyalty, Join CX (Intermail), FSC Enterprise, Presta Shop, Adobe Commerce (Magento), Sitecore, Drupal, Optimizely, Shopify (removed), Facebook Lead Ads (removed), WooCommerce, SuperOffice, Playable, Sleeknote, Postman Collection]
session_type: knowledge-transfer
---

# Session Overview

This session covers two main areas. First, Erik provides a comprehensive walkthrough of the legacy plugin repositories under the `apsis-integrations` prefix in the FSC, explaining the historical business model that led to their creation, the strategic shift away from APSIS owning plugin support, and the current deprecated/open-source status of most of these connectors. Second, Erik walks through a live debugging narrative of a significant bug affecting Tribe CRM customers, where a virtual "lead" entity concept in Tribe's generic connector configuration caused phantom empty profiles to be created in Apsis One. The fix and its implications are explained in detail.

---

## Plugin Repository Overview: The `apsis-integrations` Repositories

### Original Business Model for Plugins

The original strategy when the integration platform was first implemented was: for any CRM system that requires a plugin to be installed (e.g., Microsoft Dynamics, Presta Shop, Episerver/Optimizely), APSIS would pay an external company to build the plugin, because that knowledge didn't reside inside APSIS.

- **APSIS owned the code** developed by these external companies.
- **APSIS also owned the support.** If a customer had an issue, they contacted APSIS, who would investigate and, if the fault lay with the plugin, coordinate with the external developer.
- Bug fixes were free; new features requested by APSIS cost significant money per hour.

### Strategic Shift Under New Leadership

When Felix became head of APSIS, the strategy changed:

> "APSIS no longer wanted to own any support for plugins because it takes a lot of time for us to handle that."

The new model:
- APSIS stopped paying external companies to build and own plugins.
- Instead, APSIS sought **business partners** who would own the plugin themselves.
- The **customer pays the third-party developer** (not APSIS) for the plugin. That company owns the customer relationship and the support.
- **APSIS's benefit:** the partner ships a large volume of profiles per month into the Apsis platform.
- **Support boundary:** APSIS only provides support if the partner confirms they are sending correct data but are receiving errors or incorrect responses from APSIS. This significantly reduces the support burden, as the partner's own developer vets the issue first.
- **New feature enablement:** When APSIS adds a feature (e.g., event tool syncing), it's up to the development partner to adopt it — they are incentivized to do so because it strengthens their product offering.

### Current Status of the Legacy Plugin Repositories

The following repositories exist under the `apsis-integrations` prefix. Most are effectively dead:

| Connector | Status | Notes |
|---|---|---|
| Presta Shop | Open source, unsupported | No current customers. Plugin is marked **pre-launch** — not even released to customers. |
| Adobe Commerce (Magento) | Open source, unsupported | No current customers. |
| Sitecore | Open source, unsupported (private repo) | No current customers. |
| Drupal | Open source, unsupported (private repo) | One very old customer using a customized version, but not actively supported by the integration platform. Keep the repo; do not delete. |
| Optimizely | Open source, unsupported (private repo) | No current customers. |

> [Erik Andersson]: "Consider them dead connectors for all I care. You should never need to sit and investigate why a customer can't install Drupal — that's their problem. You should never give any support for them."

**Important caveat:** These are considered open source. A customer could theoretically download the source code and self-maintain/extend the plugin. APSIS offers no support in any case.

**Repository retention rationale:** The decision was made (against Erik's preference) to migrate these repositories to the new repository structure rather than delete them, as deletion was considered a waste. They should be treated as non-existent for practical support purposes.

**Only exception for future work:** If Product formally decides to build a new connector for one of these systems, someone would need to work on it. Until that decision is made at a management level, no work should be done on these.

---

## Shopify Connector: A Cautionary History

A complete Shopify connector was built by the integration team, including overtime work due to urgency. After completion, it was discovered that utilizing Shopify's platform in this way required **paying Shopify 12% of annual APSIS revenue**. The entire connector was scrapped.

> [Erik Andersson]: "Yes, thank you. Thank you so much."

The Shopify repository/listing should be cleaned up. There is no Shopify connector.

---

## Facebook Lead Ads: Another Deprecated Connector

Facebook Lead Ads was contentious internally — neither the integration team nor the "what" team wanted to own it. Eventually a hybrid was built where Integration handled authentication and the "what" team built the logic. When the "what" team was let go, the connector died with it.

**Current workaround for customers:** Zapier has real-time triggers for new Facebook Lead Ads leads, which can be used to funnel data into Apsis One.

---

## Zapier Integration Repository

The `apsis-integrations-zapier` repository contains the source code for the Apsis app published in the Zapier platform.

- The Zapier app is built locally and then packaged and uploaded to Zapier from a developer's machine.
- This repository is **owned and maintained by the integration team.**
- The app uses the **One API 100%** — everything that happens in Zapier goes through One API.
- The integration is stable; major changes are not anticipated, but it has a meaningful customer base.

---

## Lime CRM Plugin Repository

### Background and Purpose

A Lime CRM plugin was built approximately four years ago by Erik (at the time working as a consultant). It is **not in use** and was never successfully completed, but the repository is **preserved as a reference implementation**.

**Original goal:** Replicate a feature that existed in Microsoft Dynamics — displaying email events and website events for a specific contact, retrieved from Apsis One via the One API, as a custom UI tab inside the CRM.

**Why it was built for Dynamics but attempted for Lime:** The feature existed in Apsis Pro, so it had to exist in Apsis One's Dynamics connector. Lime doesn't natively know how to retrieve Apsis One events, so a custom plugin with custom UI elements and custom endpoints was attempted.

### Why It Failed

> [Erik Andersson]: "There were unfortunately some quite complicated configuration issues with database access and persistence — how do you keep the data in the instance for only one customer?"

The specific blocker: if a Lime instance is shared with multiple accounts/customers, using the wrong configuration or library could accidentally persist data globally across the whole environment rather than scoping it to a specific customer. This was never solved reliably.

Interest in the feature also faded quickly, so the project was scrapped.

### Why the Repository Is Preserved

The repository is kept as a **reference for how to build a Lime plugin**:
- How to build and name components
- How to generate them
- The overall framework structure

> [Erik Andersson]: "The plugin and implementation is absolutely ancient by now. I don't think you'll be able to run it, but the process still holds true — you would still do a lot of the same setup."

### Future Lime Integration Possibility

If there were ever a business decision to integrate Apsis with Lime CRM using the generic connector model, a Lime plugin could be built that implements all the custom endpoints the generic connector requires. Lime supports custom endpoints. **This plugin would be built and owned by a business partner (e.g., a consulting company), not the APSIS integration team.**

[Erik Andersson]: "You wouldn't build that plugin, but someone else that will own the customer could build that plugin."

### Important Warning: Cold Storage Behavior in Dynamics (Observed During Demo)

> [Erik Andersson]: "If a [Dynamics] instance has not been utilized for some time, it goes into a kind of cold storage. The first API request to it can take an obscenely long time."

This caused failures historically when services were Lambda functions + API Gateway (request to install would exceed the API Gateway timeout). This is **no longer an issue** because the service was migrated to ECS tasks + a load balancer.

---

## Active Third-Party Integrations: Architecture and Key Details

These integrations are **not owned by APSIS** in terms of product direction, but APSIS provides the platform integration layer.

### Sleeknote, Playable, WooCommerce (One API Only)

For these connectors, the integration team's role is minimal:

1. When the customer clicks "Connect," the integration platform:
   - Generates a **One API key** for the customer.
   - Creates a **Sleeknote key space** (or equivalent) for the customer in the background.
2. From that point on, everything is handled by the third-party system via the One API directly.
3. The customer configures credentials (API key, key space, section) inside the third-party platform (e.g., Sleeknote).

> [Erik Andersson]: "The only thing we really do in integration for these third-party ones is bootstrap the key space, generate a One API key, and then we are done."

### iAM Loyalty (Special Case: Consent Mapping)

iAM Loyalty uses the One API for everything **except one case:** they want **consent changes** exported from Apsis back to iAM Loyalty. The One API does not support pushing consent changes outbound, so the integration platform handles this:

- Consent mappings are configured in the integration platform.
- When a consent change occurs in Apsis One, the integration platform detects it and sends it back to iAM Loyalty.

**Future:** iAM Loyalty is being decommissioned. The same company (Intermail) is migrating all iAM Loyalty customers to their **Join CX** platform. When that migration is complete, iAM Loyalty will be removed as a connector.

### Join CX / Join Loyalty (Full Generic Connector Implementation)

Join CX has implemented the **full generic connector protocol**:
- Full field mappings
- Full syncs
- Complete configuration

It is listed as a "third-party" integration because APSIS doesn't control their product direction — they decide their own priorities and features. However, they have chosen to adopt the generic connector fully.

Both iAM Loyalty and Join CX are owned by the same company: **Intermail**. They are heavily integrated with Apsis because Apsis handles marketing while Intermail handles loyalty programs (points, offers, etc.).

### SuperOffice

SuperOffice has been removed from the active connectors list (but may still appear in UI). It was a generic connector implementation.

---

## Integration Platform UI: Connector Cards and Cleanup

The integration platform page lists connectors with either a "Connect" button (available for installation) or a "Read More" link (not available). The integration team controls this via their own list of available integrations.

**Currently identified for cleanup** (connectors that should be removed from the UI and/or backend lists):

- Shopify (completely deleted from codebase)
- Facebook Lead Ads (dead)
- Tessitura (deleted)
- Acourse / A-course (deleted)
- Google Data Studio (deleted)
- Umbraco (deleted)
- Power BI (should be removed)
- Maxo (discontinued)
- Presta Shop (pre-launch, never released — still showing as a card)

**Still active:**
- WooCommerce
- iAM Loyalty
- Enterprise/FSC 12.1 (the newer implementation; FSC 12.0 is the old one)
- E-deal
- Web CRM
- Dynamics by Sideshop
- Join CX

**Cleanup plan:** Erik will create a story to clean up the back-end integration list. The front-end connector cards also need to be removed, but this involves UI component work (Angular) that will need the front-end team (Chris's team) to handle data model implications. Erik to coordinate with that team.

> [Erik Andersson]: "This is aching to false advertisement because it hasn't been like this for quite some time."

---

## Tribe CRM: Virtual Lead Entity Bug — Root Cause and Fix

This is the primary debugging narrative from the session.

### Background: How Lead Collection Works (E-deal Example)

When a customer submits a form in Apsis and a generic connector sync is configured to an CRM like E-deal:

1. Apsis sends the form submission data to E-deal.
2. E-deal responds with a **lead entity** (a `silhouette`), including a `record_id` (silhouette ID).
3. Apsis creates a profile with a **silhouette ID** (not a CRM ID) in the lead key space.

Key rule: **Lead gathering only flows from Apsis → CRM.** Apsis creates silhouette-ID profiles only when:
- The CRM responds with a silhouette in a form submission response, OR
- During a full sync

**Apsis never downloads leads from the CRM** — there has been no use case for it.

#### Webhook Registration for Silhouettes

Apsis does register webhooks even for silhouette entities because there is one valid use case: **if the CRM decides not to proceed with a lead, it can send a DELETE request** to Apsis for that silhouette.

> [Erik Andersson]: "Previously we never registered any webhooks for these silhouettes because we never expected them to send us any updates. We did forget the use case where they want to delete the silhouettes."

The fix was to register webhooks for **all configured entities** (not just the main profile entity) during installation, to catch these delete requests.

---

### The Tribe CRM Problem: Virtual Lead Entity

#### Tribe's Dynamic Entity Configuration

Tribe CRM has a configuration in their admin UI: a **dropdown selector** where the customer chooses which Tribe entity should be created when a form is submitted from Apsis. This entity is completely dynamic — it could be "Contact," "Potential Customer," or any custom entity defined in Tribe.

The generic connector cannot handle fully dynamic entities because it needs to know:
- The entity name (to register webhooks)
- The unique identifier field name for that entity
- The field mappings

#### The Virtual "Lead" Entity in the Tribe Configuration

To work around this, the Tribe generic connector configuration defines a **virtual entity called "lead"** on the Apsis side. When Apsis asks Tribe for the mappable fields for "lead," Tribe is supposed to return the fields for whatever entity is currently selected in that dropdown.

**The problem:**
- When a form is submitted and Tribe processes it, Tribe responds with the **actual entity** (e.g., `potential_customer`), not with an entity called `lead`.
- But Apsis registered a webhook endpoint for the virtual entity `lead`.
- Tribe (incorrectly, on their side) then **also sent the same contact data to the lead webhook endpoint**.
- Apsis received this data at the lead endpoint and interpreted it as: "A new lead has been created in Tribe."
- The `lead` entity has its own separate key space with different mappings than `contact`.
- Apsis extracted the ID from the incoming data (which happened to be the same field as contact's ID) but created a **new empty profile in the lead key space**.
- Result: **Phantom empty profiles with only a silhouette ID**, running in parallel with the real contact profiles.

> [Erik Andersson]: "Because the ID field here from the contact is the same as this ID field here, we took the ID for the contact but created a new empty profile in the lead key space. And as you said, we have no real mappings for this, so it was an empty profile that happened to have a lead ID that lived in parallel with the real contacts."

#### The Fix Applied

Erik added a parameter to the webhook registration logic: **`do_not_create_webhooks`** flag on specific entities. During installation, before adding an entity to the list of webhooks to register, the code checks this flag. If set, it skips webhook registration for that entity.

This prevents the phantom profile creation for new installations.

**For affected existing customers:** Erik manually deleted the incorrectly registered lead webhooks using the Postman collection (see below). The correct webhooks for the contact entity remain in place.

#### Ideal Long-Term Fix (Not Yet Implemented)

> [Erik Andersson]: "What I would really want to do for Tribe is have them always give us some static entity so we have some predictability. Right now — what mappings should you set up? What consents should you configure for the initial consent? It's not a well-working flow."

The preferred resolution: **remove the virtual lead entity entirely from the Tribe configuration** and always use the `contact` entity directly. This would:
- Remove the need for the virtual entity abstraction
- Eliminate the mismatched webhook registration
- Make the outbound mapping, field mappings, and consent mappings straightforward

Erik noted that **Rose is now a product owner at Tribe**, providing a channel to potentially influence Tribe to standardize on always returning `contact`. This would allow the Tribe connector to be simplified on the Apsis side.

**Current practical reality:** As far as Erik has been told, virtually all Tribe customers select `contact` as their entity anyway, so the contact configuration happens to match the lead configuration — but this is coincidence, not design.

#### Warning About Other Affected Customers

> [Erik Andersson]: "There might be other customers suffering from this same issue. The SOC team will need to monitor for empty profiles from Tribe with only a silhouette ID — that is essentially guaranteed to be this bug."

**Diagnostic signature:** An empty profile in the Tribe customer's account that has only a silhouette ID and no other mapped attributes indicates this bug.

---

## Postman Collection for Generic Connector Debugging

The generic connector Postman collection is an important operational tool. It includes endpoints to:
- **Create** webhooks in a CRM instance
- **List** existing webhooks in a CRM instance
- **Delete** webhooks from a CRM instance
- **Get schema / mappable fields** from a CRM instance

**How to use:**
1. Obtain the customer's CRM environment URL (sometimes via an account manager).
2. Obtain the customer's API key (the one stored in Apsis for that customer's installation).
3. Configure these in the Postman collection.
4. You can then directly inspect and manage webhooks, or debug field mapping issues (e.g., empty schema responses, permission errors, 404s).

> [Erik Andersson]: "This Postman collection is very, very useful for debugging for a customer."

---

## Support Process: Why Questions Must Go Through SOC

[Erik Andersson] emphasized this point strongly in the context of the Tribe bug:

> "This is why it is so important that these questions go via SOC and support. Now they know about this problem. We've described it to them. If there are more customers that experience the same thing, they already know what the problem is. If these questions get pinged to us directly from account managers, we are circumventing that whole process and that knowledge sharing will not be there — and then you will get the same question from SOC and support in the future as well."

The integration team should **not** accept direct escalations from account managers, bypassing SOC/support. This breaks the knowledge-sharing chain.

---

## Decision-Making Around Building New Plugins: Strong Warning

[Erik Andersson] was emphatic on this point:

> "You should never volunteer to do it. It will be a time sink — you will own the maintenance, the support, the debugging, and even more research on how the external system works. You don't want to do that."

If Product or Professional Services comes requesting a new plugin/connector for a specific CRM:
1. **Preferred option:** Find an external company/partner to build and own the plugin (the current business model).
2. **Alternative option only if forced:** The integration team builds it — but decision-makers must be explicitly informed of the ramifications (ongoing support burden, hours disappearing into maintenance).

> [Erik Andersson]: "I don't care if they pay 500,000 — we will lose money on this from an R&D perspective. Of course a product manager could still decide that. But then you have these points to bring up."

External company mentioned as a potential resource for this: **Silverlord Consulting**.

---

## Key Takeaways

1. **Legacy plugin repos (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely) are effectively dead.** They are open source and unsupported. No support should be provided. Drupal repo should be retained but no work done on it.
2. **The business model shifted:** APSIS no longer owns plugin development or support. Partners own their plugins; APSIS provides the platform.
3. **The Zapier app** is owned and maintained by the integration team and uses One API exclusively.
4. **The Lime CRM plugin** is a reference implementation only — not functional, not used — preserved to show how a Lime plugin could be built if ever needed by a partner.
5. **Third-party integrations (Sleeknote, WooCommerce, Playable):** Integration team's role is only to bootstrap a key space and generate a One API key. Everything else is on the third party.
6. **iAM Loyalty** is being decommissioned in favor of **Join CX** (same company: Intermail).
7. **Tribe's virtual "lead" entity is a known architectural fragility.** The webhook registration bug has been patched (do-not-create-webhooks flag), and existing affected customers have been manually remediated. Long-term fix requires standardizing on the `contact` entity.
8. **The Postman collection** is the primary debugging tool for generic connector webhook and schema issues.
9. **Never accept direct escalations from account managers** — all support must go through SOC to preserve institutional knowledge.
10. **Never volunteer to build a new plugin** without clear management buy-in and explicit acknowledgment of the ongoing maintenance cost.

---

## Unresolved Questions and Action Items

- [ ] **[Erik]** Create a story to clean up deprecated integrations in the integration backend list (Shopify, Facebook Lead Ads, Tessitura, Acourse, Google Data Studio, Umbraco, Power BI, Maxo, Presta Shop pre-launch entry).
- [ ] **[Michal / Chris's front-end team]** Create a story for the front-end team to remove deprecated connector cards from the integration platform UI. Michal to connect Erik's back-end story with the front-end story.
- [ ] **[Erik]** Submit a PR for the `do_not_create_webhooks` fix for the Tribe lead entity webhook registration bug.
- [ ] **[SOC team]** Monitor for existing Tribe customers with empty profiles containing only a silhouette ID — these indicate exposure to the lead entity webhook bug and may require manual remediation.
- [ ] **[Erik / Tribe via Rose]** Ongoing: pursue standardizing Tribe's form submission response to always use `contact` entity, enabling removal of the virtual `lead` entity from the Tribe connector configuration. ⚠️ *Timeline and outcome uncertain.*
- [ ] **[Ambiguous]** It is unclear which specific customers (beyond the one Erik manually fixed) may be affected by the Tribe lead entity bug. Investigation scope not defined in this session.