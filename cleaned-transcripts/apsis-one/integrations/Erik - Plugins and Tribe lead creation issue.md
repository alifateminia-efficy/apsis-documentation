---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One - Integrations
topics: 
  - Plugin repositories and legacy integrations
  - Business strategy shift from owned to partner-managed plugins
  - Third-party vs native integrations
  - Generic connector architecture
  - Tribe CRM lead entity webhook issue
  - Integration cleanup and deprecation
speakers:
  - Erik Andersson (Integration Lead)
  - Michal Rosikiewicz (Product/Team Lead)
  - Tomasz Kowalski (Team Member)
key_components:
  - Apsis Integrations repositories (Adobe Commerce, Presta Shop, Sitecore, Drupal, Optimizely, Zapier, Lime)
  - Generic Connector protocol
  - One API
  - Webhook registration and management
  - Key spaces (silhouette IDs, lead entities)
  - Third-party integrations (SuperOffice, Sleeknote, WooCommerce, Join CX, I am Loyalty)
  - Native FSC integrations (FSC 12.1, E-deal, Web CRM)
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson walks the team through the landscape of plugin repositories in the Apsis One Integrations domain, explaining the historical business model shift that transformed APSIS from owning third-party integrations to partnering with external vendors. The session covers deprecated open-source plugins (Adobe Commerce, Presta Shop, Sitecore, Drupal, Optimizely), actively maintained third-party connectors, and native FSC integrations. A significant portion addresses a critical bug discovered with the Tribe CRM integration involving incorrect webhook registration for virtual lead entities, which caused empty profile creation in parallel to real contact records.

---

## Historical Context: The Plugin Business Model Shift

### Original Integration Strategy

[Erik Andersson]: When integration was first implemented, the strategy was straightforward: for any CRM system requiring a plugin, APSIS would pay an external company to develop it. Examples included:
- Microsoft Dynamics
- Presta Shop
- Episerver
- Optimizely

The knowledge of building integrations with these systems did not reside within APSIS, so external development was necessary.

**Business model:** APSIS owned both the code and support responsibility. If a customer experienced issues with the plugin, they contacted APSIS support, who would investigate. If the issue lay within the CRM system itself, APSIS had to coordinate with the plugin developer to fix it. Bug fixes were provided at no cost, but feature additions or configuration adjustments could be expensive per-hour consulting costs.

### Strategic Shift Under New Leadership

[Erik Andersson]: When Felix became head of APSIS, the business strategy changed fundamentally. APSIS no longer wanted to own support for plugins because:
- Maintaining plugin support consumed significant internal resources
- The original plugin developers could diagnose issues much faster from their own logs
- It was inefficient for APSIS support to be the middleman

**New model:** Rather than APSIS paying external companies to build plugins and owning them, APSIS shifted to a **partner ecosystem model**. Now:
- External vendors build and own the plugins
- Customers pay the vendor directly, not APSIS
- APSIS still benefits through data volume (profiles synced per month)
- Support responsibility transfers entirely to the vendor

**Advantage of the new model:** When APSIS releases new platform features (e.g., event tool syncing), the responsibility to enable support in plugins falls on each vendor. This incentivizes vendors to keep their integrations up-to-date, as expanded functionality makes their products more valuable to customers.

---

## Current Integration Repository Landscape

### Deprecated Plugin Repositories (Open Source, No Support)

The following repositories represent integrations APSIS originally owned but have been deprecated. They are now classified as **open source with zero official support**:

#### Presta Shop
- **Status:** Pre-launch, not released to customers
- **Historical context:** E-commerce platform integration
- **Current support:** None. Customers can download and use the plugin at their own risk, but APSIS will not provide updates or bug fixes.

#### Adobe Commerce (formerly Magento)
- **Status:** Open source, no customers
- **Historical context:** Major e-commerce platform
- **Current support:** None.

#### Sitecore
- **Status:** Open source, no customers
- **Historical context:** CMS system
- **Current support:** None.

#### Drupal
- **Status:** Open source, one legacy customer still using a customized version
- **Current support:** None, even for the existing customer
- **Clarification:** Even if a customer paid for this integration historically, zero support will be provided going forward.

#### Optimizely
- **Status:** Open source, no customers
- **Current support:** None.

#### Episerver
- Not visible in current repository list (appears to have been removed).

[Michal Rosikiewicz]: All of these except Presta Shop and Adobe Commerce are marked as private repositories; they are not shared publicly.

[Erik Andersson]: The decision was made—against Erik's technical preference—to completely drop support for these connectors. They remain available as open-source code for customers who wish to self-maintain, but APSIS provides nothing. In Erik's assessment, these repositories are **"essentially dead connectors."** The only scenario where work would resume is if APSIS Product decides to rebuild a connector from scratch.

### Rationale for Keeping Deprecated Plugins

Although these integrations are unsupported, APSIS chose not to delete the repositories entirely because:
1. They were already developed and stored
2. Complete deletion was considered wasteful
3. They serve as reference implementations for plugin architecture
4. If a future customer request arises, the codebase is preserved
5. They exemplify how to build plugins for different CRM systems

---

## Actively Maintained Integrations

### The Zapier Integration (Apsis Integrations Zapier)

**Repository:** `apsis-integrations-zapier`

[Erik Andersson]: This is an application APSIS built within Zapier. The source code is stored in this repository because the development cycle involves building locally and packaging uploads to Zapier's servers.

**Key characteristics:**
- **Ownership:** APSIS owns and maintains the Zapier app
- **API utilization:** Uses the **One API 100%** — all Zapier app functionality goes through One API endpoints
- **Customer usage:** Quite significant customer base using this integration
- **Maintenance burden:** Minimal; the app runs and exists but requires little ongoing work
- **Security model:** APSIS provides the Zapier app code but customers interact through Zapier's hosted platform

**Use case:** Customers use Zapier's real-time triggers for events like new leads, which can be funneled into APSIS. This is particularly useful for scenarios like Facebook Lead Ads, where APSIS chose not to build a native connector.

### The Lime Plugin (Apsis Integrations Lime)

**Repository:** `apsis-integrations-lime`

[Erik Andersson]: This is an **ancient, reference-only repository** kept for documentation and architectural guidance. The actual plugin is non-functional and should never be deployed to production.

**Historical context:**

Four years ago, Erik attempted to build a plugin for Lime CRM to replicate a feature that existed in Microsoft Dynamics: retrieving email events and website events for a contact from APSIS One and displaying them within the Lime CRM interface.

**Why it was attempted:**
- APSIS had good contacts at Lime (several former APSIS employees had moved there)
- Lime provided access to test environments
- Erik had Python development skills, which were rare in the team at the time
- Lime had decent documentation

**Why it failed:**

Multiple complications emerged around **data persistence and isolation**:
- Lime plugins share instances across multiple customer accounts
- Configuring data so it persists only for one customer while not leaking to others proved extremely difficult
- The team could never achieve an efficient, safe solution
- The feature interest declined quickly

**Current purpose:**

The repository is preserved as a **reference for plugin architecture patterns**, showing:
- How to build a Lime plugin
- How to structure custom UI elements within Lime
- How to implement custom API endpoints
- General plugin scaffolding and naming conventions

[Michal Rosikiewicz]: If in the future APSIS needed to build a Generic Connector implementation for Lime, the pattern in this repository would be instructive. However, the actual plugin code is too outdated to run.

[Erik Andersson]: The process for building a Lime plugin would likely be similar today, but the implementation details are ancient. The important takeaway is that **custom plugin development for specific CRM systems should be avoided unless absolutely necessary**, because it transfers maintenance and support burden entirely to APSIS.

---

## Third-Party Integrations (No Plugins Required)

These integrations do **not** require plugins installed in customer systems. Instead, they use the **One API** and, in some cases, the **Generic Connector protocol**.

### SuperOffice

**Status:** Third-party CRM, fully functional

**API model:** One API

**Setup:** During integration installation, APSIS generates a One API key. Everything then flows through One API.

### Sleeknote

**Status:** Third-party customer engagement platform

**API model:** One API

**Bootstrap process:**
1. Customer initiates integration from APSIS One
2. APSIS generates a One API key
3. APSIS creates a Sleeknote key space
4. Customer copies/pastes credentials into Sleeknote's system
5. Sleeknote configures which APSIS accounts, sections, key spaces, and One API credentials to use
6. All subsequent data flows via One API

[Erik Andersson]: APSIS's role is minimal: bootstrap the key space, generate the API key, and then step back. Everything else is Sleeknote's responsibility.

### WooCommerce

**Status:** Third-party e-commerce platform, actively used

**API model:** One API

### I am Loyalty

**Status:** Third-party loyalty program platform (being migrated to Join CX)

**API model:** Primarily One API, **with an exception for consent handling**

**Special characteristics:**
- Uses One API for all standard customer data
- **Exception:** One API does not support consent **export** (sending consent changes from APSIS back to I am Loyalty)
- **Solution:** APSIS implements **consent mappings** for this integration
- **How it works:** When a customer consent changes in APSIS One, APSIS listens to those changes and sends them back to I am Loyalty via a webhook configured during installation

[Erik Andersson]: This makes I am Loyalty slightly special because despite being a "third-party" integration, it requires consent mapping infrastructure that other simple One API integrations don't need.

[Michal Rosikiewicz/Tomasz Kowalski]: I am Loyalty's parent company, Intermail, is heavily integrated with APSIS. They handle loyalty programs, points, and offers while APSIS handles marketing. The relationship is very collaborative.

### Join CX (formerly Join Loyalty)

**Status:** Third-party CRM/loyalty platform, actively maintained and expanding

**API model:** One API + Full **Generic Connector** implementation

**Key distinction:** Join CX has chosen to implement the complete Generic Connector protocol, including:
- Field mappings
- Sink configurations (what data flows in which direction)
- All Generic Connector features

[Erik Andersson]: Even though Join CX is labeled "third-party," they've fully embraced the Generic Connector architecture. This gives them and APSIS complete control and visibility over the data flow. They handle all their own priorities and product decisions, but the integration architecture is fully standardized.

**Migration note:** Intermail is migrating all I am Loyalty customers to Join CX, which will reduce the number of connectors APSIS needs to maintain.

---

## Native FSC Integrations

These are built and maintained by APSIS (FSC = Apsis One) team or are part of the core product.

### FSC 12.1 (New Generic Connector Implementation)

**Status:** Current, actively maintained

**Architecture:** Uses the Generic Connector protocol

**Scope:** The primary native CRM integration for FSC 12.1

### FSC Enterprise (Generic Connector)

**Status:** Active

**Characteristics:** Does not have a lead concept; only works with contacts. This simplifies the data model compared to systems like Tribe CRM.

### E-deal

**Status:** Active

**Architecture:** Uses Generic Connector

**Notable feature:** Has a lead entity configured and actually responds to form submissions with lead records

**Unique identifier handling:** E-deal sends back a `silhouette ID` when leads are created, which APSIS uses to track the lead in the system

### Web CRM

**Status:** Exists, future unclear

**Architecture:** Generic Connector

---

## Removed/Discontinued Integrations

[Erik Andersson]: Several integrations have been completely deleted from the codebase:

### Shopify
- **Status:** Completely removed from codebase
- **Reason:** License cost prohibitive
- **Story:** APSIS built a complete Shopify connector, even scheduling overtime to meet tight deadlines. Upon completion, legal discovered that using Shopify in this manner would require APSIS to pay Shopify **12% of annual APSIS revenue**. The entire connector was scrapped immediately.

### E-commerce (Generic)
- **Status:** Removed
- **Reason:** Budgetary constraints; the collaboration was scrapped
- **Note:** Shows as "Coming Soon" in some systems

### Facebook Lead Ads
- **Status:** No custom integration exists
- **History:** Integration team was asked to build a connector but declined. The request was then pushed to the WHat (What? unclear from transcript) team, which also declined. Eventually some hybrid approach was attempted where APSIS handled authentication and the other team handled logic. When that team was disbanded, the connector was abandoned.
- **Workaround:** Customers can use Zapier's Facebook Lead Ads trigger to funnel leads into APSIS

### Tessitura
- **Status:** Completely deleted

### ASCourse
- **Status:** Deleted

### Google Data Studio
- **Status:** Deleted

### Ambraco
- **Status:** Deleted

### Maxo (FSC Pro)
- **Status:** Discontinued
- **Note:** This is a discontinuation that "pains" Erik but is necessary

### Power BI
- **Status:** Marked for removal

---

## The Tribe CRM Lead Entity Webhook Bug

### Context: Tribe's Dynamic Entity Model

Tribe CRM has a unique characteristic: customers must manually select which entity to create when forms are submitted. There is no static "lead" or "contact" entity—customers choose from a dropdown what entity should be created when APSIS sends form submission data.

### The Generic Connector Configuration Problem

In the Generic Connector, APSIS has a configuration entity called **"lead"** for Tribe. This is a **virtual entity**—it doesn't represent a real Tribe entity, but rather represents "whatever entity the customer selected in the Tribe configuration dropdown."

**Why this matters:**

When APSIS needs to:
1. **Get mappable fields:** APSIS asks Tribe: "Give me the attributes for the lead entity." Tribe responds with attributes for whatever entity the customer selected.
2. **Set up outbound mappings:** APSIS displays these (dynamically named) fields so customers can map APSIS fields to Tribe fields.
3. **Receive form submissions:** Tribe sends back the actual entity name they selected (e.g., "potential_customer"), not the virtual name "lead."

### The Bug

During installation, APSIS registers webhooks for all entities in the Generic Connector configuration. Because APSIS had the virtual "lead" entity defined, it registered a webhook saying: **"Tribe, please send us updates for the lead entity."**

**What actually happened:**

1. Tribe never sends explicit "lead" updates (because "lead" isn't a real entity)
2. However, Tribe **misinterpreted** the webhook registration
3. Tribe began sending **contact updates to both the contact webhook endpoint AND the lead webhook endpoint** (duplicating the data)
4. APSIS received the contact data on the lead endpoint
5. Because APSIS expected lead data to have a different key space and different mappings, APSIS created a **brand-new empty profile** in the "lead" key space
6. The profile had a lead ID matching the contact ID, but was completely empty (wrong key space, no field mappings applied)

**Result:** For every real contact synced from Tribe, a parallel empty profile was created in the wrong key space.

[Tomasz Kowalski]: Can the parallel empty profiles be identified?

[Erik Andersson]: Yes. Empty profiles in the lead key space with only a silhouette ID are a guaranteed indicator of this bug occurring on that Tribe customer instance.

### The Fix

[Erik Andersson]: Added a parameter `skip_webhook_registration` to prevent webhook registration for virtual entities like Tribe's lead. During installation, the system now:
1. Extracts all entities from the connector configuration
2. Checks if webhooks should be created for that entity
3. Skips webhook registration for entities flagged with `skip_webhook_registration`

For Tribe specifically, no webhook is registered for the "lead" entity, preventing the duplicate data issue.

**Implementation location:** Generic Connector functions, specifically the webhook registration logic.

### Deeper Root Cause

[Erik Andersson]: The fundamental issue is Tribe's **fully dynamic entity model** combined with APSIS's need for static configuration:

- APSIS requires knowing which entity will be created so it can pre-configure field mappings and consent mappings
- Tribe allows this to be changed at any time in their configuration
- If a customer changes the selected entity in Tribe **after** setting up APSIS mappings, all configurations become misaligned
- There is no validation or synchronization between Tribe's selected entity and APSIS's configured entity

**The ideal solution (not yet implemented):** Force Tribe to always use the contact entity, eliminating the virtual lead entity concept entirely. This would require a discussion with Tribe product, and APSIS has a relationship with Rose, who is now a product owner at Tribe, providing a potential path forward.

[Erik Andersson]: Currently, everyone happens to select contacts as the creation entity, so configurations match by luck. However, this is fragile and could break if a customer selects a different entity.

### Implications for Existing Customers

[Erik Andersson]: Because this is a recent discovery, there may be other Tribe customers suffering from this same issue. The fix will apply only to new installations.

For existing customers, the SOC (support) team now understands the problem and can identify affected instances by looking for empty profiles in the lead key space.

### Why This Matters for Support Routing

[Erik Andersson]: This issue underscores the importance of routing all questions through the SOC and support team:
- They now have knowledge of the Tribe lead entity bug
- If similar issues arise with other customers, SOC can immediately identify and explain the root cause
- If questions bypass SOC and go directly to account managers or engineering, this knowledge sharing is circumvented
- Future questions about the same issue will require re-investigation instead of referencing the documented problem

---

## Integration Cleanup Work

### Current State

Several deprecated integrations remain visible in the APSIS One integration marketplace UI:
- Presta Shop (pre-launch, not released to customers)
- Adobe Commerce (pre-launch)
- Sitecore
- Drupal  
- Optimizely
- Tessitura (deleted but still showing?)
- ASCourse (deleted but still showing?)
- Google Data Studio (deleted but still showing?)
- Ambraco (deleted but still showing?)
- Power BI

### Why They're Still Visible

[Erik Andersson]: The integration UI consists of two parts:
1. **Backend list** (API endpoint returning available integrations): APSIS maintains a list determining which integrations can be "connected"
2. **Frontend UI cards:** Angular components displaying integration options

APSIS has cleaned up the backend list for many deprecated integrations, but the **UI card components remain**, and they still appear in pre-launch state.

**Mechanism:** The backend `integrations` list controls whether a card is clickable ("Connect" button enabled) or read-only ("Read More" state). Removing an integration from the backend list causes the card to show "Read More" instead of "Connect," but the card itself still renders.

### Cleanup Work Needed

[Erik Andersson]: Plans to create a Jira story to clean up remaining deprecated integrations:
- Remove from backend integration list
- Remove corresponding UI components (Angular card components)
- Ensure no false advertising that these integrations are available

[Michal Rosikiewicz]: This cleanup will be quick for the frontend team (Chris) to execute. Creating a story linking backend and frontend cleanup tasks is the right approach.

---

## Form Submission Data Flow and Lead Handling

### Standard Generic Connector Flow

When a customer sets up form submissions to sync to a Generic Connector CRM:

1. **Form submit trigger:** APSIS receives a form submission from a customer's form
2. **Payload construction:** APSIS constructs a request with form data
3. **Send to CRM:** APSIS sends this data to the CRM
4. **CRM response:** The CRM responds with information about what was created
5. **Profile creation:** APSIS uses the CRM's response to track the record

### E-deal Example (Straightforward)

[Erik Andersson]: E-deal responds to form submissions with **lead records**:

```
Request: Form submission with contact data
Response: {
  "newRecords": [{
    "recordID": "lead_123",
    "entity": "lead"
  }]
}
```

APSIS creates a profile with `silhouette ID = lead_123` in the lead key space. This profile is linked to the lead entity created in E-deal. When E-deal performs a full sync (downloading persons), APSIS also creates profiles for the actual person/contact records in the person key space.

**Direction:** Form submissions only flow from APSIS to E-deal. E-deal never sends webhook updates for new leads (only for changes to existing persons after full sync). E-deal can, however, send a delete request if they decide a lead is not worth pursuing.

**Webhook registration:** During installation, APSIS registers webhooks for all configured entities (persons, silhouettes). APSIS doesn't expect updates on silhouettes but has prepared for the case where E-deal sends a delete request.

### Tribe Example (Problematic)

Unlike E-deal, Tribe's response to form submissions is **unpredictable**:

1. **Configuration stage:** Customer logs into Tribe and selects which entity to create (e.g., "potential_customer") from a dropdown
2. **APSIS configuration:** APSIS has a virtual "lead" entity representing this selection
3. **Mapping stage:** APSIS asks Tribe for attributes of "lead"; Tribe returns attributes of "potential_customer"
4. **Form submission:** APSIS sends form data to Tribe
5. **CRM response:** Tribe responds with "potential_customer" entity (not "lead")

The problem: APSIS expects "lead" but gets "potential_customer." Since APSIS doesn't have a "potential_customer" configuration, it doesn't know how to handle it. The old behavior was to treat it as if it matched the "lead" configuration anyway, causing mismatches.

**Webhook registration issue:** Because APSIS registered a webhook for "lead," Tribe thought APSIS wanted all lead-related updates. Tribe then sent contact updates to the lead webhook endpoint (in addition to the contact endpoint), causing the duplicate/parallel profile bug.

---

## Key Architectural Principles for Integration Development

### Don't Build Plugins for CRM Systems

[Erik Andersson]: Building custom plugins for third-party CRM systems should be avoided unless absolutely necessary. Reasons:

1. **Support burden:** Once APSIS builds a plugin, APSIS owns support and bug fixes indefinitely
2. **Maintenance cost:** Every new platform feature must be enabled in the plugin—or customers can't use it
3. **Time sink:** Debugging CRM-specific issues requires deep knowledge of that system
4. **Business model misalignment:** Modern integrations should use partner ecosystems where vendors own their own integrations

**Exception scenario:** If Product explicitly decides a major customer needs a plugin integration, make sure decision-makers understand the cost implications before committing.

[Michal Rosikiewicz]: When Professional Services brings requests for plugin development, the preferred path is to find an external vendor who will build and own the plugin. If that's not possible, APSIS can consider building it, but someone in management (not the engineer) should make the decision with full cost awareness.

### Leverage the Generic Connector Protocol

[Erik Andersson]: When possible, design CRM integrations to use the Generic Connector protocol. It provides:
- Standardized field mapping
- Predictable entity handling
- Webhook management
- Consent mapping support
- Clear data flow semantics

This is better than ad-hoc One API usage because it standardizes and simplifies configuration.

### Design for Predictability, Not Maximum Flexibility

[Erik Andersson]: The Tribe lead entity issue illustrates why maximum flexibility (dynamic entity selection) creates implementation complexity:
- **Predictable:** "You always create contacts, we'll map contacts" → simple, reliable
- **Flexible:** "You can create any entity, we'll figure it out" → fragile, error-prone

When designing CRM integrations, favor static, predictable entity models.

---

## One API as the Universal Integration Layer

[Erik Andersson]: Almost all non-plugin integrations use One API as the data transport layer:
- Zapier app: 100% One API
- SuperOffice: One API for everything
- Sleeknote: One API for everything
- WooCommerce: One API
- I am Loyalty: One API + consent webhooks
- Join CX: One API + Generic Connector

One API is the standardized, maintained interface for all data exchanges between APSIS and third-party systems.

---

## Key Takeaways

1. **Plugin repositories are deprecated:** Apsis-integrations repositories for Presta Shop, Adobe Commerce, Sitecore, Drupal, and Optimizely are open source with zero support. Don't build plugins for third-party CRM systems unless absolutely necessary.

2. **Partner ecosystem model:** APSIS no longer owns support for CRM integrations. External vendors own their plugins and support them. APSIS benefits through data volume and new feature enablement.

3. **Zapier is actively maintained:** The Zapier integration is fully functional and used by many customers. It's the recommended path for customers needing ad-hoc integrations (e.g., Facebook Lead Ads).

4. **Tribe CRM lead entity bug fixed:** Virtual lead entities in Tribe caused webhook misregistration and empty profile creation. The fix skips webhook registration for virtual entities. SOC team now knows to watch for this pattern in other Tribe customers.

5. **Tribe's dynamic entity model is fragile:** Tribe allows customers to select any entity for form submissions. APSIS should push Tribe to standardize on a static entity (contacts) to eliminate the virtual entity concept.

6. **Third-party integrations are minimal effort:** SuperOffice, Sleeknote, WooCommerce, and others require only One API key bootstrap. After setup, everything flows through One API—APSIS's role is minimal.

7. **Join CX and I am Loyalty:** Join CX is the future (full Generic Connector). I am Loyalty is being sunset. Both are from Intermail, which is deeply integrated with APSIS.

8. **Cleanup needed:** Several deprecated integrations (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely, Tessitura, Power BI, etc.) should be removed from the UI marketplace. Frontend and backend cleanup stories needed.

9. **Generic Connector is the standard:** Modern CRM integrations should implement the full Generic Connector protocol (field mappings, sinks, consent mappings) rather than ad-hoc One API usage.

10. **Decision authority matters:** When asked to build a plugin or take on integration support, understand the cost implications and make sure the right people (Product, management) make the decision—not individual engineers.

---

## Unresolved Questions and Action Items

### Immediate Action Items

1. **Create Jira story for integration cleanup**
   - Owner: Erik Andersson
   - Tasks: Remove deprecated integrations from backend list; coordinate with frontend team (Chris) to remove UI cards
   - Integrations to remove: Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely, Tessitura, ASCourse, Google Data Studio, Ambraco, Power BI
   - Status: Not yet created as of meeting end

2. **Monitor Tribe customers for lead entity bug**
   - Owner: SOC/Support team
   - Task: Identify customers with empty profiles in lead key space; flag as known Tribe webhook bug
   - Status: SOC team briefed and understands the issue

3. **Investigate Tribe entity standardization**
   - Owner: Rose (Tribe product owner contact point)
   - Long-term goal: Push Tribe to always use contact entity, eliminate virtual lead entity
   - Status: Proposed but not yet prioritized

### Open Questions

1. **Are there other Tribe customers affected by the lead entity bug?**
   - Currently unknown; requires SOC monitoring
   - Can be identified by empty profiles in lead key space

2. **What is the future of Web CRM integration?**
   - Status: Active but unclear direction
   - No immediate changes proposed

3. **How many customers are still using legacy Drupal integration?**
   - Estimate: One old customer with customized version
   - No special support planned

---

## Additional Context and War Stories

### Lime Plugin Development Experience

[Erik Andersson]: Four years ago, attempted to build a Lime plugin to replicate Dynamics' email/website event retrieval feature.

**Challenges encountered:**
- Multi-tenant data isolation in Lime was complex
- Shared instances across multiple customer accounts made persistence difficult
- Easy to misconfigure and accidentally expose data across all customers
- Feature interest waned quickly
- Never achieved a production-ready solution

**Lessons learned:**
- Custom plugin development is time-consuming and risky
- Deep knowledge of the target CRM system is essential
- Multi-tenant architecture has subtle gotchas
- External vendor support (Lime provided test environments and documentation) helped but wasn't enough

**Current status:** Code preserved as architectural reference; plugin is non-functional.

### Shopify Connector Disaster

[Erik Andersson]: APSIS invested significant effort (including overtime) to build a complete Shopify connector. Upon completion, legal discovered licensing implications:
- Using Shopify in this manner requires paying Shopify **12% of annual APSIS revenue**
- Entire connector was immediately scrapped
- Years of work discarded

**Lesson:** Legal and business team review should happen **before** development, not after.

### Cold Storage Timeout Issue with Microsoft Dynamics

[Erik Andersson]: During lambda/API gateway architecture, API calls to Dynamics instances that hadn't been used recently would experience extreme delays:
- Microsoft Dynamics cold storage: instances not accessed for a time are moved to cold storage
- First API request to a cold instance could take **over a minute**
- API gateway timeout (typically 30 seconds) would trigger before the request completed
- Installation process failed on first attempt, succeeded on retry (after instance warmed up)

**Resolution:** Migrating to ECS tasks and load balancers eliminated this issue. ECS tasks can handle longer-running requests without strict timeout constraints.

### Facebook Lead Ads Ownership Confusion

[Erik Andersson]: Facebook Lead Ads became a contentious responsibility issue:
- Integrations team was asked to build a connector; declined
- Request was pushed to WHat team; they also declined  
- Eventually some hybrid approach emerged (APSIS handled auth, WHat team handled logic)
- When WHat team was disbanded, the connector was abandoned
- **Result:** No native Facebook Lead Ads connector; customers must use Zapier

**Lesson:** Clarify ownership and responsibility upfront to avoid handoff failures.

### E-commerce Collaboration Cancelled

[Erik Andersson]: APSIS started building e-commerce integrations but the collaboration was scrapped due to budget constraints. No connector exists today despite the "Coming Soon" status remaining visible in some systems.