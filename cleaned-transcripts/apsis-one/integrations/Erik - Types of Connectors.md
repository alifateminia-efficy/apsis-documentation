---
source_file: Erik - Types of Connectors.txt
domain: Apsis One Integrations
topics: [Types of Connectors, Legacy Connectors, Generic Connector, Third-Party Integrations, Connector Configuration, Outbound Mappings, Integration Partnerships]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Shreevidhya Ganesan]
key_components: [Justin (integration platform), Legacy Connectors, Generic Connector, Third-Party Integrations, Installer Options, Field Mappings, Subscription Mappings, Outbound Mappings, Event Tool, Form Tool]
session_type: knowledge-transfer
subdomains: ["Different Types of Connectors"]
---

## Session Overview

This knowledge transfer session covers the three distinct types of connectors in the Apsis One Integrations domain: **legacy connectors** (built in-house in 2018), **generic connectors** (standardized interface requiring CRM developers to implement), and **third-party integrations** (built and maintained by external companies). Erik Andersson explains the architectural evolution from unsustainable one-off connectors to the scalable generic connector approach, the business models for partnership integration, and the configuration mechanisms (installer options, field mappings, outbound mappings) required to support different CRM systems. The session establishes that new development should exclusively use the generic connector pattern and clarifies when each connector type is appropriate.

---

## Historical Context: Why Three Types of Connectors Exist

### The Problem with Legacy Connectors

When the integration project started in late 2018, Apsis built connectors in-house by directly adapting to each CRM's existing APIs. [Erik Andersson]:

> When we started this project back in like 2018, I was a consultant here in the late 2018 for this. We started building the connectors in house. So for example we wanted to connect to Microsoft Dynamics and the way we did this is that we utilized the existing APIs inside of Microsoft to extract whatever data we needed.

The approach required significant data transformation work:

> Abscess one needs to have a true or false for boolean, but Microsoft Dynamics likes having one and zero for their boolean, which we cannot put into abscess. So like we we had to adapt to their APIs, transform the data, handle any potential edge cases.

This became unsustainable as features were added. Each new feature (like email event syncing) required replication across every connector:

> When you get up to like even 4 integrations then you will notice that this is not a sustainable way of developing this because like just adding one new feature would take like months of work and then you have the maintenance of them all will be a absolute nightmare.

### Current Legacy Connector Customer Base

[Lukasz Grabowski] asked about the number of customers using legacy connectors. [Erik Andersson] provided estimates:

- **Microsoft Dynamics**: ~60 customers (uncertain of exact count, but significant)
- **FSC Enterprise 12.0**: Many enterprise customers who resist upgrading on-premise versions. "The enterprise customers are usually like very big. Corporate the customers who don't like change. So if they have a like on premise version, they aren't very hesitant to upgrade or migrate. That has been a recurring headache for us."
- **Lime CRM**: 1-2 customers, not a large base but paying well enough to maintain

**Policy**: No new legacy connectors are being developed, and no new features are being added to existing ones—with the rare exception of very high-paying customers willing to fund development.

---

## The Generic Connector: A Paradigm Shift

### Core Philosophy

The generic connector inverts responsibility: instead of Apsis adapting to each CRM, CRM developers implement a standardized interface. [Erik Andersson]:

> The idea behind the generic connector is that instead of APSIS adapting to the CRM and us adapting to their data structure and having to do ugly hacks to get things working, the generic connector shifts the responsibility to the CRM instead.

### Single Implementation, Multiple CRMs

There is literally one generic connector in Apsis. When new features are added (like SMS event syncing), they are added once:

> We have like 1 connector in APSIS which is literally called generic connector. Where we Add all of the support for. So say now that we want to enable syncing the SMS events from Appsys to the CRM, then we add the support for this in the general connector like we Add all of the functions we add the endpoints to it and then it is up to the CRM, to the CRM developers to add the expected endpoints for them.

All CRMs are called in the same way:

> We will always call say slash records slash page number something something we will call that end point for whatever CRM system might be out there. The only thing that will differ is of course the host name so we might we call today we call tribe the E deal and web CRM like we call all of them in exactly the same way we expect them to respond with the exact same data formats and behave in the same way.

### Benefits for Testing and Debugging

> From a debugging perspective it get it gets very easy to pinpoint the blame, so to say, because if you can see that the generic connector flow works for say tribe, but when it comes to E deal it is behaving in a weird way, then it is easy to see, yeah, but then the problem exists in ED because it is working as it should in tribe or web CRM.

### Data Type Enforcement

A critical difference from legacy connectors: CRMs must send data in the correct format. The generic connector enforces this contract. [Michal Rosikiewicz] asked whether customization is needed per CRM:

> No. We don't because in the generic connector they can't send us stringified data. The end point if they if they say that this field is a integer field, then the contract dictates that they must send us as an integer like they have to. We force them to send us data in the format they say like if it is supposed to be a boolean, then it has to be a boolean. If it is an integer, it has to be an integer. If it is a float, it has to be a float.

### FSC Enterprise: A Case Study in Why Generic Connectors Matter

FSC Enterprise 12.0 demonstrated why standardization was necessary. [Erik Andersson and Shreevidhya Ganesan]:

> For FC enterprise, because it was a very, very interesting CRM system to integrate with, for example, if you don't have a date field in the in the CRM but you don't add any data to it, they set a default date of 1800 ninety-nine in December but 18199 in December like that is a legit date as far as abscess is concerned. So suddenly suddenly you started having like MA flows that could trigger and all customers were now suddenly 100 years old because FC enterprise provided like their default data.

Additional data inconsistencies:

> They also have string. They also have integer fields that they say like, yeah, we totally support integer fields. The problem is that they don't send us integers, they send us us stringified integers. They also don't send booleans in their boolean field. They send us one and 0, but they also don't send us one and 0 stringified.

[Shreevidhya Ganesan] noted the timeline:

> Basically summer 2022 we started this project I guess like you know of having the generic connectors and since then we have been developing only generic connector.

### Feature Flags and CRM Support Capabilities

When a new feature is added to Apsis, CRMs declare support via feature flags. Apsis only exposes the feature if the CRM reports support. [Michal Rosikiewicz] asked about versioning:

> So we if we add something new in apsis that the CRM does not support, like we don't start sending that until they have enabled that feature flag in the CRM so. Let's say now for example the event tool is a good example. So we are not displaying this option in Appsys until the CRM has stated that now we support event tool.

The CRM provides metadata about supported capabilities:

> When you call integration and say like please list every integration that might be installed on this section. As part of this we are providing you with the information like can sync e-mail activity, can sync SMS activity, can sync event tool activities. This information we retrieve from the CRM system like they tell us hello, I can sync e-mail, I can sync Event tool, I can sync this, I support this, yada yada yada and then this is propagated up to the service in APSIS so we can enable or disable disable that.

---

## Legacy Connectors Still in Use

### FSC Enterprise 12.0 vs. 12.1

FSC Enterprise has both a legacy (12.0) and generic connector (12.1) implementation. [Erik Andersson]:

> For example, enterprise, I actually forgot to add the new enterprise version here. So you have enterprise 12 dot O as it's called. That is the old connector that we built in like that is an older version of enterprise that doesn't use generic connector. So there the connector code does all of this black magic transformation but they have also built a new version of enterprise which does support general connector which is 12.1 and if the customer installs that one then we we don't do any of that transformation then enterprise handles all of that.

### Visibility and Migration Strategy

The legacy 12.0 connector cannot be hidden because existing on-premise customers must maintain access to their configurations:

> It's not that easy because unfortunately there might still be old versions of the enterprise sold for whatever reason and you can't use the new connector on the old version. So for example, say that you or you have existing enterprise, I don't know what's. Yeah, you have existing 12.0 customers that haven't been using Appsys, but they want to start using Appsys. Then we can't hide that connection option for them.

### Web CRM Success Story

Web CRM successfully completed migration because it's a SaaS service with no on-premise deployments:

> Web CRM is a SAS service like you don't have any on premise instances or things. So every customer could be migrated to the new version and in that case we could hide the the old integration. Because every new customer could just start using the new connector by default. The only time we showed the old one was if you already had it installed, because then of course you had to be able to access your configuration.

---

## Third-Party Integrations

### Definition and Scope

Third-party integrations are built and maintained by external companies. Apsis provides minimal support: keyspace creation, CRM ID whitelisting, and API key generation. [Erik Andersson]:

> These third party connectors or integrations, they don't really utilize Justin for most of their logic. They go through one. Instead like retrieving some e-mail events or creating profiles or consent. So they don't utilize essentially any of our logic like Appsys does not control this integration. We don't really fully know how they work, nor do we really care about it.

### What Apsis Provides

> The only thing which we provide support for here is that we enable them to install using our platform and the only thing we do when you install a third party integration is that we create the key space for them. So like if you install Super Office which is a third party one, we create the Super Office key space. We whitelist the CRM I oops. We whitelist the CRM ID attribute for this key space and we also generate A1 API key, but we don't. Like if they're if the sinks are not happening as they should, like we don't support any of that because we don't know how they how they work.

### Customer Count and Examples

[Lukasz Grabowski] asked about customer usage. [Erik Andersson] provided estimates:

- **Super Office**: 5-6 customers
- **Intermail Loyalty**: 1 customer (discontinuing Join CX)
- **Join CX**: 2-3 customers
- **Lead Family**: A couple of customers
- **WooCommerce**: 3-4 customers

> It is quite common actually that some of the bigger CRM CRM users. They also have like a third party integration on the side. For example intermail loyalty that is not so much handling like profile data as it is handling like loyalty points from like, yeah, well-being a loyal loyal customers. So they add like this loyalty points as an attribute to profiles in Axis, but that and then they can do some outs inside of apps is using that attribute that they synced.

### Intermail CX: Hybrid Approach

Intermail is a special case: they are a customer of Apsis using the platform for marketing, and they have also built a generic connector integration. [Erik Andersson]:

> Intermail are using Appsys for their marketing solutions, so. Intermail has built a integration like they are maintaining it on their side and they have built support for the generic connector like they are using essentially every everything in it. But it's not a apsis connector like we we don't control what they support or if they have added any new any more functionality.

For Intermail, the flow is different than other third-party integrations. They provide an API key that Apsis uses:

> So the intermail loyalty is a that this is like a pure third party connectors like they get the one API key and they there there you copy paste it in their UI. In their new join CX connector here they have added like a full generic connector support.

---

## Partnership Model and Business Considerations

### Three Approaches to New Integrations

When a new CRM integration is requested (e.g., Salesforce), three approaches exist:

#### 1. Apsis Pays for Development (Not Recommended)
Apsis funds a development partner to build the connector, and Apsis owns and maintains it. Apsis charges customers for use. This creates high maintenance costs and support burden.

#### 2. Shared Ownership Model (Legacy Example: Microsoft Dynamics)
Apsis paid CRM Consult to develop the connector; Apsis maintains it and charges customers. [Erik Andersson]:

> As it is with the old Dynamics connector here, because APSIS paid CRM Consult now to develop this, APSIS is charging customers for the use of the connector.

#### 3. Partner Owns Maintenance (Recommended: Site Shop Model)
The external partner owns the customer relationship, maintenance, and support. Apsis receives recurring revenue from customer usage without maintenance burden. [Erik Andersson]:

> The way it works with Siteshop is that it is not apsis that owns the customer, so to say. It is Siteshop that owns the customer, so Siteshop charges the customer for the use of the integration and apps is oh sorry. Appsys benefits from this because Siteshop is losing customers to Appsys which they they then pay for all the profiles that they import and all the emails that they send.

**Key advantage**:

> Site Shop owns the maintenance and the supports for the connector on their side. So if the customers start having issues with the data sync to Apsis, then it is first and foremost Site Shop that handles the customer support cases. Of course they might reach out to appsys if they need to have some help with logs on our side, but everything is not funneled to appsys.

### Cost Warning for Apsis-Owned Connectors

Maintenance costs can quickly exceed revenue:

> For example, like I think it costs 2500 crown per hour to get the old Microsoft Dynamics plugin fixed and like, that's not the cost that is sustainable. A lot of this that we've talked about now like this should not fall on R&D like this is a product question, but as as long as you should just be aware like that you should not take it upon yourself to start building.

### Recommendation for R&D

[Erik Andersson's] clear direction:

> You should absolutely refuse to do it otherwise. Really try to be firm with that like no find someone that wants to build an implementation for the generic connector and support it and then you would you would be more than happy to to like add the configuration in apps is required and of course assist with the development of this connector like that we can never get away from, but that is a completely different other thing.

---

## Installer Options and Configuration

### Purpose of Installer Options

Installer options define how each CRM integration behaves in Apsis. Because each CRM has different entity names, data types, and capabilities, configuration is required. [Erik Andersson]:

> As we've touched upon, like every CRM system differs a bit in what they can do and of course what's what are the entities in the in the CRM call like Microsoft Dynamics call their and this contacts. So like we've seen contacts and contact ID in tribe it is persons and it is like a per ID. All of this is handled in the installer options.

### Key Configuration Fields

**Display Name and Integration ID**:
> In each of the connectors we configure like what is the name of the connector? Of course, what is the logical ID of the connector? Like this is very very important to differ from like you have the display name and you have the what we call integration ID.

**Feature Support Toggles**:
For CRM systems that may not have universal support (e.g., FSC Enterprise may not support Event Tool):

> We have like a kind of deadbolt on these can sync activities, for example if we know that FSC enterprise for example like no they have not developed support for the event tool for like in anywhere.

If disabled, Apsis doesn't check if the specific customer instance supports the feature. If enabled, Apsis queries each instance:

> If it is true then we will we will go to the CRM and check like does this specific environment support event to like maybe this a development instance where the developers have added it to, but the customer's production system do not yet cover it.

**Concurrency Configuration**:
> We can configure like a concurrency so when we download contacts from the CRM system, for example that which showed yesterday for a full sync like we can configure how many threads do you do we use when we download things like how big are the page sizes.

**Profile Entity Definition**:
> What we call like the profile entity, like the main entity in the CRM that we will download. Like each CRM always has like this entity represents the actual customers. It is what they want to download to Appsys and utilize in their e-mail send outs.

**Inbound and Outbound Flags**:
> We can also configure like should we download this data to APSIS or not and should we send things from APSIS to to the CRM system. So for example, if you run a full sync and inbound is enabled, then we will request make a request to the CRM system.

**Unique Identifier Field**:
This is critical for all integrations:

> An important part is also the ID field name, because this is the unique identifier. As you know in apps is like you always need to specify like a key space and a key and whatever value is contained here in the ID field name. This is the unique identifier like the key that we utilize in apps and identify the records with.

---

## Consent and Data Sync Direction

### CRM as Primary Data Source (with Consent Exception)

The general rule is that the CRM is the authoritative source for attributes. However, consent is **bidirectional**. [Erik Andersson]:

> Usually like whenever you have an integration like the the rule of thumb is that the CRM is the like main source of the data, whatever, whatever is in the CRM should be what is in apsis, but this only holds true for the attributes because when it is, when as far as consent is.

### Why Bidirectional Consent is Necessary

If a customer unsubscribes via an Apsis email, that change must sync back to the CRM. Otherwise, the next inbound sync would re-subscribe them:

> For the concerns, the end customers can opt out when you send an e-mail sending. So for example say that it is a company that that install apps is. They install Dynamics in Appsys, they import their customers, then they do an e-mail sending and one of the customers responded to an Appsys mail saying no, no, no, I am not interested in this or I never signed up for this. Remove me from this. We this is something like they do on an apps service and thus we need to make sure that this is reflected in the CRM system because what happens otherwise well. The profile in APSIS will be opted out if they press on unsubscribe, but the consent would still be opted in in the CRM system and the next time we do a sync to APSIS for whatever reason, then we would override their opt out with a opt in and now we would start sending things which they have explicitly said. You are not allowed to contact me on this.

**Attribute changes are one-directional** (CRM → Apsis only):

> We never sync attribute changes in apps to the CRM for this reason also because if you want to change attributes on the profile then you should change it in the CRM system because that is main source of the data.

---

## Outbound Mappings

### Definition and Purpose

Outbound mappings enable Apsis to send enriched data to the CRM when a form is submitted. They map Apsis attributes to CRM lead/silhouette fields. [Erik Andersson]:

> What outbound actually means here is that like normally we just sync this submit events like whatever they entered in the form that is what we sent, but we have a option to set up what we called outbound mappings.

### The Problem They Solve

Without outbound mappings, the CRM receives raw form submission data but has no context about field meanings:

> Say that you create a form which has a field, my super cool cool field and favorite color like? What in that form should be utilized? The first name like you have no idea unless it explicitly says like first name so the outbound mappings here lets you map essentially the reverse of the field mapping.

With outbound mappings, Apsis enhances the request with attribute data:

> We take what field in apsis should be mapped to which field in the CRM system. So say that you create now create a form and you create a custom field. You map that custom field to a attribute in abscess as you can do. What we will try to do if you have set up the outbound mappings is that we will take the attributes on the profile that has the submit event. And then uh create like a a Jason object with um the data in this apps is field put into the field that the CRM would recognize.

### Data Enrichment Example

[Erik Andersson] provides a concrete example:

> Let's say that your silhouettes or the lead structure in your CRM has a first name, last name and a mobile number as here. You can then say like this field in APSIS should be represent this field in the silhouette, this field in APSIS should be the last name etcetera when you submit the form and if you have done the field to attribute mapping then we can take these attributes and map them to the corresponding one in the CRM system.

**Result with outbound mappings enabled**:

> We can give this the CRM the whole submit event, but we can also give them a exact mapping. This is the value you should put in the first name. This is the value you should put in the last name, and this is the value you should put in the e-mail field.

### Outbound Enable Flag Behavior

The `outbound_enabled` flag on an integration controls whether this enrichment happens. [Michal Rosikiewicz] asked about visibility:

[Erik Andersson] clarified:

> Not really. The the difference it will make is that if you submit, if outbound is disabled for an entity, then we will just send the submit event and you will have to guess essentially what fields you should put in like the first name, last name or e-mail address. If outbound is enabled, then we will extract all of the data from the attributes from the resulting attribute in appsys and map it according to your well to these mappings and we are sending that as an additional property in to the CRM.

### When Outbound Data Is Sent

Outbound mappings are only relevant for **form submissions**, not for attribute changes:

> So outbound is only applicable like when you have submitted a form and we want to like send this data from APSIS to the CRM or or when a profile has entered a task node inside of MA.

> The only time we send these attributes in abscess to the CRM is when you have submitted a form and we send like the profile data that has the submit event.

### Missing Attributes in Form Submissions

If a form field doesn't collect data for an attribute with an outbound mapping, that attribute simply won't be sent:

> So like if if the attribute doesn't exist, of course we can't send it.

However, other mapped attributes are still sent even if some are missing:

> Yeah, like if if if the attribute doesn't exist, of course we can't send it. But that's as far as far as argument. Yeah, like if I take this e-mail here like I have mapped it to e-mail the first time is to first name, last name to last name. So this this would result I mean a profile with all of these attribute field. And let's take that example now like I don't feel in the mobile number here, but I have the mobile number in the outbound mappings like that is absolutely fine. Then we will still send to the CRM like this is the first name to the lead and this is the last name.

### Realistic Example with Custom Fields

[Erik Andersson] provides a more realistic example:

> Let's make it more fun or more realistic and not call the input field last time. So this is like Eric's cool field and this is now sailing boats. So it's a more realistic example. Like if if you did not have the outbound mappings here, like the CRM would have absolutely no idea that the sailing boat is actually the first name or the cool field is actually the last name. It will however be resolved with our outbound mappings because the customer on the integration page has stated that this APSIS attributes is the last name or the first name.

---

## Subscription Mappings and Consent Outbound

### How Consent Sync Is Triggered

Subscription mappings regulate when consent changes are sent to the CRM. Unlike forms (which are manually selected), consent changes are automatic when a subscription mapping exists. [Erik Andersson]:

> So the the consent if you have say like this now like the the consent will always be sent like if you have a mapping like as soon as you have set up a mapping from like to this default subscription e-mail, we will then register a listener that listens to send changes and for each subscription in Appsys that you have set up a mapping for like we will send changes in apsis for this subscription to the CRM like this happens disregardless of the of the the form.

### Difference from Form Sync

[Michal Rosikiewicz] confirmed understanding:

> So if they enable, for example, try, if they enable events can manage or or whatever the flag they use, you will start automatically sending those events to try only, not to ED or whatever. CRM but but only to specific instance.

[Erik Andersson] confirmed:

> Yes. So we we would start sending it when someone has like gone in and created an event and selected please send to or please sync to the CRM.

---

## Feature Flag: Sync Event Tool Activities

### Current Implementation

Event Tool syncing has both a **feature flag** (account level) and a **capability flag** (per integration). [Michal Rosikiewicz] asked about the mismatch:

[Erik Andersson] explained the issue:

> When if we check here in the network tab for example if I create an event now and I pick in this section here, so you make a request to us. Uh, for this section and it was the corporate one, so for on on this section there exists some installation. Anyway in in the integration we respond here with like can sync e-mail activities true and if this is true then that option should be visible.

### Problem with Account-Level Feature Flags

The feature flag approach is problematic because a single account may have both production (unsupported) and development (supported) instances:

> The problem with having that feature flag on the account level is that the customer might have one old instance like the their production environment where the event tool syncing is not supported and then of course it shouldn't be visible, but it might also be that they have a section with a develop instance where they want to test out the event tool syncing functionality. So on the account it might differ. You can't say like every section, every installation on this account should have this available.

### Recommendation: Remove Feature Flag

[Erik Andersson] recommends removing the feature flag entirely and relying solely on the integration endpoint response:

> You have can sync event tool activities. If this is true, then that option should be visible. I would in all honesty recommend you to remove this feature flag and instead rely on the data that we send to you.

[Lukasz Grabowski] confirmed this is leftover development code:

> I think the feature flag is not needed because it was done for development time just to be, you know, double secure. But I think we, yeah, we just missed it somehow, but because this is released to customers. I mean, this is not in development phase, right? We finished it some time ago, yeah. So I think it's a leftover.

---

## Known Issues and Notes

### FSC Enterprise 12.1 Outbound Mappings Bug

[Erik Andersson] noted a regression:

> There is also something we need to check on when I'm done with all the handover things, but at least in theory it is supported because the outbound mapping is one of the big reasons why they wanted to have the generic connector support for it.

The outbound mappings UI should be visible in FSC Enterprise 12.1 but is not currently.

---

## Key Takeaways

1. **Three connector types exist for different reasons**:
   - **Legacy**: Built in-house pre-2022, unsustainable to maintain, only supporting existing customers
   - **Generic**: Standardized interface (launched summer 2022), all new development must use this pattern
   - **Third-party**: External company integration with minimal Apsis involvement

2. **Generic connector is the future**: All new integrations must implement the generic connector pattern. If a prospect wants a new CRM integration, the answer is "they must implement the generic connector endpoints."

3. **Responsibility shifts to CRM developers**: With the generic connector, CRM systems are responsible for data format correctness, consistency, and implementing required endpoints. Apsis maintains one unified implementation for all CRMs.

4. **Data sync is mostly unidirectional (CRM → Apsis)**:
   - Attributes flow from CRM to Apsis
   - Consent is bidirectional (necessary to prevent unsubscribe being overridden)
   - Outbound mappings enrich form submissions sent to the CRM

5. **Feature support is capability-driven**: CRMs report what features they support via feature flags. Apsis only enables UI options (e.g., Event Tool sync) when the CRM declares support.

6. **Partnership model matters**: If building a new integration, insist the partner owns maintenance and customer support. Do not let Apsis own the connector—it becomes a cost sink (€2500/hour for support calls).

7. **Justin is the platform name**: When discussing the integration domain, Justin is the platform name. Connectors are installed and managed within Justin.

8. **Installer options configure each CRM**: Entity names, ID fields, concurrency, inbound/outbound flags, feature toggles, and other CRM-specific settings are defined here.

---

## Unresolved Questions and Action Items

1. **Exact customer counts for legacy connectors**: Erik estimates were approximate; exact figures should be verified.

2. **FSC Enterprise 12.1 outbound mappings visibility bug**: Currently not showing in UI despite being supported in theory; needs investigation.

3. **Event Tool feature flag removal**: [Lukasz Grabowski] planned to create a story to remove the feature flag and rely solely on integration endpoint capability response.

4. **Code repository walkthrough**: Next session should cover the repository structure, key folders, and where to find different components.

5. **Detailed outbound flow with logs**: Erik planned to show actual request/response logs to demonstrate how outbound mappings change the CRM-bound payload.

---

## Session Notes

- Total session length: 1 hour 33 minutes before break; resumed after break for additional 30+ minutes
- The meeting covered installer options, connector configuration, and outbound behavior
- Next logical break point identified: before diving into code repository structure
- Participants requested multiple sessions due to complexity and breadth of material