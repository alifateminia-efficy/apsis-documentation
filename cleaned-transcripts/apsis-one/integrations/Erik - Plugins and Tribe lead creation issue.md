---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One Integrations
topics: [Plugin Strategy and Ownership Model, Legacy Plugin Repositories, Third-Party vs Native Integrations, Generic Connector Implementation, Tribe Lead Entity Configuration Issue, Webhook Registration Bug in Tribe, Form Submission Flows, Integration Cleanup and Deprecation]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Apsis One API, Generic Connector, Tribe CRM, Microsoft Dynamics, Lime CRM, E-deal (Efficy Corporate), SuperOffice, Zapier, Webhook Registration, Key Spaces, Silhouette IDs, Lead Entity, Postman Collection for Generic Connector]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Inbound Flow, Outbound Flow, Tribe, E-deal (Efficy Corporate), Lead creation, Duplicate profiles in Apsis]
---

## Session Overview

This knowledge transfer session covered the historical and current state of plugin repositories within Apsis One Integrations, focusing on the business model shift from APSIS-owned plugins to partner-owned integrations. The majority of the discussion centered on the technical and architectural reasons behind this change, the status of legacy plugin integrations, and a critical bug discovered in the Tribe CRM integration related to virtual lead entity webhook registration that was causing duplicate profile creation. Erik also outlined the need for frontend cleanup to remove deprecated integrations from the UI.

---

## Plugin Strategy and Historical Business Model

### Initial Plugin Ownership Model

When integration was first implemented, **APSIS owned and maintained all plugins** for third-party CRM systems that required custom development. The rationale was:

- APSIS contracted external companies to build plugins for systems like **Microsoft Dynamics**, **Presta Shop**, **Episerver**, and **Optimizely**
- APSIS owned the resulting code and was responsible for all customer support
- If customers encountered issues, they would contact APSIS support first, and APSIS would coordinate with the plugin developer for fixes
- Bug fixes were provided free of charge; new features requested by customers incurred significant hourly costs from external development partners

[Erik Andersson]: > "If any customer had any issue with the plugin then the customer would reach out to APSIS. APSIS would need to investigate where it goes wrong. And if we don't see that there's something wrong in the CRM, then we need to set up a meeting with the company that develops the plugin and have them correct this."

### Strategic Shift to Partner-Owned Model

When Felix became head of APSIS, the business strategy fundamentally changed:

- **APSIS stopped building new plugins** and stopped owning support responsibility
- Instead, **external companies became business partners** and customers paid those partners directly for plugins, not APSIS
- This model shifted support burden from APSIS to the plugin developers who could easily diagnose issues from their own logs
- **APSIS revenue comes from volume of data synced**, not licensing the plugin: customers send high-volume profile data to APSIS every month through these partner plugins

[Erik Andersson]: > "Appsis no longer wanted to own any support for plugins because it takes a lot of time for us to handle that. Now we wanted to help make them like business partners. The customer is paying whoever develops the plug-in the money, so that company owns the customers."

**Benefit of this model for new features**: When APSIS adds new functionality (e.g., event tool syncing), APSIS doesn't need to update every plugin—partners have incentive to implement the new functionality themselves to improve their offering to customers.

---

## Legacy Plugin Repositories and Current Status

### Repositories Containing Unsupported Plugins

The following plugin repositories exist in the `apsis-integrations` namespace but are **no longer actively maintained or supported**:

- **Presta Shop** (e-commerce platform) — public repository
- **Adobe Commerce** (formerly Magento, e-commerce) — public repository  
- **Sitecore** (CMS) — private repository
- **Drupal** (CMS) — private repository
- **Optimizely** (optimization platform) — private repository

[Erik Andersson]: > "These old integrations that there was a plugin for except for Microsoft Dynamics, which we still have quite some customers for like they were not that successful. We essentially had no customer in the ones that we had, they churned. The decision was made that APSIS would completely drop the support for these connectors."

### Open Source Status and Customer Responsibility

These legacy plugins are treated as **open source**:

- Customers can still download and attempt to install them
- Source code is available for customers to modify or extend
- **APSIS provides zero support** for these integrations, even for existing customers who may have paid for them
- If a customer has deployed a legacy plugin (e.g., Drupal), they are responsible for maintenance and debugging
- Only exception: if a new product decision requires building a new connector, someone else (preferably an external partner, not the integration team) should build it

[Michal Rosikiewicz]: > "You should not give any support for them. The only case would be if product comes and says like now we want a brand new Presta shop connector. Then someone would need to do something with general connector for it."

[Erik Andersson]: > "No, no, we we don't. There there is no support even for existing customers."

### Why These Plugins Exist in the Repository

Legacy plugins are retained rather than deleted because:

1. **Preservation of reference implementations**: They serve as examples of plugin architecture and structure (how to build components, naming conventions, generation patterns)
2. **Sunk cost reasoning**: It was deemed wasteful to completely remove them after migration to new repository structure
3. **Future generic connector possibilities**: If a CRM like Lime ever requires integration, the plugin repository pattern could be reused as reference

---

## Lime CRM Plugin: Historical Experiment

### Background and Motivation

Lime CRM is a commercial CRM system where APSIS had good relationships with the company (former APSIS employees worked at Lime). The integration team attempted to build a custom plugin to:

- Retrieve email events from APSIS One for specific contacts (feature that existed in the legacy Apsis Pro system)
- Retrieve website events from APSIS One for specific contacts  
- Display this data in a custom Lime UI element

[Erik Andersson]: > "This was an attempt to replicate this feature inside of Lime because it doesn't exist. Lime doesn't natively know how to retrieve Apsis one events, so this plugin was an attempt by us to add a custom UI element inside of Lime and also adding custom endpoints inside of Lime."

### Technical Challenges and Project Abandonment

The implementation faced significant obstacles that were never resolved:

- **Database access and persistence issues**: Complex configuration required to keep customer-specific data isolated when instances are shared across multiple accounts; easy to accidentally persist data globally instead of per-customer
- **Inefficient implementation**: Could not find an efficient way to handle data isolation
- **Waning interest**: Customer interest in this functionality declined rapidly

[Erik Andersson]: > "There were unfortunately some quite complicated configuration like with database access and persistence like how do you keep the data in the instance for only one customer? If you didn't configure it in the right way or use the correct library, you could accidentally like persist that data for everyone on the whole environment."

The plugin code remains in the repository for **reference purposes only** and is no longer functional or maintained.

### Why Plugin Building Should Be Avoided

Erik strongly advised against building custom plugins going forward:

[Erik Andersson]: > "Nowadays I would not suggest anyone in your team sit and build custom plugins because then again you will have to own the maintenance or support and bug fixing for that and you you don't want to do that."

The cost implications of building a plugin include:

- Ongoing maintenance and bug fixes (hours "magically disappearing")
- Continuous debugging and customer support
- Additional research required to understand the CRM system
- Often unprofitable even at premium pricing

[Erik Andersson]: > "This has been a recurring question like oh, this super big customer wants this. Can't you build this like no, like I I don't care if they pay me like 500,000 like we will lose money on this from R&D perspective."

**Recommended approach**: Direct customers to external consulting partners (e.g., **Silverlord Consulting**) who can build and own plugins instead.

---

## Overview of Integration Repository Structure

### Third-Party Integrations (No Plugin Required)

These integrations use **APSIS One API exclusively** and require no custom plugin installation:

- **SuperOffice** (CRM) — using Generic Connector
- **Sleek Note** (marketing platform)
- **WooCommerce** (e-commerce)
- **I am Loyalty** (loyalty program platform) — with special consent mapping support
- **Join CX** / **Join Loyalty** (loyalty platform) — full Generic Connector implementation by Intermail

For these integrations, APSIS's role is limited to:

1. Generating an APSIS One API key for the customer during setup
2. Creating a key space in Sleek Note configuration for those requiring it
3. All data flow and configuration happens via the One API or within the third-party system

### Native Integrations (APSIS-Owned)

These are maintained and developed by APSIS or the FSC team:

- **Microsoft Dynamics 365** (CRM) — still actively supported; significant customer base
- **E-deal** / **Web CRM** (Efficy systems) — implemented via Generic Connector
- **Maxo** — discontinued (being removed)
- **FSC Enterprise 12.0** (legacy CRM system)
- **FSC Enterprise 12.1** (current CRM system)
- **Tribe** (CRM) — currently experiencing issues (see separate section below)

---

## Zapier Integration

### Repository and Maintenance

The **`apsis-integrations-zapier`** repository contains the source code for the APSIS One Zapier application:

- APSIS owns and maintains this application
- The application code is built locally and then packaged and uploaded to Zapier's platform
- Moderate customer usage; considered stable with minimal ongoing development needed

[Erik Andersson]: > "I don't think there's gonna happen much for Zapier like it's just running there and existing, but there are quite some customers utilising it."

### Implementation Details

The Zapier integration uses **APSIS One API exclusively** (100% API-driven):

- All Zapier operations route through the One API
- No custom integrations or special handling required
- Customers use Zapier's native trigger/action interface to connect with APSIS One

---

## Microsoft Dynamics and Cold Storage Instance Issue

### Cold Storage Behavior

**Microsoft Dynamics instances automatically enter "cold storage" mode** if unused for a period of time:

- First API request to a cold instance can take an extremely long time (described as "obscenely long")
- Previously caused installation failures when using Lambda/API Gateway architecture: first API request would timeout before installation could complete

[Erik Andersson]: > "If a instance has not been utilized for some time, it kind of goes into the cold storage. If you make any API requests to the instance and it hasn't been utilized like the first API request to, it can take an obscenely long time."

### Architecture Solution

Current architecture using **ECS tasks and load balancers** resolves this issue:

- ECS tasks have longer timeout tolerances than API Gateway
- Installation process no longer fails on first cold instance access
- Load balancer maintains persistent connection capacity

### Historical Feature: Event Retrieval from APSIS

Dynamics included a historical feature (no longer used):

- **Contact details page in Dynamics** displayed an "Apsis" tab
- This tab showed all email events and website events for that contact retrieved from APSIS One via API
- This feature existed because it had precedent in the legacy Apsis Pro system
- Feature is now discontinued and no customers use it

---

## Tribe CRM Integration: Virtual Lead Entity Bug

### Context: Generic Connector and Entity Mapping

The Tribe integration uses the **Generic Connector** architecture with field mappings, consent mappings, and full synchronization support. However, Tribe has a unique architectural characteristic that creates complexity.

### The Virtual Lead Entity Problem

In Tribe, **all entities are dynamically configurable** by the customer:

- When setting up APSIS integration in Tribe, the customer must select which entity should be created when APSIS sends form submissions
- This entity selection is **not static**—the customer chooses from a dropdown which entity type to use (e.g., "Contact," "Potential Customer," "Prospect," etc.)

[Erik Andersson]: > "One mandatory configuration that you have to do in tribe before you start using APSIS is that all of the entities in tribe and all of your custom entities in tribe that you can utilize you select from this list, which entity should be created inside of tribe when there is a form submitted?"

**APSIS handles this dynamism by using a virtual "lead" entity**:

- When APSIS needs to request available fields for mapping configuration, it asks Tribe for attributes of the "lead" entity
- Tribe actually returns the attributes of **whatever entity the customer selected in the dropdown**
- This virtual entity exists only in APSIS configuration; it does not exist in Tribe

### How Lead Creation Normally Works (Other CRMs)

In **E-deal** and **FSC Enterprise**:

- Customer submits a form in APSIS
- APSIS sends form data to CRM
- CRM responds with the created entity (silhouette ID for E-deal, contact ID for FSC)
- APSIS stores this ID in the key space to maintain bidirectional sync

**Tribe's different behavior**:

- Customer submits a form in APSIS
- APSIS sends form data to Tribe
- Tribe responds with whatever entity type was configured (usually "Contact," but could be "Potential Customer," etc.)
- Tribe **never responds with "Lead"** because that entity doesn't actually exist in their system

### The Bug: Incorrect Webhook Registration

During installation, the integration platform registers webhooks for all entities in the Generic Connector configuration:

[Erik Andersson]: > "During installation, we retrieved all of the list or all of the entities for the connector. Before I actually add them to the list of registered webhooks, we now check if we should not create webhooks for this."

**The problem**: APSIS was registering a webhook for the virtual "lead" entity, even though Tribe would never send updates to it:

- Webhook registered for: `lead` entity updates
- Webhook registered for: `contact` entity updates (the real entity)
- Tribe's behavior: When a form is submitted, send contact data to **both** the contact webhook endpoint **and** the lead webhook endpoint (because both were registered)

### The Duplicate Profile Issue

Because of the incorrect webhook registration:

1. Form submission triggers: Tribe sends contact data to both webhooks
2. Contact webhook processes correctly: APSIS creates a profile with contact ID and proper mappings
3. Lead webhook processes incorrectly: 
   - APSIS interprets this as "a new lead was created"
   - Looks for the unique identifier field in the request
   - Finds the same ID as the contact (because it's contact data)
   - **Creates a new profile in the "lead" key space with only the ID, no other data**
   - Results in an empty duplicate profile with only a silhouette ID

[Erik Andersson]: > "They send us the data for the contact to the contact webhook endpoint. But because we had registered the lead endpoint, they sent the same contact data to the lead endpoints, but because the lead endpoint for us is a completely different, completely different entity that has completely different mappings, we interpret this as 'Oh yeah, someone has created a new lead inside of Tribe.'"

**Consequences**:

- Two profiles created for each form submission: one valid contact profile, one empty lead profile
- No mappings configured for the virtual lead entity
- Difficult to debug because the duplicate profiles appear in parallel with correct profiles

### Investigation Evidence

In the audit logs, Erik showed:

```
Profile with silhouette ID: [ID]
Lead entity (virtual) key space: [ID with no other attributes]
Contact entity key space: [ID with all proper mappings and data]
```

Both profiles live in parallel in the audience until the bug is fixed.

### Solution Implemented

Erik added a **conditional check during webhook registration**:

```
If (entity should not have webhooks registered) {
  Continue to next entity
  Do not register webhook
}
```

Now, the virtual lead entity webhook is **not registered** during installation:

- Only contact entity webhooks are registered
- Tribe receives no instructions to send lead data anywhere
- Contact data is sent only to the contact endpoint (where it belongs)
- No duplicate profiles created

[Erik Andersson]: > "What I've done now is that I've added like an additional parameter saying like do not create webhooks. So during the installation process here we retrieve all of the list or all of the entities for the connector. But before I actually add them to the list of registered webhooks to be registered, I check if we should not create webhooks for this."

### Why This Bug Occurred

The root cause is **Tribe's dynamic entity selection model**:

- APSIS designed the virtual lead concept to handle Tribe's flexibility
- But the generic webhook registration logic didn't account for virtual entities
- Previous implementation treated all configured entities as "real" entities that needed webhooks
- The mismatch between the virtual entity concept and webhook registration assumptions created the bug

### Workaround for Existing Customers

For customers already affected by this bug, Erik manually cleaned up using the **Postman Generic Connector collection**:

- API key for customer's Tribe environment
- Directly delete incorrect webhooks for the lead entity via the Generic Connector endpoints
- This prevents future duplicate profile creation

[Erik Andersson]: > "Because during uninstallation we make requests to delete webhooks in the CRM as a clean up step. That means that we have an API key that can access the customer's environment. We have the generic connector functions here, for example here webhook record, delete record. If you have the API key like directly because you're working with some account manager, you can configure it here inside of the collection and then delete all of these things on your own."

### Why This Knowledge Matters for Support Channels

This bug highlights why **all customer issues should flow through the SOC team, not directly to the integration team**:

1. **Knowledge sharing and documentation**: Once SOC/support understands a problem, they can identify it in future cases
2. **Avoiding duplicate investigation**: Direct routing to engineers causes repeated work for the same issue
3. **Prevention of circumventing the process**: If account managers bypass support channels, documentation doesn't improve

[Erik Andersson]: > "This is why it is so important that these questions go via SOC and support, because now they know about this problem. If there are more customers that experience the same thing, they already know what the problem is. If these questions get pinged to us directly from, say, account managers, we are circumventing that whole process and that knowledge sharing will not be there."

### Monitoring for Other Affected Customers

Erik put up a PR to fix the bug for new installations. The SOC team should monitor for:

- Empty profiles in Tribe instances with only silhouette/lead IDs
- Patterns of duplicate profiles appearing after form submissions
- These symptoms indicate the customer also has the old webhook registration issue

---

## Ideal Tribe Configuration (Recommended Future State)

### Problem with Current Flexibility

The virtual lead entity creates unnecessary complexity:

- Field mappings configuration becomes ambiguous: "What fields should I map when I don't know which entity will be created?"
- Consent mapping setup is fragile: if customer changes the selected entity in Tribe, all configuration breaks
- Webhook registration requires special handling because entity mappings are unknown at install time

[Erik Andersson]: > "Their possibility of setting this up completely dynamic creates a lot of hassle for us in this flow and it is a bit fragile."

### Proposed Solution: Static Contact Entity

**Always use "Contact" as the entity for form submissions**:

- Simplifies configuration to a known, static entity
- Eliminates the virtual lead entity from APSIS configuration
- Customers configure field mappings once for the contact entity, knowing this is what will be created
- Webhook registration becomes straightforward
- If Tribe customer later needs a different entity type, they would configure it separately as a different integration

[Erik Andersson]: > "What I would really want to do for tribe is I would want to do like this. I wanted them to always give us like some static entity so we have some kind of predictability."

### Current Workaround (Status Quo)

Currently, the system works because **all customers happen to select "Contact"** as their entity:

- By luck, the field mappings configured for the virtual "lead" entity match the contact entity mappings
- If a customer ever selected a different entity, the entire configuration would break
- This is not a reliable long-term solution

[Erik Andersson]: > "The good thing is, as far as I have been told, everyone selects contacts as the entity to be created and then it just so happens that the configuration we have for contacts will be exactly the same that we have for leads, but that's just pure sheer luck that that is the case."

### Product Owner Engagement

Rose (product owner for Tribe) is working to standardize this. Since Rose now has influence at Tribe, there may be opportunity to guide Tribe's configuration UI to require or default to contact entity selection.

---

## Integration Cleanup and Deprecated Integrations

### Frontend Visibility Problem

Many deprecated integrations are still visible in the APSIS customer UI:

**Deprecated integrations still showing:**

- Presta Shop (pre-launch, plugin unsupported)
- Adobe Commerce (unsupported plugin)
- Sitecore (unsupported plugin)
- Drupal (unsupported plugin)
- Optimizely (unsupported plugin)
- Tessitura (completely deleted from codebase)
- As Course (deleted)
- Google Data Studio (deleted)
- Ambraco (deleted)
- Power BI (can be removed)

**Still active and supported:**

- WooCommerce
- I am Loyalty
- Enterprise D
- Drupal (only if explicitly kept)

[Erik Andersson]: > "So now now you can support the Facebook lead ads by utilising a Zapier. Like Zapier does have real-time triggers for like new leads, so you can funnel them into apps is that way if you would if you would want to."

### Architecture of Visibility Control

The integration UI has two layers of control:

1. **Backend list** (`apsis-integrations`): Returns available integrations and whether they're installable
2. **Frontend UI components**: Displays integration cards and "Connect" buttons

The backend integration list controls:

- Whether a "Connect" button appears (marked as `available: true/false`)
- Whether the card shows "Read More" instead of "Connect"
- Which integrations appear in the list at all

**Problem**: The backend list was recently cleaned up to remove deprecated integrations, **but the frontend UI components still reference the old card definitions**, so cards still appear.

### Cleanup Task

Erik committed to creating a story for Chris (frontend developer) to:

1. Remove deprecated integration card UI components
2. Ensure backend list and frontend card definitions are synchronized
3. Handle any data model implications from removing cards

[Michal Rosikiewicz]: > "It will be really quick for Chris. So I think creating a story for them it's it's totally OK and just connect it with your story on our board and we will take care about it."

Deprecated integrations to remove from UI:

- Presta Shop
- Adobe Commerce  
- Sitecore
- Tessitura
- As Course
- Google Data Studio
- Ambraco
- Power BI
- Optimizely (if not already removed)

[Erik Andersson]: > "Yeah, because this is aching to false advertisement because it hasn't been like this for quite some time."

### Historical Casualties: Integrations That Were Built But Never Released

Several integrations were significantly developed but never made it to production:

**Shopify**: 

- Complete connector was built; team even worked overtime to meet rushed timeline
- Upon completion, discovered Shopify licensing model: **12% of annual APSIS revenue** required if APSIS acts as integration provider
- Entire connector was scrapped due to cost
- Not a single customer was acquired for Shopify

[Erik Andersson]: > "We did build a complete Shopify connector. We were even asked to work overtime because it got super, super rushed. And then when we had finished it, someone discovered that if we utilize Shopify in this way, then we have to pay Shopify 12% of the annual apps's revenue."

**E-commerce**:

- Integration build started
- Abandoned for budget reasons before completion
- Still marked "coming soon" in some systems

**Facebook Lead Ads**:

- Attempted integration resulted in a mishmash of ownership
- Integration team handled authentication
- "What" team (now dissolved) was supposed to handle the logic
- The entire "What" team was eventually let go
- No Facebook Lead Ads connector exists

[Erik Andersson]: > "Facebook lead ads. That was a hot potato that people tried to push it on integration, but we didn't build any custom integrations. So then they tried to push it on the what team and they didn't want to do it and then it became some mishmash that we handled the authentication for it."

**Workaround for Facebook Lead Ads**: Customers can use **Zapier** with Facebook Lead Ads real-time triggers to funnel leads into APSIS via the Zapier integration.

---

## Key Takeaways

1. **Plugin ownership strategy changed fundamentally**: APSIS moved from owning plugin code and support to partnering with external companies. Partners now own customers and provide support; APSIS earns revenue from data volume.

2. **Legacy plugins are unsupported**: Presta Shop, Adobe Commerce, Sitecore, Drupal, and Optimizely repositories exist as open-source references but receive zero support, even for existing customers. Do not volunteer to build new plugins.

3. **Virtual entity concept in Tribe creates complexity**: The dynamic entity selection in Tribe led to a critical bug where empty duplicate profiles were created due to incorrect webhook registration for a non-existent "lead" entity.

4. **Webhook registration now has guards**: The fix prevents webhook registration for entities marked as virtual. This prevents duplicate profile creation for new Tribe installations.

5. **Tribe's ideal future state**: Standardize to always use static "Contact" entity instead of dynamic selection to eliminate configuration fragility.

6. **Frontend cleanup is needed**: Many deprecated integrations still appear in customer UI. The backend was cleaned but frontend card components were not. A dedicated story is needed to remove old card definitions.

7. **Support channel matters**: Customer issues should flow through SOC/support teams, not directly to engineers. This ensures knowledge sharing and prevents repeated investigations of the same bug.

8. **External consulting is the model**: When customers need new plugin integrations, direct them to external consulting partners (e.g., Silverlord Consulting) rather than APSIS building them internally.

---

## Unresolved Questions and Action Items

### Action Items:

1. **Create cleanup story** (Erik): Remove deprecated integrations from UI (Presta Shop, Adobe Commerce, Sitecore, Tessitura, As Course, Google Data Studio, Ambraco, Power BI). Coordinate with Chris (frontend developer) for UI component removal.

2. **Monitor for affected Tribe customers**: SOC team should watch for empty profiles with only silhouette/lead IDs, indicating customers affected by the pre-fix webhook registration bug.

3. **Standardize Tribe entity selection** (Product/Rose): Work with Tribe to standardize form submission entity to always be "Contact" rather than dynamic selection.

### Open Questions:

1. **Existing Drupal customer support**: One customer is still using a customized Drupal installation. Confirmation needed on whether any support should be provided (answer: no, but don't delete repository).

2. **Tribe configuration auditing**: Unknown how many existing Tribe customers have the webhook registration bug. May need audit of deployed configurations.

3. **I am Loyalty vs Join CX migration timeline**: When will all I am Loyalty customers migrate to Join CX? This affects how long support burden continues for I am Loyalty.