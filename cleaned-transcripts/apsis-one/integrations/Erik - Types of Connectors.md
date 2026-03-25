---
source_file: Erik - Types of Connectors.txt
domain: Apsis One Integrations
topics: [Connector Architecture, Legacy Connectors, Generic Connector, Third-Party Integrations, Configuration and Installation Options, Outbound Data Flow, CRM Integration Patterns]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Shreevidhya Ganesan]
key_components: [Justin (Integration Platform), Legacy Connectors, Generic Connector, Third-Party Integrations, Installer Options, Field Mappings, Consent Synchronization, Outbound Mappings]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Outbound Flow]
---

## Session Overview

This knowledge transfer session covers the three primary types of connectors within Apsis One's Justin integration platform: legacy connectors (built in-house for specific CRMs like Microsoft Dynamics, Lime, and FSC Enterprise 12.0), the generic connector (a standardized approach that shifts responsibility to CRM partners), and third-party integrations (managed externally by partner companies). Erik Andersson explains the historical evolution from maintenance-intensive legacy connectors to the sustainable generic connector architecture introduced around summer 2022. The session includes detailed discussion of installer options, configuration requirements, and the critical distinction between inbound (CRM to Apsis) and outbound (form submissions and consent changes to CRM) data flows.

---

## Part 1: Types of Connectors Overview

### Historical Context and Problem Statement

[Erik Andersson]: The integration domain started in late 2018 when we began building connectors in-house. We utilized existing APIs from partners like Microsoft Dynamics to extract data and handle transformations. For example, Apsis One requires boolean values as true/false, but Microsoft Dynamics uses 1/0 for booleans, so we had to adapt and transform the data accordingly.

The fundamental problem with this approach became evident quickly: **adding new features required replication across every connector**. When we added email event syncing capabilities to Apsis One, we had to implement this separately for Microsoft Dynamics, Lime CRM, and FSC Enterprise. By the time you have 4+ integrations, this becomes **unsustainable** because a single new feature could take months of development, and maintenance becomes a nightmare.

[Lukasz Grabowski]: Approximately how many customers use these legacy connectors now?

[Erik Andersson]: We have quite a few for Microsoft Dynamics—roughly 60 customers on my estimate. We have many FSC Enterprise 12.0 customers because that was heavily promoted when we were acquired. Enterprise customers are large corporations hesitant to upgrade or migrate from on-premise versions, which has been a recurring headache. Lime CRM has only 1-2 customers, but they pay well, so we maintain it.

### Current Status of Legacy Connectors

However, we have not developed any new legacy connectors for quite some time, and we don't add new features to them. The only exception would be if a very large customer paid significantly to add a specific feature—**this has not happened yet**.

[Erik Andersson]: Justin is the name of the platform. Everything here is connected to Justin because connectors are installed within Justin. Each connector may differ significantly in what they support. The older ones (Microsoft Dynamics, Lime, FSC Enterprise) are full-blown CRM integrations supporting field mappings, contact retrieval, consent syncing, and data synchronization via webhooks.

---

## Generic Connector Architecture

### Core Philosophy

[Erik Andersson]: The generic connector shifted responsibility from Apsis to the CRM partner. Instead of Apsis adapting to each CRM's data structure with "ugly hacks," the CRM now adapts to a standardized specification. We have **one generic connector** in Apsis, and all support for new features is added there once—not replicated across multiple systems.

### Operational Model

When adding a new feature like SMS event syncing from Apsis to the CRM:
1. We add the support and endpoints to the generic connector
2. The CRM developers implement the expected endpoints on their side
3. All CRM systems call identical endpoints with the same format (only hostname differs)

**Example**: We call standardized endpoints like `/records/page_number` for all CRMs—Tribe, E-deal, Web CRM. We expect identical data formats and behavior from all of them.

### Benefits

- **New features are added once and work for all CRM partners** that have implemented support
- **Debugging is straightforward**: If generic connector works for Tribe but fails for E-deal, the problem clearly resides in E-deal's implementation
- **Testing is simplified**: Easy to isolate which system is not conforming to the contract
- **Reduces data manipulation**: CRMs must send data in the exact format specified—no more string-to-integer conversions, 1/0 to true/false hacks

### Feature Flag Communication

[Michal Rosikiewicz]: When you add new features to the generic connector, do you use versioning, or do CRMs only receive features they've enabled?

[Erik Andersson]: When we add something new that a CRM doesn't support, we don't start sending it until they enable that feature flag. For example, we don't display the Event Tool sync option in Apsis until the CRM has stated they support it. We don't do traditional versioning on the generic connector—we haven't removed or modified the contract in ways requiring version bumps. The primary mechanism is that **CRMs tell us what they support**, and we only send/display features they've enabled.

[Michal Rosikiewicz]: So if Tribe enables event syncing, you start sending events only to Tribe, not to E-deal or others?

[Erik Andersson]: Correct. We start sending when a user creates an event and selects "sync to CRM." That option doesn't appear until they've enabled it in the CRM. When we receive notification that they've selected the sync option, we create listeners on Audience and start sending events to that specific CRM instance.

### Historical Development

[Shreevidhya Ganesan]: We started the generic connector project around summer 2022. Since then, we've been developing only generic connectors.

[Erik Andersson]: The transition accelerated due to FSC Enterprise's complexity. For instance:
- If a date field is empty, FSC Enterprise sets it to December 1899, which is technically a valid date and could trigger MA flows, suddenly making all customers appear 100+ years old
- They claim to support integer fields but send stringified integers
- They claim to support booleans but send stringified 1/0 instead of true/false
- The amount of data manipulation, inconsistency, and bugs was enormous

This motivated us to standardize on **one set of endpoints** that the CRM must conform to, not the other way around.

### Configuration Responsibility

**All responsibility for data consistency resides in the CRM system**. Apsis does not perform workarounds or data manipulation. If a CRM says a field is an integer, it must send an actual integer, not a string. If it's a boolean, it must be a boolean, not 1/0.

---

## Legacy vs. Generic Connector: The FSC Enterprise Case

### Two Versions Exist

**FSC Enterprise 12.0**: The old legacy connector that Apsis built, handling all the data transformation "black magic" on our side.

**FSC Enterprise 12.1**: The new version that implements generic connector support. Enterprise handles all data transformation themselves.

### Migration Challenge

[Lukasz Grabowski]: Shouldn't we hide the old FSC Enterprise 12.0 connector since we have 12.1?

[Erik Andersson]: Unfortunately, no. We still have approximately 30 FSC Enterprise 12.0 customers who are hesitant to migrate. Some may be on old on-premise versions incompatible with 12.1. If a new customer has an existing 12.0 system and wants to start using Apsis, we cannot hide that option.

### Contrast with Web CRM

Web CRM, being a SaaS service with no on-premise instances, could be fully migrated. We were able to hide the old legacy connector because every customer could upgrade to the new generic connector version. We only show the old connector to customers who already have it installed.

[Erik Andersson]: **The dream state** is that every old legacy connector customer is migrated, and I'd recommend this be pushed forward in tandem with consultancy, as they typically handle the actual migration since they sit with the customer's data.

---

## Third-Party Integrations

### Concept

External companies have built **third-party integrations** to Apsis without using Justin extensively. They bypass our platform logic and use different mechanisms. Apsis's role here is minimal.

### Apsis's Limited Role

We provide support only for the installation process:
1. Create a keyspace for the third-party integration
2. Whitelist the CRM ID attribute for that keyspace
3. Generate a single API key

**We do not**:
- Support the actual data syncing if it fails
- Add new features to these integrations
- Know how they work internally

The customer is responsible for entering this API key into the third-party system's configuration interface.

### Examples and Customer Count

- **Super Office**: ~5-6 customers
- **Intermail Loyalty**: 1 customer (being discontinued)
- **Join Xi**: 2-3 customers
- **Lead Family**: A couple of customers
- **WooCommerce**: 3-4 customers

**Total: Well under 10 customers** across all third-party integrations—nowhere near the volume of true CRM integrations.

This is largely a relic from when Jonas was the product owner and attempted to sell integrations to other companies.

### Intermail Loyalty Exception

Intermail uses Apsis for their marketing solutions and has built integration support. Unlike other third-party integrations, **Intermail has implemented full generic connector support**. They handle the connector on their side and maintain it themselves. Customers enter the API key in Intermail's UI, and Intermail makes the calls to our platform.

[Michal Rosikiewicz]: How does Intermail differ from Super Office? Do they get the API key the same way?

[Erik Andersson]: Yes, Intermail gets the API key from our UI. When they enter it, our code starts making connections to set up webhooks and retrieve data using that key. We never call Intermail's system to retrieve data—they use the API completely for that. For other third-party integrations, Apsis only displays the key in the UI; the customer copy-pastes it into the third-party's configuration page outside our control.

---

## Strategic Guidance on Building New Integrations

[Erik Andersson]: **If someone asks you to build an integration, your answer should be: No. We've built connector support in the generic connector. Build the endpoints for the generic connector, and it will work.**

### Why Not to Take Ownership

The cost of building and maintaining proprietary connectors is unsustainable. **A bug fix for an older Microsoft Dynamics plugin can cost 2,500 SEK/hour**. This cost can quickly consume all revenue if you're unlucky.

### Recommended Approach: Partner Model

The ideal model is what we have with **Siteshop**:
- The partner owns the customer relationship (not Apsis)
- The partner charges the customer for the integration
- Apsis benefits because Siteshop loses customers to Apsis, who then pay for profiles and emails
- **Most importantly**: The partner owns maintenance and customer support

Compare this to old Microsoft Dynamics:
- Apsis paid a development partner to build the connector
- Apsis charges customers for use
- Apsis is stuck funding support and maintenance—unsustainable long-term

[Erik Andersson]: Find someone who wants to build a connector to Apsis **and own the customer relationship and support**. In exchange, Apsis gets the value from customers they're syncing. Do not take on the maintenance burden yourself. R&D should focus on adding configuration support within Apsis for the generic connector and assisting with development, but not maintaining the connector code itself.

---

## Installer Options and Configuration

### Overview

Every CRM differs in entity names, field names, and supported features. The **Installer Options** UI allows configuration of these differences without code changes.

### Key Configuration Fields

For each CRM connector (example: FSC Enterprise):

**Display Name and Integration ID**
- Display name: shown to users
- Integration ID (logical ID): critical for internal differentiation

**Feature Toggles (Deadbolts)**
- Example: `can_sync_email_activities`, `can_sync_event_tool_activities`
- If known that a CRM version doesn't support a feature, we disable it completely
- If enabled, we check each customer's specific instance to see if they have support (useful for mixed development/production environments)

**Performance Parameters**
- Concurrency settings for full syncs
- Page size configuration when downloading contacts

**Entity Configuration**
- Profile entity: the main CRM entity representing customers
- Display name of entity (e.g., "Contact" for Dynamics, "Person" for Tribe)

**Sync Direction**
- **Inbound**: Should we download data from CRM to Apsis?
- **Outbound**: Should we send form submissions and consent changes to CRM?

**ID Field Name**
- **Critical**: The unique identifier field that Apsis uses as the key
- Must be unique for every contact
- Called "ID", "contact_ID", "identifier", etc. depending on the CRM
- This field value becomes the key in Apsis keyspace for identifying and updating records
- If not specified correctly, duplicate profiles will be created

[Lukasz Grabowski]: If outbound is disabled, we can't sync consents, right?

[Erik Andersson]: Consent is an exception. Consent must always be synchronized bidirectionally, regardless of outbound settings.

---

## Consent Synchronization: Bidirectional Exception

### Why Consent is Different

Normally, the CRM is the single source of truth for attributes—what's in the CRM should be in Apsis. However, **consent is bidirectional**:

**Scenario**: 
- Customer imports contact database from CRM to Apsis
- Sends an email campaign
- Recipient clicks "unsubscribe" in the email
- Apsis marks the profile as opted out
- **Problem**: If we don't sync this back to the CRM, the next sync from CRM will restore the opted-in consent, and we'll violate the customer's wishes by sending unauthorized emails—**legal and compliance disaster**

### Solution

We sync consent changes from Apsis back to the CRM whenever:
- A customer unsubscribes via email link
- Support team manually updates consent (GDPR requests)
- Any other mechanism changes consent in Apsis

**Never** do we sync attribute changes from Apsis to CRM, as customers should update attributes in their system of record (the CRM).

---

## Outbound Flow: Forms and Event Tool

### What is "Outbound"?

Outbound means sending data from Apsis to the CRM, specifically:
1. **Form submissions**: When a lead fills out a form in Apsis
2. **Event Tool syncing**: When an event is created and marked to sync to CRM
3. **MA profile movements**: When a profile enters certain task nodes (covered in MA session)

### Problem: Form Fields to CRM Lead Mapping

**Scenario**: Customer creates a form with custom fields like "My Cool Field" and "Favorite Color." They want leads from this form sent to the CRM. But how does the CRM know which form field maps to "First Name," "Last Name," or "Email" in the lead structure?

**Without outbound mappings**, the CRM only receives raw form submission data and must guess.

### Solution: Outbound Mappings

Outbound mappings allow reverse mapping:
- Take Apsis attributes
- Map them to CRM lead/entity fields
- When a form is submitted, extract the profile's attribute values
- Create a JSON object with the correct field names the CRM expects

**Example**:
- Form field "Eric's Cool Field" (Apsis attribute) → maps to "first_name" in CRM lead
- Form field "Sailing Boats" (Apsis attribute) → maps to "last_name" in CRM lead
- Form field "Email" → maps to "email" in CRM lead

When the form is submitted, Apsis sends:
```json
{
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com"
}
```

Instead of just raw form submission data, the CRM can now auto-populate the lead.

### Enabling Outbound Mappings

[Lukasz Grabowski]: Is outbound mapping shown on the front end only if the outbound flag is enabled?

[Erik Andersson]: Not exactly. Outbound mapping configurations are shown if supported by the CRM. Whether or not they're **used** depends on the outbound flag:

- **If outbound is disabled**: Form submissions are sent as raw events; the CRM receives no attribute mapping
- **If outbound is enabled**: Form submissions are enhanced with mapped attribute data from the profile

The mapping itself still provides value even if some attributes aren't filled in the form; they're sent if they exist on the profile.

### Missing Attributes in Form

[Michal Rosikiewicz]: If the form doesn't contain a "mobile number" field but you have it in outbound mappings, will it be filled?

[Erik Andersson]: If the attribute doesn't exist on the profile due to not being in the form, we can't send it. But if other attributes exist (like email, first name, last name), those are sent regardless.

### Event Tool Syncing

[Lukasz Grabowski]: The Event Tool doesn't show a "sync to CRM" checkbox in my setup. How is this configured?

[Erik Andersson]: You need the feature flag enabled, and it requires that the CRM has stated it supports event tool syncing (the `can_sync_event_tool_activities` flag). The feature flag might be a leftover from development.

[Lukasz Grabowski]: Actually, we check both the feature flag and the endpoint response. The feature flag may be a leftover we should remove and rely only on the integration endpoint's capability declaration.

[Erik Andersson]: Good idea. The endpoint response already tells us what the CRM supports; the feature flag is redundant for a released feature.

### Technical Note on FSC Enterprise

[Lukasz Grabowski]: Why can't I see outbound mappings in FSC Enterprise 12.0?

[Erik Andersson]: FSC Enterprise 12.0 (legacy) doesn't support outbound mappings. The 12.1 version (generic connector) does, though there's a suspected bug where it's not visible. Outbound mappings were a key reason they wanted generic connector support.

---

## Key Takeaways

1. **Three connector types exist**: Legacy (in-house built), Generic (standardized), and Third-party (externally managed). New development should be generic only.

2. **Legacy connectors are in maintenance mode**. We maintain Microsoft Dynamics, Lime, and FSC Enterprise 12.0 for existing customers but develop no new legacy connectors. Migration to generic versions is ongoing but blocked by customer reluctance to upgrade on-premise systems.

3. **The generic connector is the future**: All new CRM partners should implement the generic connector specification. This approach shifts data transformation responsibility to the CRM, making features scalable and maintenance tractable.

4. **Installer Options** configure entity names, field mappings, sync directions, and feature support per CRM without code changes.

5. **Consent is bidirectional**: Unlike other attributes, consent changes in Apsis must always sync back to the CRM to prevent compliance violations.

6. **Outbound mappings** enable smart lead creation from forms by mapping Apsis attributes to CRM field names. They're optional but valuable when configured.

7. **Do not build proprietary connectors**: Unsustainable cost and maintenance burden. Instead, find partners willing to build and own the integration using the generic connector and support their own customers.

8. **Third-party integrations** (Super Office, WooCommerce, etc.) are minimal support—we provide keyspace and API key generation only. Intermail Loyalty is a rare exception with full generic connector support.

---

## Unresolved Questions and Action Items

- **FSC Enterprise 12.1 outbound mappings visibility bug**: Confirm if there's an actual bug preventing outbound mappings from displaying in version 12.1
- **Event Tool feature flag cleanup**: Create a story to remove the redundant feature flag and rely solely on the integration endpoint's capability declaration
- **Customer migration timeline**: Recommend coordination with consultancy to push FSC Enterprise 12.0 customers to 12.1
- **Code repository structure**: Next session will cover repository organization and where to find key components