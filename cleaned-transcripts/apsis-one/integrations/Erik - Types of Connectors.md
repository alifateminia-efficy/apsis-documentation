---
source_file: Erik - Types of Connectors.txt
domain: Apsis One Integrations
topics: [Connector Architecture, Legacy Connectors, Generic Connector, Third-Party Integrations, Installer Options, Outbound Mappings, Integration Sustainability]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Shreevidhya Ganesan]
key_components: [Justin (Integration Platform), Microsoft Dynamics, FSC Enterprise 12.0, FSC Enterprise 12.1, Lime CRM, Tribe, E-deal, Web CRM, Intermail, Super Office, Siteshop, Audience, Field Mappings, Consent Mappings, Outbound Mappings]
session_type: knowledge-transfer
subdomains: [Architecture, Microsoft Dynamics Integration, Efficy Enterprise 12.0 Integration, Efficy Enterprise 12.1 Integration, Tribe Integration, E-deal Integration]
---

## Session Overview

This knowledge transfer session covers the three primary types of connectors in the Apsis One integration platform (codenamed **Justin**): legacy connectors, the generic connector, and third-party integrations. Erik Andersson explains the architectural evolution from custom-built connectors to a standardized generic approach, detailing why legacy connectors are no longer sustainable, how the generic connector shifts responsibility to CRM partners, and the role of third-party integrations. The session also covers installer options and outbound mappings that control how data flows between Apsis and external CRM systems.

---

## Connector Architecture Overview

### The Three Connector Types

Erik outlines three distinct approaches to CRM integration within the Justin platform:

1. **Legacy Connectors** – Custom-built by Apsis in-house
2. **Generic Connector** – Standardized interface requiring CRM partners to adapt
3. **Third-Party Integrations** – External companies building connectors with minimal Apsis involvement

[Erik Andersson]: "There are three important types of connectors that we will need to cover here."

---

## Legacy Connectors: The Original Approach

### Historical Context and Implementation

When the integration project began in late 2018, Apsis built connectors in-house by leveraging existing CRM APIs. The team would extract whatever data was needed (contact fields, consent states, etc.) and handle any necessary data transformation.

**Example**: Microsoft Dynamics uses `1` and `0` for boolean values, while Apsis One requires `true` or `false`. Each connector had to perform these transformations individually.

### Why Legacy Connectors Became Unsustainable

The fundamental problem with legacy connectors is **non-DRY development**: whenever new logic was added to Apsis (e.g., syncing email events from Apsis to the CRM), the team had to replicate that logic across every connector.

[Erik Andersson]: "When you get up to like even 4 integrations then you will notice that this is not a sustainable way of developing this because like just adding one new feature would take like months of work and then you have the maintenance of them all will be a absolute nightmare."

### Current Legacy Connector Portfolio

As of the session date, Apsis maintains legacy connectors for:

- **Microsoft Dynamics**: ~60 customers
- **FSC Enterprise 12.0**: ~60 customers (enterprise customers resistant to migration)
- **Lime CRM**: 1–2 customers (small base but paying well)

[Lukasz Grabowski]: "Approximately how many customers nowadays use those legacy connectors?"

[Erik Andersson]: "We have quite a few for Microsoft Dynamics. We also have quite a few for FSC Enterprise 12.0. I don't have the exact number, but I would but on like from the top of my head maybe around 60 or something like that... We have a lot of enterprise 12.0 customers because when we were procured that is the information that was heavily pushed for and also the enterprise customers are usually like very big. Corporate the customers who don't like change."

### Development Policy for Legacy Connectors

**No new legacy connectors are being built.** No new features are being added to existing legacy connectors unless a major paying customer requests it (which has not occurred).

[Erik Andersson]: "However, we have since quite long said like we are not developing any new legacy connectors, we are also not adding any new features... The exception to this would probably be like if some very big customer tells us like hello, we will pay you a boatload of money if you add this specific feature. This has not happened though."

---

## The Generic Connector: Standardized Integration

### Core Philosophy

Instead of Apsis adapting to each CRM's data structure and quirks, the **generic connector** shifts responsibility to the CRM partner. Apsis maintains one connector in the codebase and defines a standardized contract that all CRM partners must follow.

[Erik Andersson]: "The idea behind the generic connector. Is that instead of APSIS adapting to the CRM and us adapting to their data structure and having to do ugly hacks to get things working? The generic connector shifts the responsibility to the CRM instead."

### How It Works

- Apsis defines one set of endpoints (e.g., `/records/{pageNumber}`) with a specific data format.
- All CRM systems (Tribe, E-deal, Web CRM, Dynamics via v2) call these endpoints in the same way; only the hostname differs.
- CRM partners are responsible for implementing these endpoints and returning data in the exact format Apsis expects.

[Erik Andersson]: "We will always call say... slash records slash page number something something we will call that end point for whatever CRM system might be out there. The only thing that will differ is of course the host name so. We might we call today we call tribe the E deal and web CRM like we call all of them in exactly the same way we expect them to respond with. With the exact same data formats and behave in the same way."

### Key Advantages

**1. Single Maintenance Point**  
When a new feature is added (e.g., SMS event syncing), it's implemented once in the generic connector and automatically available to all CRM partners who support it.

**2. Easier Debugging**  
If a feature works for Tribe but fails for E-deal, it's immediately clear that the issue lies in E-deal's implementation, not in Apsis's generic connector code.

**3. Capability Declarations**  
Each CRM provides Apsis with a list of supported features (e.g., can sync email activities, SMS activities, event tool activities). Apsis only enables UI options for features the CRM declares support for.

[Erik Andersson]: "Each CRM system they also have a possibility to provide apps is with. These are the different features that we support... When you call integration and say like please list every integration that might be installed on this section. As part of this we are providing you with the information like can sync e-mail activity, can sync SMS activity, can sync. Event tool activities."

### Handling Feature Flags and Capability Announcements

CRM partners declare capabilities via feature flags. Apsis does not show an option or send data to a CRM until that CRM explicitly states it supports that feature.

[Michal Rosikiewicz]: "When you add new features to generate connector, do you have some kind of versioning or this is depending on this feature flags that CRM supports and if this is something new then you simply don't send it to this CRM."

[Erik Andersson]: "So we if we add something new in apsis that the CRM does not support, like we don't start sending that until they have enabled that feature flag in the CRM so... the event tool is a good example. So we are not displaying this option in Appsys until the CRM has stated that now we support event tool."

Versioning of the generic connector itself has not been necessary because Apsis has only **added** features, never removed or modified the core contract in a breaking way.

### Historical Timeline and FSC Enterprise Example

Apsis began the generic connector initiative approximately **two years ago** (around summer 2022), driven in part by the complexity of FSC Enterprise 12.0 integration.

[Shreevidhya Ganesan]: "Basically summer 2022 we started this project I guess like you know of having the generic connectors and since then we have been developing only generic connector, right Eric?"

### FSC Enterprise Data Quality Issues (War Story)

FSC Enterprise 12.0 exposed significant data inconsistency problems that justified the generic connector approach:

- **Default Date Handling**: If a date field is empty, FSC Enterprise sets it to December 18, 1899. Apsis treats this as a legitimate date, causing MA flows to trigger incorrectly (customers appear 100+ years old).
- **Type Mismatches**: FSC Enterprise claims to support integer and boolean fields but sends stringified versions (`"1"`, `"0"`, `"true"`) instead of native types.

[Erik Andersson]: "For just to take some examples for FC enterprise, because it was a very, very interesting CRM system to integrate with, for example, if you don't. If you have a date field in the in the CRM but you don't add any data to it, they set a default date of 1800 ninety-nine in December but. 18199 in December like that is a legit date as far as abscess is concerned... They also have string. They also have integer fields that they say like, yeah, we totally support integer fields. The problem is that they don't send us integers, they send us us stringified integers."

With the generic connector, **all responsibility for correct type handling rests with the CRM partner**. Apsis enforces the contract: if you declare a field as an integer, you must send an integer.

[Erik Andersson]: "No. We don't because in the generic connector they can't send us stringified data. The end point if they if they say that this field is a integer field, then the contract dictates that they must send us as an. Integer like they have to. We force them to send us data in the format. They say like if it is supposed to be a boolean, then it has to be a boolean."

### Configuration Requirements

Each CRM integration requires configuration of:

- **Connector Display Name** and **Integration ID** (unique identifier within Apsis)
- **Entity Names** (e.g., "contact," "person," "silhouette")
- **ID Field** (the unique identifier used to track records; must be different for every contact)
- **Support Toggles** (which features does this CRM support?)
- **Concurrency Settings** (page sizes, thread counts for bulk syncs)
- **Inbound/Outbound Flags** (should we pull data from the CRM? Should we send data to the CRM?)

---

## Migration from Legacy to Generic: FSC Enterprise Example

### FSC Enterprise 12.0 vs. 12.1

- **12.0**: Uses legacy connector (Apsis handles all data transformation)
- **12.1**: Supports generic connector (FSC Enterprise handles data transformation)

The dream state is to migrate all legacy customers to generic connectors, but this is complicated by:

1. **On-Premise Deployments**: Many FSC Enterprise customers run on-premise versions and are reluctant to upgrade.
2. **Backward Compatibility**: New 12.0 customers (though rare) still exist and must be supported.

[Erik Andersson]: "It's not that easy because unfortunately there might still be old versions of the enterprise sold for whatever reason and. You can't use the new connector on the old version... For example, say that you or you have existing enterprise, I don't know what's. Yeah, you have existing 12.0 customers that haven't been using Appsys, but they want to start using Appsys. Then we can't hide that connection option for them."

**Contrast with Web CRM**: Web CRM is a SaaS product, so all instances can be upgraded automatically. Apsis can hide the legacy Web CRM connector entirely; the old connector only appears if you already have it installed.

### Migration Recommendation

[Erik Andersson]: "The dream state is of course that every old, every old one is migrated and I would highly suggest that like this is pushed for in tandem with consultancy because it's consultancy that typically. Handles the actual migration because they sit with the customer and their data."

---

## Third-Party Integrations

### Overview

External companies build and maintain their own connectors to Apsis, without using the Justin codebase for most logic. They integrate via **Audience** (Apsis's event bus) rather than the generic connector.

[Erik Andersson]: "These third party connectors or integrations, they don't really utilize Justin for most of their logic. They go through one. Instead like retrieving some e-mail events or creating profiles or consent."

### Apsis's Minimal Role

When a customer installs a third-party integration, Apsis:

1. Creates a **key space** for that integration
2. Whitelists the **CRM ID attribute** for that key space
3. Generates a **one-time API key**

After this, Apsis has minimal involvement. If data syncs fail, the third-party company is responsible for debugging and support.

[Erik Andersson]: "The only thing which we provide support for here is that. We enable them to install using our platform and the only thing we do when you install a third party integration is that we create the key space for them. So like if you install Super Office which is a third party one. We create the Super Office key space. We whitelist the CRM I oops. We whitelist the CRM ID attribute for this key space and we also generate A1 API key, but we don't."

### Customer-Facing Flow

The customer receives the API key from Apsis's UI and pastes it into the third-party integration's configuration page. The third-party system then uses this key to authenticate API calls to Apsis.

[Erik Andersson]: "We create the key space using the. I think it's actually your endpoints. We whitelist it and then we... The customer copy paste this one API key and they go into the configuration page for this third party."

### Current Third-Party Integrations

- **Super Office**: ~5–6 customers
- **Intermail Loyalty**: 1 customer (company is discontinuing this product)
- **Join CX**: 2–3 customers
- **Lead Family**: A few customers
- **WooCommerce**: 3–4 customers

All of these combined are **under 10 customers**, making them a tiny portion of Apsis's integration base.

[Erik Andersson]: "I think super office there might be like 5 or 6 intermail loyalty only exist one now because like they are trying to discontinue this join XI think has two or three customers lead family... There are a couple, but it's still like it's beneath 10."

### Special Case: Intermail

Intermail is both a customer of Apsis (they use Apsis for their marketing solution) and a partner. They've built their own connector that **implements the generic connector specification**, giving them field mappings and consent mappings like native integrations.

However, Apsis does not control Intermail's connector. They maintain it independently and decide which generic connector features to support.

[Erik Andersson]: "The one exception here is that we do have a collaboration with a company called Intermail... Intermail has built a integration like they are maintaining it on their side and they have built support for the generic connector like they are using essentially every everything in it. But it's not a apsis connector like we we don't control what they support or if they have added any new any more functionality, let's use this one API that we don't know about."

### Typical Use Cases

Third-party integrations often serve **niche use cases** that don't fit Apsis's core CRM integrations:

[Erik Andersson]: "For example intermail loyalty that is not so much handling like profile data as it is. Handling like loyalty points from like, yeah, well-being a loyal loyal customers. So they add like this loyalty points as an attribute to profiles in Axis, but that and then they can do some. Outs inside of apps is using that attribute that they synced."

---

## Installer Options and Configuration

### Purpose

**Installer Options** (also called **Connector Options**) allow Apsis to configure how each CRM integration behaves, accounting for differences in entity naming, supported features, and data flow direction.

### Key Configuration Fields

#### Entity and ID Field Configuration

Every CRM has different naming conventions for the main entity representing customers:
- Microsoft Dynamics: `Contact`
- Tribe: `Person` (with field `PerID`)
- FSC Enterprise: `Contact`
- Others may use: `Silhouette`, `Account`, etc.

The **ID field** is critical: this is the unique identifier Apsis uses to match records between systems. It must be unique per contact and must be specified during setup.

[Erik Andersson]: "Each CRM always has like this entity represents the actual customers... we configure like what is the display name of this one... An important part is also the ID field name, because this is the unique identifier. As you know in apps is like you always need to specify like a key space and a. Key and whatever value is contained here in the ID field name. This is the unique identifier like the key that we utilize in apps and identify the. The records with and for in the future if they send us updates, we will update the profile on that specific key and not create a duplicate duplicate with it."

#### Inbound and Outbound Flags

- **Inbound**: Should Apsis pull contact data from the CRM? (Usually `true`; the CRM is the main data source)
- **Outbound**: Should Apsis send form submissions and other data to the CRM? (Controlled separately for each entity)

[Erik Andersson]: "We can also configure like should we download this data to APSIS or not and should we send things from APSIS to to the CRM system."

#### Consent Handling (Special Case)

**Consent is bidirectional** and always synced, regardless of the outbound flag. This is critical for GDPR compliance and preventing unsubscribe violations.

[Erik Andersson]: "Consents is a exception to this because consent must be the same in both the CRM and in APSIS... whenever you have an integration like the the rule of thumb is that the CRM is the like main source of the data, whatever, whatever is in the CRM. Should be what is in apsis, but this only holds true for the attributes because when it is, when as far as consent is."

**Why consent must be bidirectional**:

If a customer unsubscribes in Apsis (e.g., by clicking an email unsubscribe link), that change must be reflected in the CRM. Otherwise, on the next sync, the CRM's opt-in status would override the Apsis opt-out, and Apsis would resume sending emails to an unsubscribed user—a severe compliance violation.

[Erik Andersson]: "The end customers can opt out when you send an e-mail sending. So for example say that it is a company that that install apps is. They install Dynamics in Appsys, they import their customers, then they do an e-mail sending and one of the customers responded to an Appsys mail saying no, no, no, I am not interested in this or I never signed up for this. Remove me from this. We this is something like they do on an apps service and thus we need to make sure that this is reflected in the CRM system because what happens otherwise well. The profile in APSIS will be opted out if they press on unsubscribe, but the consent would still be opt in in the CRM system and the next time we do a sync to APSIS for whatever reason, then we would override their opt out with a opt in and now we would. Start sending things which they have explicitly said. You are not allowed to contact me on this and that would put this in a lot of trouble."

#### Feature Support Toggles

Apsis can disable specific features (like event tool syncing) globally for a CRM if Apsis knows that CRM doesn't support it. If enabled, Apsis will check the specific customer's CRM instance to see if it supports the feature.

[Erik Andersson]: "We have like a kind of deadbolt on these can sync activities, for example if we know that. FSC enterprise for example like no they have not developed support for the event tool for like in anywhere. They don't have any test instances or like there are no version available for it. Then we can disable this function inside of APSIS and like if we have done that we will not even go to the CRM and check does this unique customer instance supported like is it development instance or is it a? Uh, very old instance. But if it is true then we will we will go to the CRM and check like does this specific environment support event to like maybe this a. Development instance where the developers have added it to, but the customer's production system do not yet cover it."

#### Concurrency and Performance Settings

Apsis can configure:
- **Page sizes** for bulk data downloads
- **Thread counts** for parallel requests

[Erik Andersson]: "We have we can configure like a concurrency so when we. Download contacts from the CRM system, for example that which showed yesterday for a full sync like we can configure how many threads do you do we use when we download things like how big are the page sizes."

---

## Outbound Mappings and Form Submissions

### Why Outbound Mappings Are Needed

When a customer submits a form in Apsis, the CRM receives a submission event. However, without additional information, the CRM doesn't know which form field corresponds to which CRM entity field (e.g., which form input is the "first name" vs. "favorite color").

[Erik Andersson]: "Picture this that in the CRM you would preferably want to like create a lead that has a first name, last name and an e-mail address. But if we just send you the submit event. How do you know like what field in the form should I use as the first name if I create like a form which has a field, my super cool cool field and favorite color like? What in that form should be utilized? The first name like you have no idea unless it explicitly says like first name so."

### How Outbound Mappings Work

Outbound mappings are the **reverse** of field mappings. They specify:

> For each attribute in Apsis, which field in the CRM's lead/entity structure should receive this value?

[Erik Andersson]: "The outbound mappings here lets you map essentially the reverse of the field mapping. So we take what field in apsis should be mapped to which field in the CRM system... You can then say like this field in APSIS should be represent this field in the silhouette, this field in APSIS should be the last name etcetera when you submit the form and if you have done the field to attribute mapping then. We can take these attributes and map them to the corresponding one in the CRM system, because then we can give this the CRM the whole submit event, but we can also give them. A exact mapping. This is the value you should put in the first name. This is the value you should put in the last name, and this is the value you should put in the e-mail field."

### Data Flow Example

**Scenario**: A form in Apsis has three fields:
- "Eric's cool field" (maps to Apsis attribute `firstName`)
- "Sailing boats" (maps to Apsis attribute `lastName`)
- E-mail (maps to Apsis attribute `email`)

When a user submits this form:

1. Apsis creates a profile with attributes: `firstName`, `lastName`, `email`
2. If **outbound mappings are configured**, Apsis sends to the CRM:
   - `firstName` → CRM's `first_name` field
   - `lastName` → CRM's `last_name` field
   - `email` → CRM's `email` field

3. If **outbound mappings are NOT configured**, Apsis just sends the raw form data, and the CRM has to guess which field is which.

[Erik Andersson]: "And let's take that example now like I don't feel in the mobile number here, but I have the mobile number in the outbound mappings like that is absolutely fine. Then we will still send to the CRM like this is the first name to the lead and this is the last name to the. Lead and make it more fun or more realistic and not call the input field last time. So this is like Eric's cool field and this is now. Uh, sailing boats. So it's a more realistic example."

### When Outbound Mappings Are Used

Outbound data is sent in two scenarios:

1. **Form Submissions**: When a customer submits a form and that form is configured to sync to the CRM
2. **Marketing Automation**: When a profile enters a task node in MA (covered in a separate session)

[Erik Andersson]: "So outbound is only applicable like when you have submitted a form and we want to like send this data from APSIS to the CRM. Or or when a profile has entered a task node inside of MA."

### Outbound Mappings Are NOT Used for Attribute Sync

**Important caveat**: Outbound mappings do **not** apply to attribute changes. If you modify an attribute in Apsis, it is **never** synced back to the CRM (because the CRM is the authoritative source for customer data).

[Erik Andersson]: "But it is a bit unfortunate name because like I said before, like we never sync attribute changes to the CRM and this still holds true because if you change a attribute in apps, we don't send it to the CRM."

### Feature Availability

Not all CRM integrations support outbound mappings:

- **FSC Enterprise 12.0**: Does NOT support outbound mappings
- **FSC Enterprise 12.1**: SHOULD support outbound mappings (though a bug was discovered during this session where mappings aren't visible in the UI)
- **Dynamics, Tribe, E-deal, Web CRM**: Supported (via generic connector)

---

## Event Tool Syncing and Feature Flags

### Feature Flag Discovery Issue

During the session, Lukasz Grabowski noted that the Event Tool "Sync to CRM" option wasn't visible in his test environment, even though it should have been available.

[Lukasz Grabowski]: "I checked in the event tool and we don't have this checkbox. I can't remember how it was done, but it looks like we automatically enable sync if there is. Integration enabled on the section."

### Root Cause

The visibility of the Event Tool sync option is controlled by **two mechanisms**:

1. **Account-Level Feature Flag** (`sync_event_tool` or similar) – Legacy approach, problematic for customers with mixed environments
2. **Integration-Level Capability Declaration** – The recommended approach

The account-level feature flag is problematic because:

[Erik Andersson]: "The customer might have one old instance like the their production environment. Where the event tool syncing is not supported and then of course it shouldn't be visible, but it might also be that they have a section with a develop instance where they want to test out the event tool syncing functionality. So on the account it might differ. You can't say like every section, every installation on this account should have this available."

### Recommended Solution

**Remove the feature flag entirely** and rely only on the capability declaration from the generic connector. When Apsis calls the integration's `/config` endpoint, the CRM responds with `can_sync_event_tool_activities: true/false`. This response should be the only control for UI visibility.

[Erik Andersson]: "You have to have this integration enabled to Maxa or not... Because I I would in all honesty recommend you to remove this feature flag and instead rely on the data that we send to you... When if we check. Here in the network tab for example if I create an event now and I pick... We respond here with like can sync e-mail activities true and if this is true then that option should be visible... The problem with having that feature flag on the account level is that."

[Lukasz Grabowski]: "I will create story to remove it entirely and rely only on this endpoint."

### Data Structure

The integration endpoint response includes capability flags like:

```
{
  "can_sync_email_activities": true,
  "can_sync_sms_activities": true,
  "can_sync_event_tool_activities": true,
  ...
}
```

Apsis should only display/enable sync options for capabilities declared as `true`.

---

## Connector Strategy: Why New Connectors Should Be Generic

### The Directive

[Erik Andersson]: "Every new connector should absolutely be a generic connector implementation. You you should, in all honesty, refuse to do it otherwise."

When external companies or customers ask for a new integration, the R&D team should:

1. **Refuse** to build a legacy connector
2. **Direct** the partner to implement the generic connector specification
3. **Support** the partner's generic connector development

### Why This Matters

Building a legacy connector means indefinite maintenance burden, as shown by the FSC Enterprise 12.0 example. The generic connector approach scales because:

- New features are added once, benefiting all CRM partners simultaneously
- Maintenance responsibility is shared (or delegated entirely to partners)
- Debugging is straightforward (isolate by CRM, not by feature)

---

## Go-to-Market Strategy for New Integrations

### The Product/Sales Workflow

When potential CRM partners are identified (often by the product team), the sales process involves:

1. **Product identifies partner** (e.g., Salesforce)
2. **Sales/business dev contacts partner** with a collaboration proposal
3. **Negotiate terms**: Who owns the customer? Who maintains the connector?

### Preferred Model: Partner-Owned with Revenue Share

**Best Practice**: CRM partners own the customer relationship and handle all support. Apsis provides the connector framework and assistance with setup. Revenue is shared.

**Example - Siteshop**:
- Siteshop owns the customer (Siteshop charges the customer for integration)
- Siteshop maintains the Siteshop connector and provides support
- Apsis does not need to troubleshoot customer issues or open support cases
- Revenue sharing: Apsis makes money from email/profile usage; Siteshop makes money from integration fees

[Erik Andersson]: "The way it works with Siteshop is that it is not apsis that owns the customer, so to say. It is Siteshop that owns the customer, so Siteshop. Um charges the customer for the use of the integration and apps is oh sorry. Appsys benefits from this because Siteshop is losing customers to Appsys which they they then pay for all the profiles that they that they import and all the emails that they send."

### Anti-Pattern: Apsis-Owned with In-House Support

**Bad Practice**: Apsis owns the customer, maintains the connector, and provides support. This leads to cost overruns.

[Erik Andersson]: "For the old Dynamics connector here, because APSIS paid CRM Consult now to develop this, APSIS is charging customers for the use of the connector... So as it is with the old Dynamics connector here, because APSIS paid CRM Consult now to develop this, APSIS is charging customers for the use of the connector... Here for site shop, APSIS is not charging the customer for the use of the connector. It is site shop that does it and the beauty of it is that site shop owns the maintenance and the supports. For the connector on their side."

### Cost of In-House Maintenance

Support costs for legacy connectors are substantial. Bug fixes on the old Microsoft Dynamics connector cost **2500 SEK/hour** (approximately $230/hour USD), which quickly eats into any revenue from the integration.

[Erik Andersson]: "For example, like I think it costs 2500 crown per hour. To get the old Microsoft Dynamics plugin fixed and like, that's not the cost that is sustainable."

### Clear Guidance for R&D

[Erik Andersson]: "A lot of this that we've talked about now like this should not fall on R&D like this is a product question, but as as long as you should just be aware like that you should not take it upon yourself to start building. Building actual integrations because then you will be stuck in a maintenance hell mode. So really try to be firm with that like no find someone that wants to build an implementation for the generic connector and support it and then you would you would be more than."

The R&D team should:
- **Accept** requests to configure the generic connector for a new CRM partner
- **Refuse** to build and own a new legacy connector
- **Assist** with technical implementation (logging, testing, etc.) but leave customer ownership and support to the partner

---

## Key Takeaways

1. **Three connector types exist**: Legacy (dying), Generic (preferred), Third-Party (niche use cases).

2. **Legacy connectors are unsustainable** due to the need to replicate features across multiple codebases. Development and maintenance costs become prohibitive.

3. **The generic connector is the future**. It standardizes the interface, puts responsibility on CRM partners to comply with the contract, and enables features to scale across all CRM integrations simultaneously.

4. **Data type contracts must be enforced**. With generic connectors, if a CRM declares a field as an integer, it must send an integer—no stringification hacks like FSC Enterprise 12.0 used.

5. **Consent is bidirectional**. Even though the CRM is the authoritative source for contact attributes, consent changes in Apsis must always be synced back to the CRM to prevent unsubscribe violations.

6. **Outbound mappings** enhance form submissions by helping the CRM understand which Apsis attribute corresponds to which CRM entity field.

7. **Feature availability should be declared by CRMs**, not controlled by feature flags in Apsis. The integration's capability response should be the single source of truth for what features a specific CRM instance supports.

8. **Go-to-market strategy matters**. New integrations should be owned by CRM partners (not Apsis), with revenue sharing. This prevents Apsis from being stuck in maintenance hell.

9. **Do not build new legacy connectors under any circumstances**. If someone asks for a new integration, the answer is: "Implement the generic connector specification, and we'll help you build and test it."

10. **Migration from legacy to generic is an ongoing effort**, especially for on-premise CRM customers resistant to upgrades. Consultancy teams should be involved in these migrations.

---

## Unresolved Questions and Action Items

### Potential Bug Identified

[Lukasz Grabowski]: "Why I cannot see this outbound mapping in FEC enterprise?"

[Erik Andersson]: "Yeah. Because FCC enterprise 12.0 does not support outbound mappings... But. It does. We saw a bug now that it is not visible in 12.1, but it should be visible in 12.1. That is also something we need to check on when I'm done with all the handover things, but at least in theory it is. Supported because the outbound mapping is one of the big reasons why they wanted to have the generic connector support for it."

**Action**: Erik will investigate why outbound mappings are not visible in the FSC Enterprise 12.1 UI.

### Feature Flag Cleanup

[Lukasz Grabowski]: "I will create story to remove it entirely and rely only on this endpoint."

**Action**: Create a story to remove the account-level `sync_event_tool` feature flag and rely entirely on the integration's declared capabilities.

### Next Session Topic

The session ended before covering the actual code repository structure and folder organization within the Justin codebase. Erik indicated this would be a logical next session topic.

[Erik Andersson]: "The next part would be to look closer in the actual code repository structure. And like where where are the important folders? And like we are in the repository, would you look for the different things? And that might be a logical place to break now and take in a completely different session."

---

## Session Duration and Continuation

- **Total time covered**: ~1 hour 33 minutes (with a 5-minute break)
- **Actual content time**: ~1 hour 28 minutes
- **Next session**: To be scheduled, covering code repository structure and detailed implementation examples