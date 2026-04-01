---
source_file: Erik - Types of Connectors.txt
domain: Apsis One Integrations
topics: [connector types, legacy connectors, generic connector, third-party connectors, installer options, outbound mappings, consent synchronization, field mappings, CRM integration architecture]
speakers: ["Erik Andersson (outgoing integration domain expert)", "Lukasz Grabowski (incoming team)", "Michal Rosikiewicz (incoming team)", "Shreevidhya Ganesan (Apsis, context provider)"]
key_components: [Justin platform, generic connector, legacy connectors, FSC Enterprise 12.0, FSC Enterprise 12.1, Microsoft Dynamics, Lime CRM, Tribe, E-deal, Web CRM, Intermail, Super Office, Siteshop, Join CX, WooCommerce, Intermail Loyalty, one API, audience system]
session_type: knowledge-transfer
---

## Session Overview

This session is the first structured knowledge transfer from Erik Andersson (outgoing integration expert) to the incoming team (Lukasz and Michal). Erik covers the three distinct types of connectors in the Apsis One integrations platform: legacy connectors, the generic connector, and third-party connectors. He explains the historical evolution and rationale behind each approach, describes installer/connector options configuration, and introduces the concept of outbound mappings. The session concludes before reaching the code repository walkthrough, which was deferred to a subsequent session.

---

## Platform Framing: Justin and the Integrations Domain

The integration platform is called **Justin**. Every connector discussed in this session exists within Justin — connectors are installed inside Justin. The integrations domain is described as broad in terms of the number of services, but individually those services are not algorithmically complex:

> "Integration in its nature is a lot of syntactical sugar on top of Audience in order to handle messages in order [of precedence]."

A key example of the non-trivial logic that does exist: if a CRM sends a consent update at the same time that Apsis receives a consent update, there is logic to ensure the **latest update** is the one sent into Audience — to prevent overriding a newer consent with an outdated one.

The integration domain does **not** contain complex algorithms or mathematical structures like the Audience system does.

---

## Connector Type 1: Legacy Connectors

### Overview and History

**Legacy connectors** were the original approach, built in-house starting around late 2018. For each CRM (e.g., Microsoft Dynamics), Apsis engineers:
- Consumed the CRM's existing APIs directly
- Extracted required data (contact fields, consent state, etc.)
- Handled all necessary data transformations manually

Example transformation required: Apsis requires `true`/`false` for booleans; Microsoft Dynamics uses `1`/`0`.

### The Maintenance Problem

The fundamental flaw of the legacy approach: **every new Apsis feature had to be replicated separately for each connector**. For example, adding the ability to sync email events from Apsis to the CRM required separate implementations for Microsoft Dynamics, Lime CRM, FSC Enterprise, etc. With even four integrations, this became unsustainable — one new feature could take months of work, and ongoing maintenance was described as an "absolute nightmare."

### Current Connectors Still on Legacy

| Connector | Notes |
|---|---|
| Microsoft Dynamics | Significant number of customers (~60 estimated across legacy connectors total) |
| FSC Enterprise 12.0 | On-premise; large enterprise customers resistant to migration; still sold in some cases |
| Lime CRM | 1–2 customers; relatively small but paying well; still requires maintenance |

### Policy Going Forward

> "We have since quite long said we are not developing any new legacy connectors. We are also not adding any new features."

The only exception would be an extraordinary commercial case (e.g., a very large customer offering significant payment for a specific feature) — this has not occurred.

**New customers should always use the generic connector.** The team should firmly decline requests to build new legacy-style connectors.

---

## Connector Type 2: The Generic Connector

### Core Concept and Motivation

The **generic connector** was introduced approximately **summer 2022** (confirmed by Shreevidhya). The motivating factor was the growing FSC Enterprise customer base and the mounting maintenance burden across multiple CRM-specific connectors.

The fundamental design shift: **responsibility for data format compliance moves from Apsis to the CRM vendor**.

Apsis exposes one standardized connector (literally called "generic connector") with defined endpoints. Every CRM that wants to integrate must implement those endpoints and conform to the Apsis-defined data contract. The only thing that differs between CRM integrations is the hostname.

Example endpoint pattern:
```
/records/{page_number}
```
Apsis calls this identically for Tribe, E-deal, Web CRM, etc. All are expected to respond with identical data formats.

### Data Contract Enforcement

The generic connector enforces strict typing. If a field is declared as an integer, the CRM **must** send an integer — not a stringified integer. Booleans must be booleans. This eliminates the class of bugs that plagued legacy connectors.

> "All this responsibility resides in the CRM system. Apsis does not need to do any workaround in data manipulation."

This is a direct contrast to the legacy FSC Enterprise connector, which sent:
- Default dates of `1899-12-xx` for empty date fields (valid dates that triggered age-based MA flows making all customers appear ~100 years old)
- Stringified integers in integer fields
- Stringified `"1"`/`"0"` in boolean fields

### Feature Flags / Capability Negotiation (No Formal Versioning)

Each CRM implementation communicates to Apsis **which features it supports** via a capability list (e.g., `can_sync_email_activity`, `can_sync_sms_activity`, `can_sync_event_tool_activities`). Apsis retrieves this list from the CRM and uses it to enable or disable options in the Apsis UI.

- Apsis will not display a sync option to the customer until the CRM has declared support for it
- Apsis will not send data to the CRM for an unsupported feature
- This operates at the **individual CRM instance level** — a development instance may support a feature that the production instance does not

There is **no formal API versioning** of the generic connector. New features have only been added, not removed or contract-breakingly modified, so versioning has not been necessary so far.

[Michal Rosikiewicz]: Asked whether feature flags constitute versioning.
[Erik Andersson]: Characterized it as capability negotiation rather than versioning; acknowledged it hasn't been formalized.

### Per-CRM Customization

There is **no per-CRM customization** in the generic connector on the Apsis side. All CRM-specific differences (entity naming, field naming, ID field naming) are handled through **installer options / connector options** configuration (see section below), not in code.

When a feature is built for Tribe, it is automatically available to Dynamics, E-deal, and Web CRM — they simply choose when to enable it.

### CRMs Currently on Generic Connector

- Tribe
- E-deal
- Web CRM (migrated from a legacy Web CRM connector; full migration was possible because Web CRM is SaaS — no on-premise instances)
- FSC Enterprise 12.1 (new version built by FSC to support generic connector)
- Join CX (Intermail's connector — see third-party section)

### FSC Enterprise 12.0 vs 12.1

**FSC Enterprise 12.0** = legacy connector (built by Apsis, all the data transformation problems described above).

**FSC Enterprise 12.1** = generic connector implementation (FSC Enterprise built support themselves). The difference is dramatic — no data transformation required on the Apsis side.

However, ~30 Enterprise customers remain on 12.0 and are resistant to migrating. The 12.0 legacy connector must still be maintained. It cannot simply be hidden because:
1. Some customers may still legitimately purchase FSC 12.0 (on-premise)
2. New-to-Apsis customers on 12.0 need a way to connect

> "The dream state is of course that every old one is migrated. I would highly suggest that this is pushed for in tandem with consultancy because it's consultancy that typically handles the actual migration because they sit with the customer and their data."

**Web CRM** is the example of a completed migration: the legacy Web CRM connector was hidden once all customers were on the new version (SaaS made this possible). The old connector is still accessible to customers who had it installed (to access their existing configuration) but is not offered to new customers.

### Known Bug

Outbound mappings are not currently visible in FSC Enterprise 12.1 in the UI, but they should be. This is a known bug to be investigated after the handover period.

---

## Connector Type 3: Third-Party Connectors

### Overview

**Third-party connectors** are integrations built and maintained entirely by external companies. Apsis's role is minimal: essentially UI-level installation support only.

When a third-party connector is installed, Apsis:
1. Creates the **key space** for that integration
2. **Whitelists** the CRM ID attribute for that key space
3. Generates a **one API key**

The customer copies this one API key and pastes it into the third-party vendor's own configuration UI.

Apsis does **not**:
- Know the internal logic of how the integration works
- Control what data is synced or how
- Add features to these connectors
- Support data sync issues (those go to the third-party vendor)

[Erik Andersson]: "This is to a large extent a relic from when Jonas was product owner for Apsis and he tried to sell this to other companies. Hopefully you will not need to bother much about this at all."

### Known Third-Party Connectors and Customer Counts

| Connector | Approximate Customers | Notes |
|---|---|---|
| Super Office | ~5–6 | Pure third-party |
| Intermail Loyalty | ~1 | Being discontinued |
| Join CX (Intermail) | ~2–3 | Has full generic connector support (see below) |
| Lead Family | A couple | Pure third-party |
| WooCommerce | ~3–4 | Pure third-party |

Total third-party customers: fewer than 10 across all connectors.

### Intermail / Join CX Exception

**Intermail's Join CX connector** is a hybrid case: it is not an Apsis connector, but Intermail has built **full generic connector support** into it. This means it behaves like a proper generic connector installation — field mappings, consent mappings, full configuration in the Apsis UI. The customer enters the key into the Apsis UI, and then Apsis code makes connections to the CRM, sets up webhooks, and uses that key for requests.

**Intermail Loyalty** (distinct from Join CX) is a pure third-party connector: customers get a one API key, paste it in Intermail's UI, and Apsis has no further involvement. Its use case is syncing loyalty points as profile attributes in Apsis, which can then be used in MA flows.

---

## Commercial Model Considerations: Who Owns the Connector

[Erik Andersson] gave important guidance on the preferred commercial model for future connectors, using Siteshop as the reference example:

**Undesirable model (e.g., old Microsoft Dynamics):** Apsis paid a third party (CRM Consult) to build the connector. Apsis owns the connector, charges customers for it, and therefore owns all support and maintenance costs. Bug fixes cost approximately **2,500 SEK/hour** via the development partner.

**Preferred model (e.g., Siteshop):** The CRM/partner company owns the customer relationship. The partner charges the customer for connector use. In exchange:
- Apsis gains profile imports and email sends (revenue from usage)
- The partner owns all maintenance and customer support
- If there are data sync issues, the partner handles them first; they may contact Apsis only for log-level assistance

> "It is more important to find someone who wants to build a connector to Apsis and also own the customer and the support, and in exchange Apsis will get the cost for Apsis — whatever the customers are using — because the amount of time you can spend on debugging and supporting this, and the cost you need to spend to fix these bugs, can eat up any revenue you had quite fast."

**Guidance for the incoming team:** Do not take it upon yourself to build new integrations. If asked, the answer is: build endpoints for the generic connector, and it will work. R&D's role is configuration support and assistance — not ownership of new connector implementations. This is a product-level decision.

---

## Installer Options / Connector Options Configuration

Each CRM connector is configured via what Apsis calls **installer options** (also referred to as **connector options**). This is where all per-CRM differences are encoded in configuration rather than code.

Key configuration fields include:

- **Connector display name** — what users see in the UI
- **Integration ID** (logical ID) — distinct from the display name; critical for internal identification
- **Feature flags (global kill switches)** — e.g., if `can_sync_event_tool_activities` is hardcoded `false` at the connector level, Apsis won't even query the CRM instance for support. Use case: if no version of an FSC Enterprise installation has ever supported event tool, this can be disabled globally.
- **Profile entity name** — the main entity in the CRM representing customers (e.g., "contacts" in Dynamics, "persons" in Tribe)
- **ID field name** — the unique identifier field used as the key in Apsis's key space (e.g., `contact_id`, `person_id`). Must be unique per record. This is the key used to prevent duplicate profile creation and to correctly match updates.
- **Concurrency / page size** — number of threads and page sizes used during full sync (downloading contacts from CRM)
- **Inbound enabled** — whether Apsis requests data from the CRM (full sync direction: CRM → Apsis)
- **Outbound enabled** — see dedicated section below

Configuring a new CRM is described as straightforward — mostly copy-paste from an existing configuration with adjustments for entity names and ID fields.

---

## Consent Synchronization: Bidirectional by Design

Consent is the **only data that flows bidirectionally** between Apsis and the CRM. All other attribute data is unidirectional (CRM → Apsis).

### Why Consent Must Be Bidirectional

The CRM is the authoritative source for contact attributes. However, consent changes can originate in Apsis through:
- A recipient clicking an unsubscribe link in an email
- A GDPR erasure/opt-out request handled via Apsis support

If consent changes in Apsis were not synced back to the CRM, the next sync from the CRM would overwrite the Apsis opt-out with the CRM's opt-in state, resulting in sending communications to someone who has explicitly unsubscribed — a legal/compliance violation.

> "That would put this in a lot of trouble."

### How Consent Sync Is Triggered

Consent sync is governed by **subscription mappings** (configured per integration). As soon as a mapping from an Apsis subscription (e.g., "default email subscription") to a CRM consent field is set up, Apsis registers a listener for consent changes. Any change to a mapped subscription in Apsis is sent to the CRM — this is independent of forms, MA, or any other flow.

### Rule: Attribute Changes Are NOT Synced to CRM

If a contact attribute is modified within Apsis (not via a form submit), it is **never** synced back to the CRM. The CRM is the master record for attributes. The only exception is the outbound/form flow described below.

---

## Outbound Mappings and Outbound Enabled Flag

### What "Outbound" Means in This Context

Despite the name potentially causing confusion, **outbound** here refers specifically to:
1. **Form tool submissions** — when a visitor submits an Apsis form, that data can be forwarded to the CRM as a lead/opportunity
2. **MA task nodes** — when a profile enters a specific node in a Marketing Automation flow (covered in a separate MA session)

It does **not** mean continuous attribute sync from Apsis to CRM.

### The Problem Outbound Mappings Solve

When a form is submitted, Apsis sends the submit event to the CRM. But the CRM needs to know: which field in the form corresponds to "first name" in my lead object? If the form field is named "Eric's cool field" or "sailing boats," the CRM cannot infer the semantic meaning without a mapping.

**Outbound mappings** allow the customer to define: "this Apsis attribute maps to the `first_name` field in the CRM's lead/silhouette entity."

### How It Works

1. A customer creates a form in Apsis and maps form fields to Apsis attributes (standard form configuration)
2. In the integration settings, the customer configures outbound mappings: Apsis attribute X → CRM lead field Y
3. When the form is submitted:
   - Apsis always sends the raw submit event to the CRM
   - If **outbound enabled = true**, Apsis additionally extracts attribute values from the resulting profile and constructs a structured object mapping them to the CRM's expected field names, sending this as an additional property alongside the submit event

### Outbound Enabled Flag Behavior

- If **outbound enabled = false**: submit event is sent; CRM must infer field semantics itself
- If **outbound enabled = true**: submit event is sent **plus** the enriched attribute mapping object
- The outbound mappings UI is available regardless of this flag; the flag controls whether the enrichment is actually performed at send time
- **Delta sync does not apply here** — this is event-driven, not periodic

### FSC Enterprise 12.0 vs 12.1 for Outbound

Outbound mappings are **not supported** in FSC Enterprise 12.0. They **are** supported in 12.1 — this was in fact one of the primary motivations for building 12.1 with generic connector support. (There is a known UI bug where outbound mappings are not currently visible in 12.1.)

---

## Action Item: Event Tool Feature Flag Removal

During the session, a discussion arose about how the event tool's "Sync to CRM" option is displayed in the Event Tool UI.

**Current state:** The option is gated behind both a feature flag (at the account level) and the `can_sync_event_tool_activities` capability flag returned by the integration endpoint.

**Problem identified:** Account-level feature flags are too coarse. A customer may have one section (production) where event tool syncing is not supported, and another section (development) where it is. The account-level flag cannot distinguish between them.

**Recommendation from Erik:** Remove the account-level feature flag entirely and rely solely on the `can_sync_event_tool_activities` value returned by the integration endpoint (which is per-installation, not per-account). The feature flag appears to be a leftover from development time.

[Lukasz Grabowski]: "I will create a story to remove it entirely and rely only on this endpoint."

---

## Testing Notes

Unit tests should run locally, but many tests depend on AWS services (ECS tasks, Secrets Manager) and may fail locally while succeeding in the CI/CD pipeline. This is a known issue described as "not the best design." Further investigation deferred.

---

## Key Takeaways

1. **Three connector types exist**: legacy (built by Apsis, in maintenance mode only), generic (the standard going forward), and third-party (external ownership, minimal Apsis involvement)
2. **The generic connector inverts responsibility**: CRMs must conform to Apsis's data contract, eliminating all data transformation code on the Apsis side
3. **Never build a new legacy-style connector**. If asked, direct CRM vendors to implement the generic connector specification
4. **Consent is the only bidirectional data** — all attribute data flows CRM → Apsis only; consent flows both ways due to legal requirements
5. **Outbound mappings** enable CRMs to construct better leads from form submissions by providing a semantic mapping from Apsis attributes to CRM lead fields
6. **FSC Enterprise 12.0** must be maintained indefinitely due to on-premise installations; 12.1 uses generic connector and is the migration target; migration should be driven by consultancy
7. **The preferred commercial model** for new connectors is partner-owned (like Siteshop): partner owns the customer, support, and maintenance; Apsis benefits from usage revenue
8. **Installer options** are the mechanism for per-CRM configuration — entity names, ID fields, concurrency, feature support — without any code changes

---

## Unresolved Questions and Action Items

1. **[Action - Lukasz]** Create a story to remove the account-level feature flag for Event Tool CRM sync and rely solely on the `can_sync_event_tool_activities` capability returned by the integration API endpoint
2. **[Action - Erik, post-handover]** Investigate why outbound mappings UI is not visible for FSC Enterprise 12.1 (known bug)
3. **[Deferred to next session]** Code repository structure walkthrough — where are important folders, how to navigate the codebase
4. **[Deferred to next session]** Full inbound sync flow walkthrough
5. **[Deferred to separate session]** Setting up a new generic connector from scratch (Erik offered to do a live walkthrough using a pretend CRM scenario)
6. **[Deferred to MA session]** How MA task nodes trigger outbound sync to CRM
7. **[Open]** Exact customer counts for legacy connector installations — Erik estimated ~60 total but did not have precise figures