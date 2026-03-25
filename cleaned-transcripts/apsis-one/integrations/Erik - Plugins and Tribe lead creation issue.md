---
source_file: Erik - Plugins and Tribe lead creation issue.txt
domain: Apsis One Integrations
topics: [Plugin Repository Structure, Historical Business Model Changes, Unsupported Legacy Connectors, Third-Party Integrations, Generic Connector Implementation, Tribe Lead Entity Issue, Webhook Registration Problem, Customer Support Model]
speakers: [Erik Andersson, Michal Rosikiewicz, Tomasz Kowalski]
key_components: [Plugin repositories, Generic Connector, Webhooks, E-deal, Tribe, Postman collection, Lead entity, Silhouette ID, Contact entity, Key space management]
session_type: knowledge-transfer
subdomains: [Architecture, Different Types of Connectors, Generic Connector, Outbound Flow, Tribe, Lead creation]
---

## Session Overview

This session covers the architectural history and current state of the Apsis One Integrations domain, focusing on plugin repositories, the shift from owned plugins to partner-owned integrations, and a critical bug in Tribe lead creation. Erik explains why certain legacy connectors (Presta Shop, Adobe Commerce, Sitecore, Drupal, Optimizely) are maintained as unsupported open-source code, contrasts this with modern third-party integrations, and deep-dives into a production issue where Tribe's virtual lead entity caused duplicate empty profiles to be created due to incorrect webhook registration. The session emphasizes the importance of routing support questions through proper channels (SOC/support teams) rather than directly to development.

---

## Historical Context: Plugin Business Model Evolution

### Original Strategy (Pre-Felix Era)

[Erik Andersson]: The original integration strategy was that for any CRM systems requiring a plugin installation—such as Microsoft Dynamics, Presta Shop, Episerver, or Optimizely—APSIS would pay an external company to develop the plugin. APSIS then owned both the code and support responsibilities.

The business model meant:
- APSIS owned the developed code
- APSIS owned all customer support for the plugin
- When customers reported issues, APSIS would investigate and escalate to the plugin developers if needed
- Bug fixes were free; feature requests cost significant money per hour
- This model applied to: Presta Shop, Adobe Commerce (formerly Magento), Sitecore, Drupal, and Optimizely

### Strategic Shift Under New Leadership

[Erik Andersson]: When Felix became head of APSIS, the business model fundamentally changed. APSIS no longer wanted to own plugin support because:

1. It consumed too many internal resources
2. Third-party developers could diagnose issues faster from their own logs
3. APSIS wanted to transition to a partner model where external companies build and maintain plugins

**New Partner Model:**
- External companies (e.g., SideShop for Microsoft Dynamics) build and maintain the plugin
- Customers pay the plugin developer directly, not APSIS
- The plugin developer owns all support responsibility
- APSIS benefits from data volume: "Sideshop is shipping a crap load of profiles per month in our direction"
- APSIS only provides support if the partner confirms they're sending correct data but receiving errors

**Benefits of Partner Model:**
- When APSIS adds new features (e.g., event tool syncing), the partner is incentivized to implement support for those features because it increases their product value
- APSIS doesn't need to update 10+ connectors individually

---

## Legacy Plugin Repositories: Status and Maintenance

### Unsupported Open-Source Connectors

[Erik Andersson]: The following connectors are no longer actively supported but are maintained as open-source reference code:

- Presta Shop
- Adobe Commerce
- Sitecore
- Drupal
- Optimizely

**Current Status:**
- Not actively developed or maintained by APSIS
- Available for customers to download and modify (source code is public for Presta Shop and Adobe Commerce; private for others)
- APSIS provides **zero support** for these, even for existing customers
- No customer usage: "we essentially had no customer in the ones that we had, they churned"

**Why Keep Them:**
> "We still offer them like we have the plugins available here like if any customer would want to like in theory they could come and download the site core plugin. They can try and install it and if it works then all good. But if there are any problems like yeah I tough luck but we are not making any more updates to it."

These are considered **open-source reference implementations**. If a customer truly needs integration with these systems, they can:
- Download the source code
- Modify it to add functionality or fix bugs themselves
- Or hire an external company to maintain it

**Important Policy:**
[Erik Andersson]: Even if a customer has paid for and is using one of these connectors, APSIS will not provide support. The only exception would be if Product decides to rebuild a connector from scratch using the Generic Connector architecture.

> "Even if they paid for it, yes. [no support]"

### Cleanup Needed

[Erik Andersson]: Many of these integrations are marked as "pre-launch" in the UI and should be removed from the customer-facing integration marketplace. Examples include:
- Presta Shop (still showing as pre-launch despite no active development)
- Tessitura (deleted)
- Adobe Commerce (should be deleted)
- Google Data Studio (deleted)
- Ambraco (deleted)
- Power BI (marked for removal)

The cleanup requires both backend changes (removing from the integration list API) and frontend changes (removing UI cards for the integration carousel). This is backend work for the integration team and frontend work for Chris.

---

## Lime CRM Plugin: Historical Attempt and Lessons Learned

### Background

[Erik Andersson]: Approximately four years ago, APSIS attempted to build a custom plugin for Lime CRM to replicate a feature that existed in the legacy Dynamics integration: displaying email events and website events for a specific contact.

[Michal Rosikiewicz]: So this is basically what other companies do right now for other CRMs, right?

[Erik Andersson]: Exactly, yes. This was way before the Generic Connector was born. This was only to be able to retrieve email events and website events for a specific contact from Apsis One.

### Technical Challenges

The Lime plugin implementation encountered several complications:

1. **Database Persistence Issues:** Lime's plugin framework required careful configuration to ensure data persisted only for individual customers, not across the shared environment. Mistakes could accidentally expose one customer's data to all users on that environment.

2. **Custom Endpoints Required:** The plugin needed custom UI elements and custom API endpoints within Lime to retrieve data from APSIS.

3. **Complexity:** The combination of custom endpoints and database persistence made the implementation fragile and difficult to maintain.

[Erik Andersson]: There were unfortunately some quite complicated configuration like with database access and persistence like how do you keep the data in the instance for only one customer? Like it was apparently very easy to make a mistake and if you have one instance that you share with multiple accounts or customers, if you didn't configure it in the right way or use the correct library, you could accidentally like persist that data for everyone on the whole environment and not for like one particular customer.

### Why It Was Abandoned

- Interest in the functionality declined quickly
- APSIS never successfully got the feature working reliably
- The effort required wasn't justified by customer demand

### Why the Repository Remains

[Erik Andersson]: We wanted to preserve it as a reference implementation showing how to build a Lime plugin. If APSIS ever needs to implement a Generic Connector for Lime in the future (e.g., if a "major surge" in Lime demand occurs), developers can reference this code to understand:
- Plugin framework structure
- How to implement custom endpoints
- How to name components
- How to generate UI components

> "Nowadays I would not suggest anyone in your team sit and build custom plugins because then again you will have to own the maintenance or support and bug fixing for that and you you don't want to do that. But it could be good to know like it is possible to do it for line and like we have a example projects for how we tried to to do it."

### Access and Development Story

[Erik Andersson]: When building the plugin, I was able to get a test environment from Lime because of good relationships there. Several APSIS employees had previously moved to Lime, which helped facilitate access.

[Michal Rosikiewicz]: How how it worked because you had to ask someone for the slime CRM access or what?

[Erik Andersson]: Lime has quite good documentation, and there were people at APSIS with good contacts at Lime. Benjamin had connections there, and Lime was surprisingly willing to give us test environments because we said we wanted to integrate with Lime. I had full admin access to the test environment and knew how to develop in Python, which no one else on the team could do at the time.

The development attempt read through the tutorials on building Lime plugins, but the database persistence and multi-tenancy issues proved insurmountable.

---

## Policy on Building Custom Plugins

### When NOT to Build

[Erik Andersson]: Do not volunteer to build custom plugins for new CRM systems. If approached by Professional Services with a high-value customer request, escalate the decision to Product Management.

> "Important thing, it's not you that decide that that needs to be some someone else need to do that decision and priority because it has. If you say that like sure you build it against your desire, then they have to know that now there will be hours just magically disappearing because now because you are building this, you need to support it. You need to do the debugging. You will need to do even more research on how to do that in this system."

**The Real Cost:**
- You become the maintainer and debugger forever
- Hours disappear into ongoing support and bug fixes
- R&D costs can exceed revenue from high-value customers
- Even a customer paying $500,000 can become unprofitable from an R&D perspective

> "This has been a recurring question like oh, this super big customer wants this. Can't you build this like no, like I I don't care if they pay me like 500,000 like we will lose money on this from R&D perspective."

### Alternative: Partner with External Consultants

[Erik Andersson]: APSIS can recommend external consulting companies like Silverlord Consulting to build and maintain plugins, keeping the support burden external.

> "We can say that, uh, you know, we have this, uh, external uh Company Silverlord Consulting that you can ask for help."

[Michal Rosikiewicz]: How how it worked because you had to ask someone for the slime CRM access or what?

[Erik Andersson]: Yeah, so I was a consultant when I first joined APSIS and we were allowed to utilize our own accounts. It was only when Benjamin went on parental leave that I actually became an actual employee.

---

## Dead and Deleted Integrations: Lessons Learned

### Shopify: Licensing Deal Breaker

[Erik Andersson]: APSIS built a complete Shopify connector and was asked to work overtime because the project was rushed. After completion, a licensing discovery made the entire connector unusable.

> "If we utilize Shopify in this way, then we have to pay Shopify 12% of the annual APSIS revenue. So we have to scrap the whole connector."

**Outcome:** The entire connector was deleted. The Shopify integration now routes through Zapier instead.

### E-Commerce Integration: Budget Cuts

[Erik Andersson]: APSIS started building an e-commerce integration but the collaboration was scrapped for budget reasons. No plugin was ever completed.

### Facebook Lead Ads: Ownership Confusion

[Erik Andersson]: Facebook Lead Ads became a "hot potato" that got pushed around internally:
1. Tried to assign to Integration team — declined
2. Tried to assign to Product team (what team) — declined
3. Ended up split: Integration handled authentication; what team built the logic
4. The entire what team was later let go

**Outcome:** No Facebook Lead Ads connector. Now customers can use Zapier, which has real-time triggers for new leads that funnel into APSIS.

### Other Deleted Integrations

- Tessitura (deleted)
- Ascose (deleted)
- Google Data Studio (deleted)
- Ambraco (deleted)

---

## Modern Third-Party Integrations

### Integration Patterns

**Third-Party integrations that don't require plugins fall into two categories:**

#### Category 1: Direct One API Usage

These integrations use **One API exclusively** for all communication with APSIS. APSIS' role is minimal:

**Integration Setup Process:**
1. Customer clicks "Connect" on the integration card
2. APSIS generates a One API key for the customer
3. APSIS creates a key space in Sleeknote (if needed)
4. Everything else is configured within the third-party system

**Examples:**
- Sleeknote
- Super Office
- I Am Loyalty (mostly)
- Join CX

[Erik Andersson]: The only thing we really do in integration for these third party ones is bootstrap the key space, generate One API key and then then we are done.

#### Category 2: Generic Connector Implementation

These integrations fully implement the Generic Connector protocol with field mappings, sync configurations, and full integration control.

**Examples:**
- Join CX (also listed above)
- Tribe

---

## I Am Loyalty: Special Case (Consent Export Required)

[Erik Andersson]: I Am Loyalty is mostly One API but with one exception: they need consent changes exported back to them, which One API doesn't support.

**Setup:**
- One API handles 95% of communication
- APSIS sets up consent mappings to listen for consent changes in APSIS
- When consent changes occur, APSIS sends updates back to I Am Loyalty via webhook
- I Am Loyalty is the predecessor to Join CX (same company: Intermail)

**Context:**
Intermail is heavily integrated with APSIS. APSIS handles marketing campaigns; Intermail handles loyalty programs (points, offers, etc.). The relationship benefits both platforms.

[Erik Andersson]: Intermail are quite heavily integrated with Apsis like we have very good collaboration with them. Because the Apsis handles the marketing, but they handle like these loyalty programs. So like they keep track of points and they handle offers and etcetera.

---

## Generic Connector: Native Integrations

### Officially Maintained Connectors

Connectors marked as "Native" are owned and maintained by FSC (Apsis One):

- **FSC Enterprise 12.0** (old implementation)
- **FSC Enterprise 12.1** (current implementation — this is what new developers will work with)
- **E-deal** (Web CRM integration using Generic Connector)
- **Maxo** (discontinued, marked for removal)

**Setup for Native Integrations:**
Customers don't install anything. They're taken to a configuration page where APSIS hosts the integration.

---

## The Tribe Lead Entity Issue: Deep Dive

### Problem Statement

Tribe's Generic Connector configuration includes a "Lead" entity, but this lead entity is **not a real entity**—it's a **virtual representation** of the entity selected in Tribe's dropdown configuration. This architectural mismatch caused duplicate empty profiles to be created in APSIS.

### Root Cause: Virtual vs. Real Entities

**In Tribe's System:**
Before using APSIS, the customer must configure which Tribe entity should be created when forms are submitted. This is a dropdown selection. Common choices:
- Contact
- Potential Customer
- Lead
- Or any custom entity

**The Problem:**
APSIS configured webhooks for the virtual "Lead" entity during installation, but Tribe doesn't create a literal "Lead" when forms are submitted—it creates whatever entity the customer selected in the dropdown.

[Erik Andersson]: When we ask when we tell tribe like please give me the mappable fields for the lead. Because I want, I may want to set up the outbound mappings... we are using this virtual entity lead when we ask Tribe for the attributes for leads, they are to give us the attributes for the entity that is selected in the dropdown. But when they respond in the form submission, if they have selected like potential customer as their entity, they will respond with potential customer.

### How the Bug Manifested

1. **Installation:** APSIS registered webhooks for the Lead entity because it appeared in the configuration
2. **Real-Time Sync:** When Tribe sent contact updates via webhook, they sent the same data to multiple endpoints (both Contact and Lead webhooks, since we'd registered both)
3. **Data Interpretation:** APSIS received contact data on the Lead webhook endpoint
4. **Profile Creation:** APSIS created a new profile in the "Lead" key space with just the ID from the contact
5. **Result:** Empty profiles appeared in APSIS, existing only with a Silhouette ID and no field mappings

[Erik Andersson]: So somehow interpret this at they are sending the same data like they send us the data for the contact to the contact webhook endpoint. But because we had registered the lead endpoint, they sent the same contact data to the lead endpoints, but because the lead endpoint for us is a completely different, completely different entity that has completely different mappings, we interpret this as Oh yeah, someone has created a new lead inside of Tribe. Cool. Let's create like a new profile with it.

### The Lead vs. Silhouette ID Distinction

It's important to understand APSIS' key space concept:

- **Contact ID:** The real contact in the CRM (e.g., "Contact-123")
- **Silhouette ID:** A virtual profile representing a lead or incomplete contact in APSIS, waiting for the CRM to decide whether to convert it
- **Lead Key Space:** A separate key space in APSIS for lead-only profiles

In normal operation:
- Forms submit to Tribe
- Tribe optionally creates a Lead entity
- If Lead is created, Tribe sends back a Silhouette ID
- APSIS creates a profile with the Silhouette ID in the Lead key space
- Later, when the lead converts to a Contact, the CRM sends the Contact ID
- APSIS merges the silhouette profile with the real contact

But with the bug:
- Tribe sent Contact data to the Lead webhook (because both were registered)
- APSIS interpreted Contact data as Lead data
- APSIS created empty profiles with mismatched key space and mappings

### The Fix

[Erik Andersson]: I've added an additional parameter saying like do not create webhooks. So during the installation process here we retrieve all of the list or all of the entities for the connector. But before I actually add them to the list of reg webhooks to be registered, I check if we should not create web hooks for this, like just continue, don't do anything so we will not have this empty profile issue for tribe.

**Deployed Changes:**
1. Added a flag to prevent webhook registration for the virtual Lead entity
2. No webhooks registered for Lead during installation
3. Only Contact and other real entities get webhooks

**Deletion for Existing Customers:**
For customers already affected, APSIS has the ability to delete webhooks directly using the Generic Connector Postman collection (see below).

### Ideal Long-Term Solution

[Erik Andersson]: What I would really want to do for tribe is I would want to do like this. I I wanted them to always give us like some static entity so we have some kind of predictability because right now like what mappings should you set up when we send the send it to them? Like what consents here should you configure for the initial consent like it's it's not a well working flow.

**Proposed Change:**
- Require Tribe customers to **always use Contact** as the entity for form submissions
- Remove the virtual Lead entity concept entirely from APSIS
- Simplify mappings: only Contact field mappings, Contact consent mappings, Contact outbound sync

**Current Workaround:**
"The good thing is, as far as I have been told, everyone selects contacts as the as the entity to be created and then it just so happens that the configuration we have for contacts will be exactly the same that we have for leads, but that's just pure sheer luck that that is the case."

> "It needs to be reworked somehow, preferably by always using the contact entity, because then we can completely remove the lead entity on our side. We know that when we submit the form, there will be a contact. We know that you all only need to set up the outbound mapping, field mappings and consent mappings for the contact entity and no no more issues with incorrectly created webhooks for virtual entities."

**Note:** Rose is now a Product Owner for Tribe, which provides "a backdoor" for improving the integration architecture.

---

## Manual Debugging and Cleanup: Using the Postman Collection

### Generic Connector Postman Collection

The Integration team maintains a Postman collection that includes endpoints to manage webhooks for any customer environment. This is useful for both debugging and cleanup.

**Available Operations:**
- Create webhooks
- List registered webhooks
- Delete webhooks (cleanup)
- Get schema/mappings
- Test API connectivity

**How to Use:**
If you have:
1. The customer's environment URL
2. An API key with permissions for that environment

You can configure the Postman collection with these values and execute requests directly, allowing you to:
- Debug why mappings aren't showing (permission denied, 404, empty list, etc.)
- Delete incorrectly registered webhooks
- Test connectivity and API responses

[Erik Andersson]: Because the generic connector includes endpoints to create list and delete web hooks. You can also of course utilize this same collection if you need to do some debugging for a customer. If they say why can't I see any mapping? Yeah, well when I try to make this request here to get the schema like I get an empty list or I get a permission denied or I get a 404 or whatever.

### Important Process: Route Questions Through SOC/Support

[Erik Andersson]: It is so important that these questions go via SOC and support, because now they know about this problem. We've described it to them. If there are more customers that experience the same thing, they already know what the problem is. If these questions get pinged to us directly from, say, account managers, we are circumventing that whole process and that knowledge sharing will not be there and then you will get the same question from SOC and support in the future as well.

**Why This Matters:**
- SOC/Support team now knows about the Tribe empty profile bug
- If other customers report the same issue, SOC can immediately identify it
- Knowledge is centralized, preventing duplicate debugging work
- Account managers bypassing this channel causes information loss

---

## Lead Creation Flow: Example Walkthrough

### Generic Connector vs. Virtual Lead Concepts

[Erik Andersson walks through a demo of lead creation in E-deal]

### E-Deal Example (Real Lead Entity)

**Setup:**
- Customer configures a Slipform with Apsis data
- Maps fields to E-Deal's form submission webhook

**Submission:**
1. Form submits to E-Deal
2. E-Deal creates a "Silhouette" (lead) record
3. E-Deal responds with:
   ```json
   {
     "new_records": [
       {
         "record_id": "[silhouette-id]",
         "entity": "Silhouette"
       }
     ]
   }
   ```
4. APSIS creates a profile with the Silhouette ID in a dedicated key space
5. Profile is searchable in Audience under that Silhouette ID

**Unique Identifier Field:**
E-Deal specifies which field contains the unique identifier for the Silhouette record. APSIS needs this to track profiles by Silhouette ID instead of Contact ID.

[Erik Andersson]: We will never download leads from the CRM system because that there has not been a use case yet where that has been interested. The CRMS are interested in getting this data from APSIS to the CRM so they can process their leads and decide, yeah, we have like this customer or this potential customer's interested, we'll convert it into a proper customer or no, we don't have enough information to go on like we'll just discard that.

### Silhouette Delete Use Case

[Erik Andersson]: If the CRM decides a lead isn't worth pursuing, they can send a delete request to APSIS:
- This is the **only update** APSIS expects from leads/silhouettes
- It uses the same webhook as profile updates (delete is treated as an operation)
- This is why APSIS registers webhooks for lead/silhouette entities

The other CRM systems (FSC Enterprise, Microsoft Dynamics) have similar structures with their own lead entities.

---

## Comparison Across CRM Implementations

### FSC Enterprise 12.1 (Generic Connector)

- Real Contact entity, real Lead entity
- Lead entity responds to form submissions
- Webhook registration is straightforward
- Supports delete operations on leads

### Microsoft Dynamics (Third-Party Plugin)

- Actual Lead entity in Dynamics
- Will respond to form submissions with Lead entity (once feature is fully implemented)
- Predictable, non-virtual entity model

### Tribe (Virtual Lead Entity — THE PROBLEM)

- Virtual Lead entity (represents dropdown selection)
- No guarantee which entity will actually be created
- Mappings only apply to the selected entity, not all possible entities
- Tribe can change the dropdown configuration and break everything
- Webhook registration is fragile

---

## Integration Marketplace Cleanup Action Items

[Erik Andersson]: The following work is needed to clean up the integration marketplace:

### Backend Changes (Integration Team)

1. Remove from the integrations list API:
   - Presta Shop
   - Adobe Commerce
   - Sitecore
   - Drupal
   - Optimizely
   - Tessitura
   - Ascose
   - Google Data Studio
   - Ambraco
   - Power BI

2. Ensure these are marked as non-installable in the backend list

### Frontend Changes (Chris - Frontend Team)

1. Remove integration cards from the carousel UI for all deleted/deprecated integrations
2. Update the integrations detail model if needed
3. Test that removed integrations no longer appear in the customer UI

**Note:** This is mostly front-end work. Erik will create a Jira story for Chris and connect it to the backend cleanup story.

[Erik Andersson]: I'll create a story here to clean everything up in integration that needs to be cleaned up. But I think the majority of the work is just in the front end here now to delete these integration cards because I know we did a spring cleaning of this for a lot of deprecated integrations in the back end.

> "This has been aching to false advertisement because it hasn't been like this for quite some time."

---

## Key Takeaways

1. **Plugin Ownership Shift:** APSIS moved from owning plugins to partnering with external companies. Customers now pay the plugin developer, not APSIS, and the developer handles support. This saves APSIS resources while aligning incentives.

2. **Never Build Custom Plugins:** Do not volunteer to build custom plugins for new CRM systems. The hidden cost of ongoing support and maintenance will exceed any revenue. Escalate high-value customer requests to Product Management and suggest external consulting partners like Silverlord Consulting.

3. **Legacy Connectors Are Dead:** Presta Shop, Adobe Commerce, Sitecore, Drupal, and Optimizely are open-source reference code with zero support, even for existing customers. These should not appear in the customer-facing marketplace and need UI/backend cleanup.

4. **Lime Plugin Was a Learning Experience:** The Lime CRM plugin attempted to replicate email/website event retrieval but failed due to database multi-tenancy issues and complexity. The repository remains as a reference for how to build Lime plugins if a Generic Connector is ever needed.

5. **Tribe's Virtual Lead Entity Is Fragile:** Tribe's dropdown-based entity selection creates a virtual "Lead" entity concept that doesn't match reality. This caused duplicate empty profiles during a real bug. The fix: don't register webhooks for the virtual Lead entity. The ideal fix: require Tribe to always use Contact entity and remove the virtual Lead concept.

6. **Route Support Questions Properly:** SOC/Support team must be the first line for customer issues. Direct escalations from account managers bypass knowledge-sharing channels and cause duplicate debugging work later. Document issues and solutions in the SOC/Support system.

7. **Third-Party Integrations:** Most modern integrations are either (a) One API direct usage where APSIS just generates keys, or (b) Generic Connector implementations. Neither requires plugin development or ongoing support from APSIS.

8. **Postman Collection for Debugging:** Use the Generic Connector Postman collection to debug customer environment issues, test API connectivity, and manually delete incorrect webhooks when necessary.

9. **Marketplace Cleanup Needed:** Several integrations (Presta Shop, Adobe Commerce, Tessitura, etc.) are still showing in the marketplace despite being unsupported. Both backend (remove from list API) and frontend (delete UI cards) work is needed.

10. **Disclosure of Hidden Costs:** When Product or account managers request new connector builds, be transparent about the ongoing cost of maintenance and support. Even large customer payments can become unprofitable from an R&D perspective.

---

## Unresolved Questions and Follow-Up Actions

1. **Marketplace Cleanup Story:** Erik will create a Jira story for cleanup. Needs both backend (Integration team) and frontend (Chris) work. Status: To be created.

2. **Existing Tribe Customers:** Unclear how many existing Tribe customers might be affected by the empty lead profile bug. The SOC/Support team should monitor for this pattern going forward.

3. **Potential Future Customizations:** If any existing customers are using the Drupal plugin, will they need support? Policy is no, but should verify no active customers depend on it.

4. **Shopify and E-Commerce:** Full context on why these projects were abandoned (licensing, budget) suggests these decisions were made at Product/Business level, not technical. Worth noting if similar requests resurface.

5. **Tribe Product Roadmap:** With Rose as a Product Owner for Tribe, there's an opportunity to standardize the Lead entity concept, but this would require Tribe product changes outside APSIS' control.