---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One Integrations
topics: [Plugin Architecture and Ownership Model, Legacy Plugin Repositories, Third-Party Integrations, Generic Connector Implementation, Tribe Integration Complications, Lead Entity Virtual Representation, Webhook Registration Issues]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Apsis Integrations repositories, Plugin architecture, Generic Connector, Zapier integration, Lime CRM plugin, E-deal integration, Tribe integration, Webhook management, Lead entity configuration, Key space management]
session_type: knowledge-transfer
subdomains: ['Architecture', 'Tribe Integration', 'Lead creation']
---

## Session Overview

This knowledge transfer session covers the architecture and evolution of plugin repositories within Apsis One Integrations, tracing the historical shift from Apsis-owned plugin development to a partner-based model. The discussion addresses legacy plugin repositories (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely), the current third-party integration landscape, and deep technical issues encountered with the Tribe integration's virtual lead entity concept. The session emphasizes why building custom plugins is costly and should be avoided, and documents a critical bug in Tribe lead creation that resulted in duplicate empty profiles being created.

---

## Historical Context: Plugin Ownership Model Evolution

### The Original Strategy (Pre-Felix Era)

When integration strategy was first implemented, **Apsis paid external companies to develop plugins** for CRM systems where installation was required (Microsoft Dynamics, Presta Shop, Episerver, Optimizely). The business model had Apsis owning the developed code and bearing all support responsibility.

[Erik Andersson]: "If any customer had any issue with the plugin then the customer would reach out to APSIS. APSIS would need to investigate like where it goes wrong. And if we don't see that, yeah, there's something wrong in the CRM, then we need to set up a meeting with the company that develops the plugin and have them correct this."

The support burden was significant—if issues arose, Apsis had to:
- Investigate the problem
- Arrange meetings with the plugin development company
- Pay external companies per hour for corrections or feature additions (not free if the issue was on Apsis's side)

### Strategic Shift Under New Leadership

When Felix became head of Apsis, the strategy fundamentally changed. Apsis no longer wanted to own plugin support because:
- It consumed significant internal resources
- Plugin developers could often diagnose issues faster from their own logs
- It prevented scaling support across multiple plugins

**New model**: Instead of Apsis owning the plugin, **partners become business owners** of the integration. Customers pay the plugin developer directly, not Apsis. Apsis benefits from receiving large volumes of profile data monthly.

[Erik Andersson]: "The customer is paying whoever develops the plug-in the money, so that company owns the customers. For example, the Dynamics by Sideshop. Venst Patillet is not paying APSIS for the usage of the plug-in, they are paying Site Shop because Site Shop built the plug-in."

**Advantages of the new model**:
- Apsis only provides support if the partner confirms they're sending correct data and Apsis returns errors or format issues
- When Apsis adds features (e.g., event tool syncing), partners handle enabling support for their own connectors
- Partners have incentive to maintain and improve connectors (expands their customer offering)

---

## Legacy Plugin Repositories: Deprecated and Unsupported

### Current Status of Owned Plugins

The following repositories exist but are considered **dead connectors with no support**:

- **Presta Shop** (e-commerce)
- **Adobe Commerce** (formerly Magento)
- **Sitecore** (CMS)
- **Drupal** (CMS)
- **Optimizely** (CMS)

These were migrated to the new repository structure but are not actively maintained or supported.

### Open Source Availability

[Erik Andersson]: "These are considered open source. So let's say that any customer like really would want a integration with Drupal to Appsis. The customer can download the plugin and like they have the source code, they can add any functionality they would want if they have the knowledge or they can correct bugs."

**Critical distinction**: No Apsis support exists for these integrations, even for existing paying customers. The code is available for customers to modify if they have technical capability, but Apsis provides zero maintenance.

[Michal Rosikiewicz]: "Even if they paid for this, yeah."

[Erik Andersson]: "Even if they paid for it, yes."

### API Usage Pattern

Most legacy plugins use **one consistent API** for functionality. Apsis creates the key space for them as it does for third-party integrations, but no custom features or support are provided.

### Current Customer Status

- Presta Shop: No customers
- Adobe Commerce: No customers
- Sitecore: No customers
- Drupal: One old customer using a customized version (minimal to no support provided)
- Optimizely: No customers

[Erik Andersson]: "Like the ones that are in here, like, I mean essentially consider them like dead, dead connectors. For all I care, we don't have any customer for Presta shop, the Adobe Commerce one. I don't think we have anything for site core."

---

## Zapier Integration: Maintained and Customer-Active

### Architecture

The **Apsis Integrations Zapier** repository contains the source code for the Zapier app built and maintained by Apsis. The development workflow:
1. Build app locally on developer's machine
2. Package and upload to Zapier from local environment

### Current Status

- Actively maintained (though minimal ongoing work expected)
- Several customers actively using it
- **100% relies on the One API** for all functionality

[Erik Andersson]: "The Sapier integration utilises one API 100%. So in anything which happens in Sapier goes through one API like we we we own the application but we are fully utilising One API for it."

---

## Lime CRM Plugin: Archived Reference Implementation

### Purpose and Status

The **Apsis Integrations Lime** repository is an archived plugin from ~4 years ago, kept for reference and documentation purposes only. It is not functional, not maintained, and should never be deployed to customers.

### Historical Context: Why It Was Built

Early attempt to replicate functionality that existed in Apsis Pro but needed to be implemented in Apsis One. The specific feature was **retrieving email events and website events for a specific contact** from Apsis One.

In the Dynamics CRM integration, contacts had an "Apsis" tab displaying:
- All email events for that contact
- All website events for that contact

These were retrieved from Apsis One using the One API.

### Technical Approach and Failure

[Erik Andersson]: "We tried to replicate this this feature inside of Lime Serum because it doesn't exist. Like of course, Lime doesn't natively know how to retrieve apps is 1 events, so. This plugin that we have here, it was an attempt by us to. It was an attempt to replicate this by like adding a custom UI element inside of Lime and also adding custom endpoints inside of Lime."

The approach involved:
- Adding custom UI elements within Lime
- Building custom endpoints to retrieve data from Apsis One

However, **the project encountered significant complications and was abandoned**:

[Erik Andersson]: "There were so many complications with this like we never could really get it to work and. The interest for this functionality dwindled quite fast, so we we completely scrapped this project, but we did keep the. We did keep the plugin here to see like how do you actually build a plugin for Lime."

### Database Persistence Challenges

[Erik Andersson]: "There were unfortunately some quite complicated configuration like with database access and persistence like how do you do? How do you keep the data in the instance for only one customer? Like it was apparently very easy to make a mistake and if you have one instance that you share with multiple accounts or customers, if you didn't configure it in the right way or use the correct library, you could accidentally like persist that data for everyone on the whole environment and not for like one particular customer and we could never get that to work in an efficient way."

This was a critical blocker—the risk of data leaking across customer instances made the approach untenable.

### Current Value as Reference Material

The repository is preserved to document:
- How to build a Lime CRM plugin (framework structure, component naming, generation patterns)
- Process for plugin development (still applies today despite ancient implementation)
- Example of plugin architecture for potential future reference

[Erik Andersson]: "The plugin and the implementation is absolutely ancient by now. I don't think you'll be able to run it, but the process for it still holds true."

### Access and Development History

[Erik Andersson]: "There were a lot of people that used to work at APSIS that moved on to Lime, so Benjamin had quite good contacts at Lime and also it's been surprisingly easy for us throughout the history to get like. Access to like test environments that we can play around with."

Developer had:
- Full administrative access to Lime test environments
- Python development skills (rare in the team at that time)
- Support from Lime due to existing relationships with former Apsis employees

### Current Stance on Plugin Development

[Erik Andersson]: "Nowadays I would not suggest anyone in your team sit and build custom plugins because then again you will have to own the maintenance or support and bug fixing for that and you you don't want to do that."

However, if Lime becomes a strategic priority in the future and a **generic connector implementation** becomes necessary, plugins are technically possible because Lime supports custom endpoints.

---

## Why Apsis Should Not Build Custom Plugins: The Cost Reality

### Outsourcing is Always Preferred

When professional services brings a customer request for a custom plugin integration:

1. **Preferred approach**: Find an external company to build the plugin. They own the customer, support burden, and maintenance.
2. **Only fallback**: Build internally if no external partner can be found.

This decision must be made by **product/management, not the integration team**.

### Hidden Costs of Internal Development

[Erik Andersson]: "If you say that like sure you build it against your desire, then they have to know that. Now there will be hours just magically disappearing because now because you are building this, you need to support it. You need to do the debugging. You will need to do even more research on how to do that in this system."

The true cost includes:
- Initial development time
- Ongoing debugging and troubleshooting
- Feature requests and enhancements
- Customer support and maintenance
- Research time for unfamiliar systems
- Unexpected complications with foreign systems (like Lime's database persistence issues)

### Financial Reality Check

[Erik Andersson]: "This has been a recurring question like oh, this super big customer wants this. Can't you build this like no, like I I don't care if they pay me like 500,000 like we will lose money on this from R&D perspective."

Even if a customer pays significantly for a custom integration, Apsis typically **loses money on the R&D investment** over time due to ongoing maintenance and support obligations.

### Recommendation

[Erik Andersson]: "You should really try to never. You should never volunteer to do it that much. I can't say it under gunpoint."

If forced to build, escalate to decision-makers with full understanding of:
- Time sink implications
- Support burden ramifications
- Likely financial loss
- Opportunity cost (time not spent on core product)

### External Option: Silver Lorde Consulting

Apsis has an existing relationship with **Silver Lorde Consulting** that can be referenced when customers request custom integrations.

---

## Removed and Discontinued Integrations: Lessons Learned

### Shopify: Licensing Killed a Complete Implementation

[Erik Andersson]: "We did build a complete Shopify connector. We were even asked to work overtime because it got super, super rushed. And then when we had finished it, someone discovered that OK. If we utilize Shopify in this way, then we have to pay Shopify 12% of the annual apps's revenue. So we have to scrap the whole connector."

**Outcome**: Complete integration deleted after completion due to unfavorable licensing terms discovered post-development.

### E-Commerce: Budget Cuts

[Erik Andersson]: "This was something that we started building and then for budget reasons the collaboration was scrapped."

Partially-built integration discontinued. The page still shows "Coming Soon" but no work continues.

### Facebook Lead Ads: Cross-Team Ownership Failure

[Erik Andersson]: "Facebook lead ads. That was a hot potato that. People tried to push it on integration, but we didn't build any custom integrations. So then they tried to push it on the what team and they didn't want to do it and then it became some mishmash that we handled the authentication for it. And then what? The what team built the logic. But then of course the whole what team was let go. So there is no Facebook lead ads connector either."

**Outcome**: Disjointed ownership model resulted in abandoned integration when the "What" team was dissolved.

**Current workaround**: Customers can use **Zapier with real-time lead triggers** to funnel Facebook leads into Apsis One.

### Completely Removed Integrations

The following have been entirely deleted from the codebase:
- **Tessitura**
- **AS course** (discontinued)
- **Google Data Studio**
- **Ambraco**

**Impact on UI**: These integrations are still visible as "pre-launch" options in the customer-facing integration list, which is **false advertising** since they are not functional or supported.

---

## Frontend Cleanup and UI Management

### Current Problem

The integration platform UI shows many non-functional or obsolete integrations because the **integration list API** still includes them in the backend response. Frontend cards display these with either a "Connect" button or "Read More" link depending on availability.

[Erik Andersson]: "Our list here regulates like is the connect but like can you click connect or not? Is it available for installation? Otherwise it will just say like read, read more here."

### Planned Cleanup Scope

Erik is creating a story to remove:
- All legacy plugin repositories (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely)
- Deleted integrations (Tessitura, Google Data Studio, Ambraco, AS course)
- Other non-functional entries

### Required Work

- **Backend**: Remove integrations from the list API response
- **Frontend**: Remove the UI card component and any related data models (requires Angular/frontend expertise)

[Michal Rosikiewicz]: "It it will be really quick for Chris. So I think creating a story for them it's it's totally OK and just connect it with your story on our board and we will take care about it."

---

## Third-Party Integrations: One API-Based Implementations

### Third-Party Pattern

Third-party integrations that don't require custom plugins and use the One API:

- **Super Office**
- **Playable**
- **Sleeknote**
- **WooCommerce**

For these integrations, Apsis's role is **minimal**:

1. Generate a One API key for the customer
2. Create a Sleeknote key space
3. Customer configures the integration directly in the third-party system (specifying which accounts, sections, key spaces, and API credentials to use)
4. All subsequent data flow happens via One API

[Erik Andersson]: "The only thing we do is when you are here on the when you are here and you press on Connects. The only thing we do is that we have generated A1 API key for you. And we have in the background, we also created a sleek note key space for you that you can utilize in your sleek note integration because everything here is handled inside of the sleek note system."

### I Am Loyalty: Special Case with Consent Mapping

**I Am Loyalty** uses One API for all Apsis integration but has one exception: **consent export**.

One API does not support exporting consent changes from Apsis back to the third party, so the integration uses **consent mapping** instead.

[Erik Andersson]: "The reason why I am loyalty is a bit special is they are utilizing. They are utilizing one API for everything 2 apps is. The one exception is that they are interested in getting consents exported from. Um or consent changes from APSIS to back to I'm loyalty and one API does not support that."

When a consent mapping is configured:
- Apsis listens to consent changes in Apsis One
- Changes are sent back to I Am Loyalty via the consent mapping webhook

### Join CX / Join Loyalty: Generic Connector Implementation

These are **actually third-party products** (owned and maintained by company Intermail) but are treated as integrations because they have **full generic connector configuration**:
- Field mappings
- Full syncs
- Consent mappings

[Erik Andersson]: "The whole, the only reason it is marked as third party is abscess like we we don't decide anything about their product like they are utilizing it the way they they want."

**I Am Loyalty** is being **migrated/decommissioned** in favor of Join CX:

[Erik Andersson]: "I am loyalty is the predecessor to this one like they are trying to migrate all of their I am loyalty customer to enroll on this CX platform instead and then we will then they will decommission. I am loyalty and that's good for us. Because that's one less connector we need to keep track of."

### Intermail Partnership

Same company (Intermail) manages both platforms. They are:
- Heavily integrated with Apsis
- Handle loyalty programs, points, and offers
- While Apsis handles marketing

Good collaborative relationship exists.

---

## Native Integrations: Enterprise CRM Platforms

### Current Native Integrations

Integrations owned and maintained by Apsis/FSC:

- **Maxo** (discontinued—being removed)
- **FSC 12.0** (old/legacy implementation)
- **FSC 12.1** (new implementation—primary focus going forward)
- **E-deal** (WebCRM, generic connector-based, future uncertain)

### Discovery Concept

For native integrations, customers don't install anything directly. Instead, they are taken to a page where the integration is explained and the relationship is established through Apsis configuration.

---

## Deep Dive: Tribe Integration and Lead Entity Complexity

### Overview of Tribe as an Integration Platform

Tribe supports the **generic connector protocol**, allowing profile syncing between Apsis and Tribe CRM systems. However, Tribe's architecture introduces a **critical complexity**: virtual/dynamic entity selection.

### The Tribe Configuration Challenge: Dynamic Entity Selection

Unlike other CRMs with fixed entity structures, Tribe requires customers to select **which entity should be created in Tribe when a form is submitted to Apsis**.

This selection is made in a **dropdown in Tribe's configuration**, where the customer chooses which entity type (contact, potential customer, lead, etc.) should be instantiated.

[Erik Andersson]: "One mandatory configuration that you have to do in tribe before you start using APSIS is that. That's. All of the entities in tribe and all of your custom entities in tribe that you can utilize. You select. From this list, which entity should be created inside of tribe when when there is a form submitted?"

### The Virtual Lead Entity Problem

The generic connector configuration for Tribe includes a **"lead" entity** representing potential customers before conversion. However:

**The Problem**: Tribe does not actually return a "lead" entity in responses. Instead, Tribe returns **whatever entity the customer selected in their configuration dropdown**. The entity name is completely dynamic and unpredictable to Apsis.

[Erik Andersson]: "The problem for us is that. Tribe is never responding here with this lead entity. They are responding with the entity that is pro actually selected. But we don't support completely dynamic entities yet in the generic connector."

### Why This Creates a Virtual Entity

Since the generic connector requires **known entity types with defined field mappings**, Apsis created a **virtual "lead" entity** that represents:
- A placeholder for whatever entity Tribe will actually return
- When Apsis requests mappable fields for the "lead" entity from Tribe, Tribe returns attributes for whatever entity is currently selected in the dropdown

[Erik Andersson]: "So what this lead in Apsis actually is, this is a virtual representation of. Of what they have selected in this dropdown."

The workflow is:
1. Customer selects entity in Tribe dropdown (e.g., "potential_customer")
2. Apsis asks Tribe: "What are the attributes for lead?" 
3. Tribe responds with attributes for the selected entity (e.g., potential_customer's attributes)
4. Apsis displays these in field mappings UI under the "lead" entity label
5. Customer configures outbound mappings for "lead"
6. When form is submitted, Tribe responds with the selected entity (e.g., "potential_customer"), not "lead"

### The Webhook Registration Bug

During installation, Apsis registers webhooks for all entities found in the generic connector configuration, including the virtual "lead" entity:

```
webhook_registered: {
  entity: "lead",
  operations: ["create", "update", "delete"]
}
```

**The bug**: Tribe does not have a "lead" webhook because they never create anything called "lead". However, due to this registration request, Tribe interpreted the webhook registration as a request for lead updates and then:

1. Began routing **contact entity updates to the lead webhook endpoint**
2. Tribe sent the same contact data to both the contact endpoint and the lead endpoint
3. Apsis received contact data at the lead endpoint
4. Because the lead entity has completely different mappings and a different key space than contact, **Apsis created new empty profiles** in the lead key space

[Erik Andersson]: "Somehow interpret this at they are sending the same data like they send us the data for the contact to the contact webhook endpoint. But because we had registered the lead endpoint, they sent the same contact data to the lead. Endpoints, but because the lead endpoint for us is a completely different, completely different entity that has completely different mappings, we interpret this as. Oh yeah, someone has created a new lead inside of Tribe. Cool. Let's create like a new profile with it."

### Impact: Duplicate Empty Profiles

When Tribe sent contact data to the spurious lead webhook endpoint:

1. Apsis extracted the unique identifier (ID field) from the contact data
2. Apsis recognized the lead entity has a different key space than contact
3. Apsis created a **new empty profile** in the lead key space with only the ID field populated
4. This profile existed in parallel with the real contact profile, creating **duplicate records with no additional data**

[Erik Andersson]: "We took the ID for the contact but created a new empty profile in the lead key space. And as you said, we have no real mappings for this, so it was a empty profile that happened to have a lead ID that lived in parallel with the real contacts here."

### The Fix: Skip Webhook Registration for Virtual Entities

Erik implemented a filter in the webhook registration logic:

```python
# During installation webhook registration
for entity in connector_configuration.entities:
    if entity.should_skip_webhook_registration:
        continue  # Do not register webhook for this entity
    # Register webhook normally
    register_webhook(entity)
```

For Tribe, the lead entity is marked to skip webhook registration since:
- Tribe will never send updates to a "lead" endpoint
- Only contact updates are expected
- This prevents duplicate empty profiles from being created

[Erik Andersson]: "What I've done now is that I've added like an additional parameter saying like do not create web hooks. So during the installation process here we we retrieve all of the list or all of the entities for the connector. But before I actually add them to the list of reg webhooks to be registered, I check. If we should not create web hooks for this, like just continue, don't don't do anything so we will not have this empty profile issue for tribe."

### Root Cause Analysis: Virtual Entities Are Fragile

The core issue is that Tribe's dynamic entity selection **creates unpredictability**:

[Erik Andersson]: "Their possibility of setting this up completely dynamic creates a lot of hassle for us in this flow and it is a bit fragile. The good thing is, as far as I have been told, everyone selects contacts as the as the entity to be created and then it just so happens that. The configuration we have for contacts will be exactly the same that we have for leads, but that's just pure sheer luck that that is the case."

If a customer changes the selected entity in Tribe's configuration:
- All existing field mappings become misaligned
- The virtual "lead" configuration no longer matches the actual entity
- Webhook subscriptions are incorrect
- Syncs fail silently or create mismatched data

### Desired Long-Term Solution

[Erik Andersson]: "It needs to be reworked somehow, preferably by always using the contact entity, because then we can completely remove the lead entity on our side. We know that when we submit the form, there will be a contact. We know that you all. Only need to set up the outbound mapping, field mappings and consent mappings for the contact entity and no no more issues with incorrectly created webhooks for virtual entities."

**Recommended approach**:
- Require Tribe to always use the contact entity for form submissions
- Remove the virtual "lead" entity from the Apsis configuration entirely
- Simplify setup to only configure contact field mappings and consent mappings
- Removes fragility and one-to-one relationship between configuration and behavior

**Current constraint**: This would require coordination with Tribe's product team to standardize on contact entity.

---

## Form Submission Flow and Lead Entity Usage

### How Other CRMs Handle Lead Entities

**E-deal (generic connector implementation)**:
- When forms are submitted, E-deal responds with a "lead" entity
- Apsis receives the unique identifier in the lead response
- A profile is created in the lead key space with the Apsis silhouette ID
- This is used to track unconverted leads

[Erik Andersson]: "When they are submitting, when we are submitting forms to them, they are actually responding with their lead entity like they are not creating pure customers or contacts with this as FC enterprise. They are, they are actually. Utilizing their lead entity."

**Dynamics CRM**:
- Has a native lead entity
- Support for form submission responses prepared but not yet enabled
- Unlike E-deal, Dynamics tracks leads separately from contacts

**FSC Enterprise (generic connector)**:
- Does not have a lead concept
- Only uses contact entity
- Straightforward mapping and no virtual entity complications

**Tribe**:
- The entity returned is **whatever the customer selected in the dropdown**, not necessarily "lead"
- This unpredictability created the virtual entity problem

### Why Separate Lead Tracking Matters

Leads are tracked separately from contacts because:
- CRM systems use leads to represent **potential customers before qualification**
- When leads are created in the CRM via form submission, the CRM decides whether to:
  - Convert the lead to a contact/customer (qualified)
  - Discard it (not interested)
- **Apsis only sends data one direction**: forms → CRM
- CRM decides what to do with the lead; Apsis doesn't expect bidirectional syncing

[Erik Andersson]: "The only time we are creating profiles with the silhouette ID is when the CRM responds with it when you do a full sync. We will never download leads from the CRM system because that there has not been a use case yet where that has been interested."

### Webhook Behavior for Lead Deletes

While Apsis doesn't expect CRM systems to create or update leads, there is **one delete use case**:

If the CRM decides a lead is not qualified and sends a delete request, the webhook handles this deletion operation. Webhooks support create, update, and delete operations for all entities, but for leads:

[Erik Andersson]: "There is one use case where they could send updates to this silhouette still, and that is if they choose to not proceed with the lead, then they can send a delete request saying like, yeah, we received this lead, it's not of any interest, please delete it. We're not gonna merge this with any existing profile."

---

## Customer Issue Resolution and Support Procedures

### How Erik Fixed the Tribe Issue

Because Apsis maintains API keys for installations and uninstallation cleanup, Erik was able to directly fix the customer's issue by:

1. Accessing the customer's Tribe environment via API key
2. Using the generic connector's **delete webhook endpoint**
3. Removing the incorrectly registered lead entity webhooks

[Erik Andersson]: "So because during uninstallation we we are making requests to delete web hooks in the CRM as like a. Clean up step. That means that we have an API key that can that has access to the customer's environment and therefore. We we have the generic connector, the generic connector functions here, for example here webhook record. Delete record."

**Debugging tools available**:
- **Postman collection** for generic connector with endpoints for:
  - Creating webhooks
  - Listing webhooks
  - Deleting webhooks
  - Getting schema/mappable fields
  - Testing connectivity

[Erik Andersson]: "You can also of course utilize this same collection if you need to do some debugging for a customer. If they say why can't I see any mapping? Yeah, well when I try to make this request here to. To get the schema like I get an empty list or I get a permission denied or I get a 404 or whatever whatever it can be."

### Preventing Future Issues: SOC Team Monitoring

The support team (SOC) must monitor for similar issues in other customer accounts:

**Symptom**: Empty profiles in the audience with only silhouette IDs (no name, email, or other data)

**Root cause**: Accidentally registered lead entity webhook receiving contact data

[Erik Andersson]: "There might be other customers that are suffering from this same issue, though I don't know yet. But that's something that the SOC team will need to monitor a bit, but they know now that if there are empty profiles on from the from tribe. With only the silhouette ID. Then it is like essentially guaranteed that they also have installed after we accidentally introduced this bug."

### Critical: Route Support Through SOC, Not Direct Escalations

[Erik Andersson]: "This is why it is so important that these questions go. Via SOC and support, because now they know about this problem. We've described it to them. If there are more customers that experience the same thing, they already know what the problem is. If these questions get pinged to us directly from, say, account managers, we are circumventing that whole process and that knowledge sharing will not be there and then you will get the same question from SOC and support in the future as well."

**Key principle**: Centralizing issue reports through support ensures:
- Knowledge is documented and shared across the team
- Future similar issues are identified and resolved faster
- Account managers don't create parallel support channels that bypass the issue tracking system
- Pattern recognition across customers becomes possible

---

## PR and Implementation Status

### Changes Deployed

A PR has been submitted with the fix to skip webhook registration for virtual entities (specifically Tribe's lead entity). This change will apply to **new installations going forward**.

[Michal Rosikiewicz]: "Right, so this will work for new installations then."

[Erik Andersson]: "Yes."

### Customer-Specific Cleanup

Existing customer instances (like the one that encountered this bug) can be cleaned up manually using the generic connector's API endpoints and a Postman collection if necessary.

---

## Key Takeaways

1. **Plugin Ownership Model Has Shifted**: Apsis moved from owning plugin code and support to enabling partners to own integrations. This reduces support burden and aligns incentives with partner success.

2. **Legacy Plugins Are Dead Code**: Presta Shop, Adobe Commerce, Sitecore, Drupal, and Optimizely plugins are open-source but unsupported. Consider them unavailable for customer projects, even for existing customers.

3. **Never Volunteer to Build Custom Plugins**: The hidden costs (support, debugging, maintenance, research) mean Apsis typically loses money on custom plugin development. Always escalate to product/management for prioritization and look for external partners first.

4. **Lime CRM Plugin is Reference Only**: The archived plugin is useful for understanding how plugins work but is not functional or deployable. Plugin development complexity (especially database persistence) makes custom integrations high-risk.

5. **One API Simplifies Third-Party Integrations**: Super Office, Sleeknote, WooCommerce, and others require minimal Apsis involvement—just key generation and key space creation. The third party handles the rest.

6. **Tribe's Virtual Lead Entity Creates Fragility**: The dynamic entity selection in Tribe introduces unpredictability that breaks the generic connector's configuration model. The fix (skipping webhook registration for virtual entities) prevents duplicate empty profiles but doesn't solve the underlying architectural issue.

7. **Webhook Registration Has Unintended Side Effects**: Registering webhooks for entities that don't exist can cause the CRM to misroute data. Careful filtering of which entities get webhooks is critical.

8. **Frontend Cleanup is Needed**: Multiple obsolete integrations still appear in the customer-facing UI as "Coming Soon" or non-functional options. This is false advertising and needs frontend work to remove.

9. **Support Through Proper Channels is Critical**: Issues must be routed through SOC/support so knowledge is shared and patterns are recognized across customer base. Direct escalations bypass the issue tracking system and create information silos.

10. **E-deal is the Working Lead Entity Model**: Unlike Tribe, E-deal properly implements lead entity responses, making it the reference implementation for how other CRMs should handle lead tracking.

---

## Unresolved Questions and Action Items

### Action Items

1. **Create cleanup story** for removing defunct integrations from both backend API and frontend UI:
   - Remove: Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely, Tessitura, Google Data Studio, Ambraco, AS course, Power BI
   - Assign to Chris (frontend work)
   - Link to Erik's backend story

2. **Deploy PR** that adds webhook registration skip logic for virtual entities (specifically Tribe lead entity)

3. **SOC team monitoring**: Watch for empty profiles with only silhouette IDs in Tribe customer instances—these indicate the accidental lead webhook registration bug

4. **Long-term Tribe improvement**: Coordinate with Tribe's product team (Rose is a product owner there) to standardize on contact entity for form submissions and remove the virtual lead entity requirement

### Unresolved Issues

1. **Tribe entity selection fragility**: If a customer changes which entity is selected in Tribe's configuration, the field mappings and webhooks become misaligned. This is a known limitation with no current technical workaround (only training/customer awareness).

2. **Unknown scope of Tribe lead webhook bug**: Unclear how many customer instances have been affected by the incorrectly registered lead entity webhooks. SOC needs to proactively check customer instances.

3. **UI/Frontend expertise needed**: The integration list cleanup requires Angular/frontend work beyond the integration team's scope. Needs coordination with frontend team.

4. **Generic connector dynamic entity support**: The generic connector doesn't support truly dynamic entities. Supporting this would require significant architectural changes (not prioritized).