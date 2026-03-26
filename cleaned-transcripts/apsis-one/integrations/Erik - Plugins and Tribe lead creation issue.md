---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One Integrations
topics: [Plugin ownership models, Legacy vs third-party connectors, Generic connector protocol, Tribe lead entity bug, Form submission to CRM lead creation, Webhook registration issues, Integration cleanup and deprecation]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Apsis One, Generic Connector, Tribe CRM, E-deal, Zapier, Lime CRM, Microsoft Dynamics, Third-party integrations, Webhook system, Form submissions, Lead entities, Silhouette IDs]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors", "Lead creation"]
---

## Session Overview

This session covered the historical evolution of Apsis One's integration strategy, focusing on the shift from owned plugins to third-party and generic connector models. Erik walked through the current state of multiple plugin repositories (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely, Zapier, and Lime), explaining why most are now deprecated and open-sourced. The session culminated with a deep technical discussion of a critical bug in the Tribe CRM integration where virtual lead entities were causing duplicate profile creation, including the fix implemented and implications for future installations.

---

## Historical Context: Plugin Ownership Models

### The Original Strategy (Pre-Felix Era)

When integrations were first implemented, Apsis One employed a **plugin ownership model**. For CRM systems requiring installation (Microsoft Dynamics, Presta Shop, EPI Server, Optimizely), Apsis paid external companies to develop plugins.

> "APSIS owns the code that was developed, but that also means that APSIS owns the support for this. So if any the customer had any issue with the plugin then the customer would reach out to APSIS."

This created a problematic support loop:
- Customers reported issues to Apsis support
- Apsis had to investigate and either identify bugs (free fixes) or request features (expensive billable work with external developers)
- Support overhead was significant because the external development teams could identify issues more quickly from their own logs

### The Business Model Shift (Felix Era)

When Felix became head of Apsis, the strategy fundamentally changed. Instead of Apsis owning plugins and support:

**New model**: Apsis partners with third-party developers as **business partners** rather than vendors:
- The third-party company builds and maintains the plugin
- The customer pays the third-party company directly (not Apsis) for plugin usage
- Apsis revenue comes from volume (shipping "a crap load of profiles per month" through the integration)
- Support is owned by the third-party developer; Apsis only handles cases where data is sent correctly but errors occur on the Apsis side

**Advantages of the new model**:
- When Apsis adds new features (e.g., event tool syncing), the third-party developers are incentivized to implement support themselves—Apsis doesn't have to
- Reduced support burden on Apsis team
- Third-party developers have more vested interest in expanding connector functionality since it improves their product offering

Example: [Erik Andersson]: "Venst Patillet is not paying APSIS for the usage of the plug-in, they are paying Site Shop because Site Shop built the plug-in."

---

## Current Plugin Repository Status

### Deprecated Plugins (Open Source, No Support)

The following plugins are no longer supported by Apsis but remain available as open-source code in repositories:

| Plugin | Status | Notes |
|--------|--------|-------|
| Presta Shop | Open source, deprecated | E-commerce platform; no active customers |
| Adobe Commerce (formerly Magento) | Open source, deprecated | E-commerce platform; no active customers |
| Sitecore | Open source, deprecated | CMS system; no active customers |
| Drupal | Open source, deprecated | One old customer using customized version; no platform support |
| Optimizely | Open source, deprecated | No active customers |

**Key policy**: Even for customers who previously paid for these plugins, there is **no support**. Customers can use the open-source code if they choose to maintain it themselves, but Apsis will not:
- Provide bug fixes
- Add features
- Investigate compatibility issues
- Help with installation

[Michal Rosikiewicz]: "We refuse to install any of those plugins for customers. They can of course try to do it themselves."

[Erik Andersson]: "Even if they paid for it, yes." (in response to whether support applies to legacy customers)

**Rationale**: These integrations were unsuccessful—either had no customers or customers churned. The decision was made to allocate resources elsewhere.

### Why Not Delete Them Entirely?

[Erik Andersson]: "it was considered like waste of money to completely delete them. Now that's I mean we we have them here but these are not any plugins you should bother."

The plugins are kept as:
1. Reference implementations (if a customer requests a new integration, the old plugin code shows how plugins were built)
2. Open-source resource (customers with urgent needs can fork and maintain)
3. Historical artifacts

---

## Active Third-Party Integrations

### Zapier Integration

**Repository**: `apsis-integrations-zapier`

- Apsis owns and maintains this application
- Built in Zapier's platform
- Source code stored in the repository; packaged and uploaded to Zapier from developer machines
- Extensive customer usage
- Uses **one API 100%**: All Zapier-to-Apsis communication goes through the One API
- Maintenance is minimal (passive, stable)

[Erik Andersson]: "The Zapier integration utilises one API 100%. So in anything which happens in Zapier goes through one API like we we own the application but we are fully utilising one API for it."

### Lime CRM Plugin (Reference Only, Not Supported)

**Repository**: `apsis-integrations-lime` (kept for historical reference)

This plugin was an experimental attempt to replicate the email and website event retrieval feature that existed in the old Dynamics integration.

**Historical context**:
- Lime CRM doesn't natively support Apsis event retrieval
- Erik attempted to build a custom Lime plugin with custom UI elements and API endpoints
- Implementation ran into significant complications:
  - Database persistence and data isolation (difficult to keep customer data separate)
  - Configuration complexity
  - Customer interest waned quickly

**Why it's kept**: As a reference for how plugins are structured for Lime, should a future generic connector implementation be needed.

[Erik Andersson]: "Even if we are not actually utilising this one, we wanted to preserve it. If for example in the future someone decides that yeah, but we like there should be a generic connector implementation for Lime."

**Development challenge Erik faced**:
- Access to Lime test environments was straightforward (Apsis has historical relationships with Lime; several ex-Apsis employees work there)
- Technical knowledge existed (Erik knew Python, which was required)
- Problem: "quite complicated configuration like with database access and persistence like how do you keep the data in the instance for only one customer?"
- If misconfigured, customer data could leak to all accounts on a shared instance

**Cautionary lesson**: [Erik Andersson]: "Nowadays I would not suggest anyone in your team sit and build custom plugins because then again you will have to own the maintenance or support and bug fixing for that."

---

## Integrations to be Removed/Cleaned Up

During the session, Erik identified deprecated integrations still appearing in the UI that should be removed:

**Completely deleted from codebase** (but may still appear in UI):
- Shopify (license cost: Apsis would owe Shopify 12% of annual revenue)
- E-commerce plugin (collaboration scrapped for budget reasons)
- Facebook Lead Ads (ownership unclear; mishmashed between teams; support team later disbanded)
- Tessitura
- Course
- Google Data Studio
- Ambraco
- Power BI

**Status**: Pre-launch or showing as available when they shouldn't be visible to customers.

[Erik Andersson]: "I'll make sure to do some clean up here... I'll try and clean up as much as I can here to remove all integrations that are no longer relevant."

**Work needed**:
- Remove from integration list (backend responsibility)
- Remove UI components/cards (frontend responsibility; potentially involves data model changes beyond Erik's Angular expertise)
- Create story for Chris (frontend developer)

---

## The Tribe CRM Lead Entity Bug

### Background: Form Submission to Lead Creation Flow

When a customer sets up a form submission to sync leads to a CRM, the flow is:

1. Customer fills out form in Apsis One
2. Apsis One submits form data to CRM via integration
3. CRM responds with a **lead entity** (or contact/person entity depending on the CRM)
4. CRM response includes a unique identifier (silhouette ID or similar)
5. Apsis creates a **profile in Apsis One** with that identifier in a dedicated keyspace

Example from E-deal (working correctly):
```
Form submission → E-deal API
E-deal response: new_records: [{ silhouette_id: "abc123", ... }]
Apsis creates profile with silhouette_id "abc123" in "silhouette" keyspace
```

### Generic Connector Configuration for Leads

When setting up an integration, the generic connector allows configuration of mappings for different entities. For lead creation flows:

- **Entities configured**: Contact, Person, Silhouette, Lead, etc. (depending on the CRM)
- **Field mappings**: Which Apsis fields map to which CRM fields for this entity
- **Consent mappings**: Which consents should sync for this entity
- **Webhook registration**: During installation, Apsis registers webhooks for each configured entity so the CRM can send updates

### The Tribe Complexity: Virtual Lead Entity

Tribe CRM has a **critical difference**: the lead entity is **not a real entity**; it's a **virtual placeholder** for a customer-configured selection.

During Tribe installation, the customer must select which entity should be created when a form is submitted:

```
Tribe configuration dropdown:
- Contact
- Potential Customer
- Opportunity
- Lead
- [Custom Entity 1]
- [Custom Entity 2]
```

**The problem**:

1. **Virtual representation**: When Apsis asks Tribe "what attributes does the lead entity have?" we're actually asking "what attributes does the selected entity have?"
2. **Dynamic response**: If the customer selects "Potential Customer," Tribe will respond with Potential Customer attributes, not Lead attributes
3. **Webhook registration mismatch**: During installation, Apsis registered webhooks for a "Lead" entity, but Tribe never creates anything explicitly called "Lead"
4. **Data sent to wrong endpoint**: Tribe, misinterpreting the webhook registration, sent contact updates to **both**:
   - The contact webhook endpoint (correct)
   - The lead webhook endpoint (incorrect—but Apsis had registered it)

### The Bug Manifestation

When Tribe sent contact data to the lead webhook endpoint:

```
Contact update received at /lead endpoint:
{
  id: "contact_123",
  name: "John Doe",
  email: "john@example.com"
}

Apsis interpreted this as:
"A new lead with ID 'contact_123' was created"

But lead entity has completely different keyspace and mappings than contact.
Result: Create new empty profile in "lead" keyspace with ID "contact_123"
```

**Symptom**: Customers saw **duplicate profiles** in their Apsis instance—one with full contact data, one empty profile with only the lead ID.

[Erik Andersson]: "Because the ID field here from the contact is the same as this ID field here, we took the ID for the contact but created a new empty profile in the lead key space. And as you said, we have no real mappings for this, so it was a empty profile file that happened to have a lead ID that lived in parallel with the real contacts here."

### Root Cause Analysis

The virtual lead entity creates unpredictability on both sides:

- **Apsis side**: We configure mappings for a "lead" entity but don't know which real Tribe entity will be selected
- **Tribe side**: Apsis registers a "lead" webhook, but Tribe never actually creates "lead" entities; they create the customer-selected entity

This breaks the assumption that configured entities match actual CRM entities.

### The Fix

Erik implemented a solution that prevents webhook registration for the Tribe lead entity during installation:

```python
# During webhook registration loop:
if entity in NO_WEBHOOK_ENTITIES:  # Tribe "lead" is in this list
    continue  # Skip webhook registration
else:
    register_webhook(entity)
```

**Scope**: 
- Affects new Tribe installations going forward
- Existing customer affected by the bug had profiles manually deleted (Erik had API access via account manager)

[Erik Andersson]: "What I've done now is that I've added like an additional parameter saying like do not create web hooks. So during the installation process here we we retrieve all of the list or all of the entities for the connector. But before I actually add them to the list of reg webhooks to be registered, I check if we should not create web hooks for this."

### Ideal Future Solution

[Erik Andersson]: "What I would really want to do for tribe is I would want to do like this. I I wanted them to always give us like some static entity so we have some kind of predictability..."

**Proposed change**: Tribe should **always use the Contact entity** for form submissions, not allow it to be configurable. This would:
- Eliminate the virtual entity concept
- Make field mappings and consent mappings deterministic
- Simplify the integration and remove the fragility

Current workaround: In practice, most Tribe customers select Contact anyway, so the configuration happens to work by luck.

[Erik Andersson]: "The good thing is, as far as I have been told, everyone selects contacts as the as the entity to be created and then it just so happens that the configuration we have for contacts will be exactly the same that we have for leads, but that's just pure sheer luck."

---

## Webhook and Lead Management

### Webhook Registration During Installation

The generic connector registers webhooks for all configured entities during installation. This allows the CRM to:
- Send **creates** (new records)
- Send **updates** (attribute or consent changes)
- Send **deletes** (record no longer relevant)

For lead entities specifically:
- **Inbound**: Apsis expects CRM to send creates/deletes when leads are created/discarded
- **Outbound**: Apsis never expects CRM to send lead updates (leads are typically unidirectional: Apsis → CRM)

### Why Webhooks for Deletes Matter

Even though leads are typically one-way, CRMs may want to signal back: "We received this lead, it's not relevant, please delete it from Apsis."

[Erik Andersson]: "if they choose to not proceed with the lead, then they can send a delete request saying like, yeah, we received this lead, it's not of any interest, please delete it."

The delete uses the same webhook as updates—a delete operation is just a type of operation.

### Profile Creation via Form Submission

**Scenario**: Customer uses an Apsis form to collect leads for Tribe CRM.

1. Form submitted
2. Apsis sends to Tribe: `POST /api/contacts { name: "...", email: "..." }`
3. Tribe responds: `{ silhouette_id: "lead_456", ... }`
4. Apsis creates new profile in Apsis with silhouette ID `lead_456` (in the dedicated lead keyspace)

**Key distinction**: We only create profiles from **inbound form submissions**. We do **not** pull/download existing leads from the CRM.

[Erik Andersson]: "We will never download leads from the CRM system because that there has not been a use case yet where that has been interested. The CRMS are interested in getting this data from APSIS to the CRM so they can process their leads and decide..."

---

## Support Routing and Knowledge Sharing

A critical lesson emerged regarding customer issue escalation:

**Problem**: If account managers or customers directly ping integration engineers about issues (e.g., empty profiles in Tribe), the SOC/support team may not be informed, leading to:
- Loss of knowledge about recurring bugs
- Repeated questions about the same issue
- No systematic tracking of how many customers are affected

**Solution**: All customer issues should go through SOC/support team first. They:
1. Can provide first-level diagnostics
2. Build institutional knowledge of known issues
3. Escalate to engineering with full context
4. Track patterns across multiple customers

[Erik Andersson]: "This is why it is so important that these questions go via SOC and support, because now they know about this problem. We've described it to them. If there are more customers that experience the same thing, they already know what the problem is."

---

## Debugging and Investigation Tools

### Postman Collection for Generic Connector

Apsis maintains a Postman collection for testing and debugging generic connector endpoints:

**Contains endpoints for**:
- Getting schema (list of mappable fields for an entity)
- Creating webhooks
- Listing webhooks
- Deleting webhooks
- Testing mappings

**Usage**:
1. Configure with customer's environment URL and API key
2. Test field schema retrieval (to check if Apsis can see available fields)
3. Test webhook creation/deletion (useful during installation troubleshooting)
4. Debug permission errors, 404s, empty responses, etc.

**Example use case**: Customer reports "I can't see any field mappings." Engineer uses collection to:
```
1. GET /schema/contacts → If empty or 404, likely permission issue
2. Verify API key has correct scopes
3. Check if customer's environment is properly configured
```

[Erik Andersson]: "This postman collection is very very useful... If they say why can't I see any mapping? Yeah, well when I try to make this request here to get the schema like I get an empty list or I get a permission denied or I get a 404 or whatever."

---

## Comparison: How Other CRMs Handle Lead Creation

### E-deal (Generic Connector)
- **Lead entity**: Real, concrete entity
- **Form submission response**: Includes `silhouette_id` field with lead ID
- **Webhook registration**: Predictable; we know lead entity will actually be used
- **Status**: Works as expected

### FSC Enterprise (Generic Connector)
- **Lead entity**: Does not exist; only "Contact" entity
- **Form submission response**: Returns contact record
- **Webhook registration**: Straightforward; only Contact webhooks
- **Status**: Works as expected

### Dynamics (Third-party plugin, not generic connector)
- **Lead entity**: Real, concrete entity
- **Status**: Support prepared for form submissions but not yet enabled
- **Note**: Webhook registration for leads would be reliable

### Tribe (Generic Connector)
- **Lead entity**: Virtual placeholder for customer-selected entity
- **Problem**: Entity name doesn't match actual entity; webhooks registered for non-existent entity
- **Status**: Fixed by not registering lead webhooks; customers should only use Contact entity

---

## Key Takeaways

1. **Plugin ownership model evolution**: Apsis shifted from owning and supporting plugins to partnering with third-party companies. This reduced support burden and aligned incentives for feature expansion.

2. **Deprecated plugins are dead**: Presta Shop, Adobe Commerce, Sitecore, Drupal, and Optimizely plugins are open-source only. Provide **no support** even to legacy customers. Customers must maintain their own forks.

3. **Why plugins are kept**: Historical reference for future implementations and open-source availability for customers with urgent needs.

4. **Virtual entities are dangerous**: Tribe's dynamic lead entity selection created duplicate profiles because Apsis registered webhooks for a non-existent entity. The fix: skip webhook registration for virtual entities.

5. **Lead creation is unidirectional**: Apsis only creates profiles in Apsis when CRM responds to form submissions. We never pull/download leads from the CRM.

6. **Webhook registration must match reality**: Registered webhooks should only be for entities the CRM will actually send data for. Mismatches cause spurious profile creation.

7. **Support routing is critical**: All customer issues should flow through SOC/support to build institutional knowledge. Direct escalations bypass knowledge sharing.

8. **Generic connector is broadly applicable**: Most modern CRMs can be integrated via the generic connector pattern (static entities with field mappings). Custom plugins should be avoided unless absolutely necessary.

---

## Unresolved Questions & Action Items

1. **Integration cleanup story**: Erik to create a story documenting all integrations to be removed from the UI (Shopify, E-commerce, Facebook Lead Ads, Tessitura, Course, Google Data Studio, Ambraco, Power BI, etc.)

2. **Frontend work for cleanup**: Requires collaboration with Chris (frontend developer) to remove UI cards and associated data models.

3. **Tribe entity standardization**: Ideal long-term fix is to standardize Tribe to always use Contact entity for form submissions, eliminating the virtual entity concept. Rose (product owner for Tribe) is a potential ally for this change.

4. **Monitor for other Tribe customers**: SOC/support should watch for reports of empty profiles linked to Tribe installations. If found, the same bug likely affected them.

5. **PR for Tribe webhook fix**: Erik intends to submit a PR implementing the no-webhook-registration fix for the lead entity.