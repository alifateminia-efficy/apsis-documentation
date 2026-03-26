---
source_file: Erik - Types of Connectors.txt
domain: Apsis One Integrations
topics: [connector types overview, legacy connectors, generic connector, third-party connectors, connector history and rationale, installer options, outbound mappings, field mappings, consent synchronization, lead creation via forms]
speakers: ["Erik Andersson (outgoing engineer/domain expert)", "Lukasz Grabowski (incoming team)", "Michal Rosikiewicz (incoming team)", "Shreevidhya Ganesan (integration team)"]
key_components: [Justin platform, Microsoft Dynamics connector, FSC Enterprise 12.0, FSC Enterprise 12.1, Lime CRM, Tribe, E-deal, Web CRM, Intermail, Super Office, Siteshop, WooCommerce, Join CX, Lead Family, Audience, generic connector, one API]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

Erik Andersson led a knowledge transfer session covering the three types of CRM connectors in the Apsis One integrations platform (Justin): legacy connectors (built in-house), the generic connector (a standardized interface the CRM developers implement), and third-party connectors (externally built and maintained). Erik explained the historical rationale for moving away from legacy connectors toward the generic connector model, including the data quality and maintenance problems that motivated the shift. The session also covered installer/connector options configuration, the concept of outbound mappings for enriching form-submit lead data sent to CRMs, and the bidirectional nature of consent synchronization. The session concluded before covering the code repository structure, which was deferred to a follow-up session.

---

## The Justin Platform and Integration Service Overview

The team/platform name distinction is important:
- The **team name** is "Integration"
- The **platform name** is **Justin**

All connectors are installed and managed inside Justin. Erik described the integration service as relatively straightforward in nature compared to, for example, the Audience system:

> "Integration in its nature is lots of syntactical sugar on top of Audience in order to handle messages — to make sure that if the CRM sends a consent update and we at the same time retrieve a consent update, we have logic to make sure the latest update is the one which will be sent into Audience, because otherwise we might override with an outdated consent."

The service involves many distinct services/components with simple individual behaviors, rather than complex algorithms.

---

## The Three Types of Connectors

### Type 1: Legacy Connectors

**Legacy connectors** were built in-house by Apsis starting around late 2018. The approach involved:
- Directly consuming the CRM's own APIs
- Performing custom data transformation to accommodate differences between Apsis's data formats and the CRM's formats

**Example of the data transformation problem:**
> Apsis One requires `true`/`false` for booleans, but Microsoft Dynamics uses `1`/`0`. FSC Enterprise sent stringified integers, stringified booleans (`"1"`, `"0"`), and had quirks like defaulting empty date fields to `1899-12-xx`, which caused MA flows to treat all such customers as ~125 years old.

**Existing legacy connectors:**
- **Microsoft Dynamics** — significant customer base (~60 customers estimated across Dynamics and FSC 12.0)
- **FSC Enterprise 12.0** — on-premise version; large enterprise customers resistant to migration
- **Lime CRM** — one or two customers; maintained because they pay well

**Key policy:**
> No new legacy connectors are being developed. No new features are being added to existing legacy connectors. The only exception would be an extraordinarily large customer paying a substantial fee — which has not occurred.

**Maintenance cost warning:** Fixing the old Microsoft Dynamics plugin costs approximately **2,500 SEK/hour** through the external development partner. This is not sustainable at scale.

---

### Type 2: The Generic Connector

The **generic connector** was introduced approximately **summer 2022** (around the time of FSC Enterprise's growing adoption).

**Core philosophy shift:** Instead of Apsis adapting to each CRM's data structure, the CRM developers adapt to Apsis's standardized interface.

- There is exactly **one connector in Apsis** called the generic connector.
- Apsis defines and maintains the API contract (endpoints, data formats, field types).
- CRM developers implement the expected endpoints on their side.
- All CRMs are called **identically** — only the hostname differs.

**Example endpoint pattern:**
```
/records/{page_number}/...
```
Called the same way for Tribe, E-deal, Web CRM, etc.

**Data contract enforcement:**
- If a CRM declares a field as integer, the contract requires them to send an actual integer — not a stringified integer.
- Apsis performs **no data manipulation** or workarounds for individual CRMs in the generic connector.
- All data transformation responsibility belongs to the CRM.

**Feature flag / capability advertisement:**
- Each CRM tells Apsis which features it supports (e.g., `can_sync_email_activities: true`, `can_sync_sms_activities: true`, `can_sync_event_tool_activities: true`).
- Apsis only displays/enables options in the UI when the CRM has declared support.
- This is per-installation: a CRM's development instance may support a feature that their production instance does not yet support.
- There is **no formal versioning** of the generic connector contract so far, because features have only been added, not removed or breaking-changed.

**Debugging advantage:**
> "If the generic connector flow works for Tribe but behaves weirdly for E-deal, it's easy to conclude the problem exists in E-deal — because it works as expected in Tribe or Web CRM."

**Current generic connector CRM implementations:**
- Tribe
- E-deal
- Web CRM
- FSC Enterprise 12.1 (new version of Enterprise with generic connector support)
- Join CX / Intermail (full generic connector support, though third-party owned)

**Adding new features:**
> "If Tribe requests a new feature and we build support for it in the generic connector, we have automatically built that feature for Dynamics and Web CRM as well. They may not have added support for it yet, but they can do so when they choose."

**Policy for new connectors:**
> [Erik Andersson]: "Every new connector should absolutely be a generic connector implementation. You should, in all honesty, refuse to do it otherwise."

If an external company approaches Apsis wanting a connector, the answer should be: **"Build the endpoints for the generic connector — the connector is already ready on our side."**

---

### Type 3: Third-Party Connectors

**Third-party connectors** are integrations built and maintained entirely by external companies. Apsis's role is minimal:

**What Apsis does when a third-party connector is installed:**
1. Creates the **key space** for that CRM
2. **Whitelists** the CRM ID attribute for that key space
3. Generates a **One API key**
4. Displays the One API key in the UI for the customer to copy into the third-party's configuration page

**What Apsis does NOT do:**
- Control the sync logic
- Add new features
- Debug or support sync issues (since Apsis doesn't know how they work internally)
- Call the third-party system to retrieve data

The third-party connector contacts Apsis via the **One API** using the generated key.

**Known third-party connectors and approximate customer counts (at time of session):**
- **Super Office** — ~5–6 customers
- **Intermail Loyalty** — 1 customer; being discontinued
- **Join CX** — 2–3 customers
- **Lead Family** — a few customers
- **WooCommerce** — ~3–4 customers

> [Erik Andersson]: "Hopefully you will not need to bother much about this at all."

**Exception — Intermail's Join CX connector:**
Intermail has also built a connector that implements the **full generic connector spec** (not just third-party One API usage). This connector works like any other generic connector installation, with field mappings, consent mappings, etc. However, Apsis does not control or maintain the Intermail side.

**Historical context:** Third-party connectors were promoted during a period when Jonas was product owner for Apsis and was seeking to expand the partner ecosystem. They are largely a legacy artifact.

---

### Legacy Migration: Web CRM as a Success Case

Web CRM went through the full journey:
1. Original Apsis-built legacy connector
2. Web CRM built generic connector support
3. Because Web CRM is **SaaS** (no on-premise instances), **all customers could be migrated**
4. The old legacy connector was **hidden** once all customers were migrated to the new version (still accessible for existing installations, but not offered to new customers)

**Contrast with FSC Enterprise 12.0:** On-premise customers cannot easily be forced to upgrade to 12.1, so both connectors must remain available.

---

### Ownership and Commercial Model — Important Guidance

**Preferred partnership model (Siteshop example):**
- Siteshop owns the customer relationship and charges the customer for the integration
- Siteshop owns maintenance and support for their connector side
- Apsis benefits indirectly through profile and email send volume from imported customers
- Apsis does **not** handle connector-side customer support cases

**Avoid the legacy model (old Dynamics connector):**
- Apsis paid CRM Consult to develop the connector
- Apsis charges customers for its use
- Apsis owns all support and maintenance responsibility
- Maintenance cost: ~2,500 SEK/hour to external development partner — unsustainable

> [Erik Andersson]: "It is more important to find someone who wants to build a connector to Apsis and also own the customer and the support. In exchange, Apsis gets the usage revenue. The amount of time you can spend on debugging and supporting this, and the cost to fix bugs, can eat up any revenue quite fast."

> [Erik Andersson]: "This should not fall on R&D. This is a product question. Do not take it upon yourself to start building actual integrations — you will be stuck in maintenance hell. Find someone who wants to build an implementation for the generic connector and support it."

---

## Connector / Installer Options Configuration

Each connector installation in Justin is configured via **installer options** (also called **connector options**). Key configuration fields include:

- **Display name** of the connector
- **Integration ID** (logical ID — distinct from the display name; critical for system identification)
- **Profile entity name**: the main entity in the CRM representing customers (e.g., "contacts" in Dynamics, "persons" in Tribe)
- **ID field name**: the unique identifier field on CRM records used as the key in Apsis (e.g., `contact_id`, `per_id`). This is the key space key used to identify and update profiles without creating duplicates.
- **Capability flags** (e.g., `can_sync_activities`): can be hard-disabled at the connector level in Apsis if a CRM has no support at all, so Apsis won't even query the CRM instance about it
- **Concurrency / page size**: configures how many threads and how large pages are used during full sync downloads
- **Inbound enabled**: whether Apsis downloads contact records from the CRM
- **Outbound enabled**: whether Apsis sends form/MA submit events to the CRM (see section below)

Each CRM also has its own "touch" on entity naming, ID field names, etc., which must be configured per integration. However, new generic connector configurations can largely be copy-pasted from existing ones with minor adjustments.

---

## Data Sync Direction and the Special Case of Consent

**General rule — CRM is the source of truth for attributes:**
- Data flows from CRM → Apsis
- Attribute changes made in Apsis are **not** synced back to the CRM
- If you want to change a profile's attributes, change them in the CRM

**Exception — Consent is bidirectional:**
Consent must be kept in sync in both directions because:
- End customers can opt out via the Apsis unsubscribe link in an email
- If that opt-out is not pushed to the CRM, the next full sync would overwrite the Apsis opt-out with the CRM's opt-in, causing Apsis to resume sending to someone who explicitly unsubscribed — a legal/compliance risk

> [Erik Andersson]: "The consent is the only thing which is bidirectional. We sync data from the CRM to Apsis. We sync the consent from Apsis to the CRM."

**How consent sync is triggered:**
- If a **subscription mapping** is configured (mapping a CRM consent field to an Apsis subscription), Apsis registers a listener for subscription changes
- Any change to that subscription in Apsis is sent to the CRM
- This happens regardless of any form or outbound settings

---

## Outbound Mappings and Lead Creation via Forms

### What Outbound Means in This Context

**Outbound** refers to Apsis sending data *to* the CRM — specifically:
1. **Form tool submissions**: When a contact submits an Apsis form, the submit event (and enriched profile data) is sent to the CRM to create a lead/opportunity
2. **MA task nodes**: When a profile enters a task node inside a Marketing Automation flow (to be covered in a separate MA session)

> Note: "Outbound" does **not** mean syncing attribute changes from Apsis to the CRM. That never happens. Outbound only applies to form submit events and MA task triggers.

### The Problem Outbound Mappings Solve

Without outbound mappings, Apsis sends the raw form submit event to the CRM. The CRM then has no reliable way to know which form field corresponds to which CRM lead field — especially when form fields have custom names (e.g., "Eric's cool field", "sailing boats").

**Outbound mappings** let the customer (on the integration configuration page) define:
> "This Apsis attribute = first name in the CRM lead structure"
> "This Apsis attribute = last name in the CRM lead structure"

When a form is submitted:
1. Apsis resolves the profile's attributes (populated via the form's field-to-attribute mapping)
2. Apsis constructs a JSON object mapping those attribute values to the CRM's expected lead field names
3. This is sent to the CRM as an **additional property** alongside the submit event

The CRM can then create a fully populated lead record (first name, last name, email, etc.) rather than having to guess field meanings.

### Outbound Enabled Flag

- If **outbound is disabled**: Apsis sends the raw submit event only; CRM must infer field meanings
- If **outbound is enabled**: Apsis additionally sends the enriched mapped attribute object

The form sync itself (selecting "Sync to CRM" on a form) triggers Apsis to register listeners for all events connected to that form. The outbound mappings only affect *how much additional data* is sent with those events.

### FSC Enterprise 12.0 vs 12.1

- **FSC Enterprise 12.0**: Does not support outbound mappings (outbound mapping UI is not shown)
- **FSC Enterprise 12.1**: Should support outbound mappings — the outbound mapping was one of the primary drivers for building the generic connector for Enterprise. **Known bug at time of session**: outbound mapping UI is not visible for 12.1 installations despite being theoretically supported. Erik flagged this as something to investigate after completing handover.

---

## Event Tool CRM Sync and Feature Flag Cleanup

During the session, a side discussion revealed a likely leftover feature flag:

- The Event Tool has a **feature flag at the account level** to control whether the "Sync to CRM" checkbox is visible
- Erik recommended removing this flag entirely and relying solely on the `can_sync_event_tool_activities` flag returned by the integration endpoint
- Rationale: The account-level flag is too coarse — a customer may have one section/installation where the CRM supports event tool sync and another where it doesn't. The per-installation capability flag from the CRM is the correct control mechanism.

> [Lukasz Grabowski]: "I will create a story to remove it entirely and rely only on this endpoint."

---

## Key Takeaways

1. **Three connector types**: Legacy (Apsis-built, in-house), Generic (standardized spec, CRM-implemented), Third-party (externally built, One API only). All new connectors must be generic connector implementations.

2. **Legacy connectors are frozen**: No new features, no new legacy connectors. Existing ones (Dynamics, FSC 12.0, Lime CRM) are maintained only for existing customers.

3. **Generic connector shifts responsibility to the CRM**: Apsis defines the contract; CRMs implement it. No data transformation in Apsis. Feature availability is controlled by per-installation capability flags from the CRM.

4. **Third-party connectors are minimal**: Apsis only creates a key space, whitelists the CRM ID attribute, and generates a One API key. All sync logic is external and unsupported by Apsis.

5. **Consent sync is always bidirectional** (when subscription mappings are configured). All other attribute changes are CRM → Apsis only.

6. **Outbound mappings enrich form-submit lead data** sent to the CRM by mapping Apsis attributes to CRM lead field names, enabling the CRM to create fully populated lead records.

7. **Commercial model recommendation**: New connector partnerships should follow the Siteshop model — the CRM/partner company owns the customer, maintains the connector, and handles support. Apsis should not own connector maintenance.

8. **FSC Enterprise 12.1 outbound mapping visibility bug** needs investigation post-handover.

9. **Event Tool account-level feature flag** is a leftover from development and should be removed; rely solely on the `can_sync_event_tool_activities` capability flag from the integration endpoint.

---

## Unresolved Questions and Action Items

- [ ] **Bug**: Outbound mappings UI not visible for FSC Enterprise 12.1 installations — Erik to investigate after handover is complete
- [ ] **Cleanup**: Remove the account-level feature flag for Event Tool CRM sync; rely only on the per-installation `can_sync_event_tool_activities` flag returned by the integration endpoint (Lukasz to create story)
- [ ] **Follow-up session needed**: Code repository structure — where are the important folders, how to navigate the codebase (deferred from this session)
- [ ] **Follow-up session needed**: Full outbound flow in detail — exact log inspection of what the CRM request looks like with/without outbound mappings enabled
- [ ] **Follow-up session needed**: MA task node outbound flow
- [ ] **Follow-up session needed**: Inbound flow (full sync, delta sync)
- [ ] **Follow-up session needed**: Setting up a new generic connector configuration end-to-end (Erik offered to simulate a new CRM onboarding)
- [ ] Erik to connect the incoming team with relevant partner/CRM teams so they know who they will be working with