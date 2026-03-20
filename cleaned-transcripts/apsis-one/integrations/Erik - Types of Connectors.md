---
source_file: Erik - Types of Connectors.txt
domain: Apsis One - Integrations
topics: [Connector Types, Legacy Connectors, Generic Connector, Third-Party Integrations, Installer Options, Outbound Mappings, Integration Architecture, CRM Integration Patterns]
speakers: [Erik Andersson, Lukasz Grabowski, Michal Rosikiewicz, Shreevidhya Ganesan]
key_components: [Justin (integration platform), Legacy Connectors, Generic Connector, Third-Party Connectors, Installer Options, Outbound Mappings, Subscription Mappings, Field Mappings]
session_type: knowledge-transfer
---

## Session Overview

Erik Andersson delivered a comprehensive knowledge transfer on the three primary types of connectors in the **Apsis One integrations domain**. The session covered the evolution from **Legacy Connectors** (built in-house starting 2018) through the current **Generic Connector** pattern (introduced ~2022), and documented **Third-Party Integrations**. Key discussion centered on architectural decisions, maintenance sustainability, and configuration patterns. The session also covered **Installer Options** and **Outbound Mappings**, critical configuration structures for enabling CRM integrations.

---

## Historical Context: Evolution of Connector Architecture

### Why Legacy Connectors Became Unsustainable

**Legacy Connectors** were built starting in late 2018 as in-house custom implementations for specific CRM systems. The approach worked initially but created severe maintenance problems as the platform scaled.

[Erik Andersson]:
> When we started this project back in 2018, we started building connectors in house. For example we wanted to connect to Microsoft Dynamics and the way we did this is that we utilized the existing APIs inside of Microsoft to extract whatever data we needed... We had to adapt to their APIs, transform the data, handle any potential edge cases and also if we added new logic inside of APSIS that we wanted the Microsoft Dynamics feature to handle... we had to replicate the logic for each integration. When you get up to like even 4 integrations then you will notice that this is not a sustainable way of developing this because just adding one new feature would take like months of work.

The problem manifested as:
- **Data type mismatches**: For example, Microsoft Dynamics uses `1` and `0` for booleans while Apsis One requires `true`/`false`. This required transformation logic for each connector.
- **Feature replication burden**: Adding a new feature (e.g., syncing email events to CRM) required implementing it separately for each connector. With 4+ integrations, this becomes prohibitively expensive.
- **Maintenance nightmare**: Every bug fix or enhancement multiplies across all legacy connectors.

### Current Legacy Connector Landscape

Legacy connectors are maintained only for existing customers who cannot easily migrate:

- **Microsoft Dynamics**: ~60 customers on legacy connector
- **FSC Enterprise 12.0**: ~30-40 customers (they have since released 12.1 with generic connector support)
- **Lime CRM**: 1-2 customers (small but paying customers, so still supported)

[Erik Andersson]:
> However, we have since quite long said like we are not developing any new legacy connectors, we are also not adding any new features. The exception to this would probably be like if some very big customer tells us like hello, we will pay you a boatload of money if you add this specific feature. This has not happened though.

**Critical note**: New feature development for legacy connectors is explicitly blocked. Customers who need new features must migrate to Generic Connector variants.

---

## Generic Connector: The Modern Pattern

### Architectural Philosophy

The **Generic Connector** inverts the responsibility model. Instead of Apsis One adapting to each CRM's data structures and quirks, the CRM system must conform to a **standardized contract**.

[Erik Andersson]:
> The idea behind the generic connector is that instead of APSIS adapting to the CRM and us adapting to their data structure and having to do ugly hacks to get things working, the generic connector shifts the responsibility to the CRM instead. For the generic connector, we have like 1 connector in APSIS which is literally called generic connector where we add all of the support for it.

Key mechanism:
- **Single connector in Apsis One** handles all generic connector integrations
- **Standardized endpoints** that all CRMs must implement (e.g., `/records?page=<number>`)
- **Standardized data formats** for all field types (booleans must be `true`/`false`, not stringified values)
- **Feature flags** advertised by each CRM to declare what they support

### Historical Trigger for Generic Connector Adoption

[Shreevidhya Ganesan]:
> Basically summer 2022 we started this project of having the generic connectors and since then we have been developing only generic connector.

The adoption accelerated due to severe issues encountered with FSC Enterprise integrations:

**FSC Enterprise 12.0 Data Quality Issues** (pre-generic connector):
- **Default dates**: When a date field was empty, FSC Enterprise returned `1899-12-31`. This legitimate-looking date would trigger marketing automation flows, making customers appear 100+ years old.
- **Type inconsistencies**: Integer fields returned as stringified integers (`"123"` instead of `123`)
- **Boolean inconsistencies**: Boolean fields returned as stringified `"1"` or `"0"` instead of actual booleans
- **String conversions**: Required extensive data manipulation and type casting throughout the connector code

[Erik Andersson]:
> For example, if you have a date field in the CRM but you don't add any data to it, they set a default date of 1899 in December like that is a legit date as far as APSIS is concerned. So suddenly you started having marketing flows that could trigger and all customers were now suddenly 100 years old.

### Responsibility Shift Benefits

By making the CRM responsible for conforming to the contract:
1. **Apsis One has zero data transformation logic** for generic connectors
2. **Debugging becomes trivial**: If data looks wrong for one CRM but not another, the problem is clearly in that CRM's implementation
3. **Feature addition is multiplicative**: Adding a feature once benefits all conforming CRMs immediately
4. **CRM vendors control their roadmap**: They decide when to implement new features in their connectors

[Erik Andersson]:
> If you can see that the generic connector flow works for tribe, but when it comes to E deal it is behaving in a weird way, then it is easy to see, yeah, but then the problem exists in ED because it is working as it should in tribe or web CRM.

### Currently Supported Generic Connectors

- **Tribe** (E-Deal)
- **Web CRM**
- **FSC Enterprise 12.1** (newer version; 12.0 still uses legacy)
- **Microsoft Dynamics** (modern versions support generic connector)

**New CRM integrations should use generic connector exclusively.**

### Feature Declaration Mechanism

Each CRM advertises capabilities via the `/integrations` endpoint response:

```
{
  "can_sync_email_activities": true,
  "can_sync_sms_activities": true,
  "can_sync_event_tool_activities": false,
  ...
}
```

[Erik Andersson]:
> When you call integration and say like please list every integration that might be installed on this section, as part of this we are providing you with the information like can sync e-mail activity, can sync SMS activity, can sync Event tool activities. This information we retrieve from the CRM system.

Apsis One **only displays options** and **only sends data** for capabilities that the CRM has declared support for. This allows:
- Features to be hidden in the UI until the CRM supports them
- Different behavior per instance (e.g., dev environment with event tool support vs. production without)

### Feature Flag Versioning Strategy

[Michal Rosikiewicz]:
> When you add new features to generic connector, do you have some kind of versioning or this is depending on feature flags that CRM supports?

[Erik Andersson]:
> If we add something new in APSIS that the CRM does not support, we don't start sending that until they have enabled that feature flag in the CRM. For example the event tool is a good example. We are not displaying this option in APSIS until the CRM has stated that now we support event tool.

**No semantic versioning** of the generic connector exists. Instead:
- Each CRM controls when they declare support for new features
- Apsis One respects feature flags and only activates UI/data sending when declared
- A CRM can declare support for a feature independently of other CRMs

---

## Third-Party Integrations

### Architecture and Support Model

Third-party integrations are external integrations built and maintained by **partner companies**, not by Apsis One.

[Erik Andersson]:
> There are some external composite companies that have built what we call third party integrations to APSIS. These third party connectors or integrations, they don't really utilize Justin for most of their logic. They go through one instead like retrieving some e-mail events or creating profiles or consent. So they don't utilize essentially any of our logic like Apsis does not control this integration.

Apsis One's role is minimal:
- **Create a key space** when the third-party connector is installed
- **Whitelist the CRM ID attribute** for that key space
- **Generate and display a One API key** for the partner to use
- **Provide no ongoing support** beyond the initial setup

[Erik Andersson]:
> If the sinks are not happening as they should, we don't support any of that because we don't know how they work. We don't add new features to them or anything, we just provide you with the key space and the One API key and then we are done.

### Third-Party Integration Types

**Third-party partners using legacy API approach:**
- **Super Office**: ~5-6 customers
- **Intermail Loyalty**: ~1 customer (attempting to discontinue)
- **Join CX**: ~2-3 customers
- **Lead Family**: A few customers
- **WooCommerce**: ~3-4 customers

These all use the simple API key approach: partner configures integration in their own UI, copies the One API key from Apsis One.

### Special Case: Intermail Loyalty with Generic Connector Support

**Intermail** (a marketing platform using Apsis One) built their own integration and chose to implement full **Generic Connector support** themselves.

[Erik Andersson]:
> Intermail has built an integration like they are maintaining it on their side and they have built support for the generic connector like they are utilizing essentially everything in it. But it's not an APSIS connector like we don't control what they support or if they have added any new functionality.

How it works:
- When installed, the partner provides an API key (like any third-party integration)
- The partner's code implements the full generic connector specification
- Apsis One treats it like any other generic connector (field mappings, consent syncing, etc.)
- Partner owns all maintenance and feature development

This is a **hybrid model**: third-party ownership with generic connector-compatible implementation.

---

## Installer Options: Configuration Schema

**Installer Options** (also called "Connector Options") define how each CRM integration is configured. They are stored per-connector and handle the semantic details that differ between CRM systems.

### Core Installer Option Fields

#### Display Name and Integration ID

Each connector has two distinct identifiers:

```
Display Name: "Microsoft Dynamics 365"
Integration ID: "dynamics_crm_legacy"  # Used in code and APIs
```

The **Integration ID** is critical—it must differ from the display name to avoid confusion in logs and code references.

#### Feature Support Toggle

For generic connectors, Apsis One can disable features globally if a CRM **never** implemented support:

[Erik Andersson]:
> For example if we know that FSC enterprise for example like no they have not developed support for the event tool... then we can disable this function inside of APSIS and we will not even go to the CRM and check.

**If enabled**: Apsis One checks the instance's capabilities and respects the feature flag response.
**If disabled**: Feature is never shown or checked, avoiding unnecessary API calls.

#### Concurrency and Pagination Settings

When syncing contacts from CRM (full sync scenario):

```
thread_count: 4
page_size: 500
```

These tune the parallel download behavior for initial contact synchronization.

#### Profile Entity Configuration

Each CRM has a **primary entity** representing customers:

| CRM | Entity Name |
|-----|-------------|
| Microsoft Dynamics | `Contact` |
| Tribe | `Person` |
| FSC Enterprise | `Contact` |
| Web CRM | (varies) |

Configuration includes:
- **Display name**: Human-readable name for UI
- **Inbound enabled**: Should we download this entity from CRM to Apsis One?
- **Outbound enabled**: Should we send data from Apsis One (forms, task nodes) to CRM?
- **ID field name**: The unique identifier field used as the key in Apsis One

[Erik Andersson]:
> We configure like a kind of deadbolt on these can sync activities... what is the name of the connector? What is the logical ID of the connector? Like this is very very important to differ from like you have the display name and you have the what we call integration ID.

### ID Field Name: Critical for Record Matching

[Erik Andersson]:
> An important part is also the ID field name, because this is the unique identifier. As you know in APSIS like you always need to specify like a key space and a key and whatever value is contained here in the ID field name. This is the unique identifier like the key that we utilize in APSIS to identify the records with.

The ID field name is how Apsis One knows when to **update** an existing record vs. **create** a new one. Without a correctly configured ID field:
- Duplicate profiles will be created on every sync
- Consent updates will not merge correctly
- Data consistency breaks down

Different CRMs use different field names (`ID`, `ContactID`, `Identifier`, `PersonID`, etc.), so this **must be configurable per integration**.

---

## Inbound vs. Outbound Data Flow

### Inbound: CRM → Apsis One

**Direction**: Primary data flows from CRM to Apsis One  
**Trigger**: Full sync or incremental updates  
**Data**: Contact/customer records and attributes

When inbound is enabled on an entity, Apsis One requests: *"Please give us all records for the [entity name] because we want to download them to Apsis One."*

The **Rule of Thumb for Inbound**:
> The CRM is the authoritative source of all attribute data.

[Erik Andersson]:
> Usually like whenever you have an integration like the rule of thumb is that the CRM is the main source of the data. Whatever is in the CRM should be what is in APSIS, but this only holds true for the attributes.

**Exception**: Apsis One attributes are NEVER synced back to the CRM for this reason—you should not be modifying customer data in Apsis One; that should happen in the CRM.

### Outbound: Apsis One → CRM

**Direction**: Data flows from Apsis One to CRM  
**Triggers**: 
1. **Form submissions** (leads/opportunities)
2. **Task node entries** (Marketing Automation)
3. **Consent changes** (unsubscribes, GDPR requests)

#### Consent: The Bidirectional Exception

[Erik Andersson]:
> Consent is the only thing which is bidirectional, like we sync data from the CRM to APSIS. We sync the consent from APSIS to the CRM because you can modify the consent in APSIS in multiple ways, either via the unsubscribe link or like someone calling in and then ask support to do this GDPR thing on it.

**Why consent must sync both directions**:

If inbound-only, a dangerous scenario occurs:
1. Customer clicks unsubscribe link in Apsis One email → opted out in Apsis One
2. Next sync from CRM → pulls in their original opted-in consent
3. Next email campaign → customer receives email they explicitly opted out from
4. **Compliance violation and customer harm**

Therefore, whenever consent changes in Apsis One, it **must** be immediately synced back to the CRM so the CRM stays in sync.

---

## Outbound Mappings: Form Data to CRM Leads

### The Problem Without Outbound Mappings

When a form is submitted in Apsis One, we have the raw form field values. The CRM needs to create a lead, but how does it know which form field corresponds to which lead field?

[Erik Andersson]:
> Say that you create a form which has a field, my super cool cool field and favorite color like what in that form should be utilized the first name like you have no idea unless it explicitly says like first name so the outbound mappings here lets you map essentially the reverse of the field mapping.

**Example without mappings**:
```
Form submission:
{
  "eric_cool_field": "John",
  "sailing_boats": "Doe",
  "email_address": "john@example.com"
}
```

CRM receives this JSON but has no idea that `eric_cool_field` is the first name or `sailing_boats` is the last name.

### How Outbound Mappings Solve This

Outbound mappings create a **reverse mapping** from Apsis One attributes back to CRM fields.

**Configuration**:
1. Set up **field-to-attribute mappings** during initial integration setup (this maps CRM fields → Apsis One attributes for inbound sync)
2. The same configuration is **reused for outbound** to map Apsis One attributes → CRM fields
3. When a form is submitted with **outbound enabled**, Apsis One extracts the mapped attributes and creates a structured payload

[Erik Andersson]:
> We take the attributes on the profile that has the submit event and then create a Jason object with the data in this APSIS field, put into the field that the CRM would recognize. So let's say that your silhouettes or the lead structure in your CRM has a first name, last name and a mobile number, you can then say like this field in APSIS should be represent this field in the silhouette, this field in APSIS should be the last name etcetera.

**Enhanced request with outbound mappings**:
```
{
  "form_data": {
    "eric_cool_field": "John",
    "sailing_boats": "Doe",
    "email_address": "john@example.com"
  },
  "mapped_data": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com"
  }
}
```

The `mapped_data` section allows the CRM to populate lead fields with confidence.

### Outbound Enabled vs. Disabled

| Setting | Behavior |
|---------|----------|
| **Outbound Disabled** | Only raw form data sent; CRM must guess field mappings |
| **Outbound Enabled** | Both raw form data AND mapped_data sent; CRM can populate leads more completely |

**Both scenarios still send the form submission**—the flag only controls whether we enhance it with mapped attributes.

[Erik Andersson]:
> The only time we send these attributes in APSIS to the CRM is when you have submitted a form and we send like the profile data that has the submit event... we utilize those attributes when also when we send the event to the CRM system like this outbound enable. Essentially this regulates whether or not we do this whole dance in that flow or not.

### Outbound Mappings Support by CRM

- **Generic Connector CRMs**: Support outbound mappings (Tribe, Web CRM, Dynamics, FSC Enterprise 12.1)
- **Legacy Connectors**: Limited or no support
  - FSC Enterprise 12.0: No outbound mapping support (potential bug in 12.1 UI)
  - Lime CRM: No outbound mapping support

[Erik Andersson]:
> FSC enterprise 12.0 does not support outbound mappings. But the 12.1 does. We saw a bug now that it is not visible in 12.1, but it should be visible. That is also something we need to check on when I'm done with all the handover things.

---

## Integration Platform: Justin

All connectors are installed and managed within **Justin**, the Apsis One integration platform.

[Erik Andersson]:
> Justin is the name of the platform, so everything I say here is connected to Justin because like when the team name is integration but the platform name is Justin. Each connectors is handled inside of Justin. You can install them inside of Justin.

Each connector instance within Justin may support different feature sets depending on:
- Connector type (legacy vs. generic vs. third-party)
- CRM version (e.g., FSC Enterprise 12.0 vs. 12.1)
- CRM capabilities (declared via feature flags)

---

## Partner Integration Model: The Siteshop Example

### The Revenue and Support Problem

When Apsis One builds a connector and owns the customer relationship:
- Apsis One charges the customer for the connector
- Apsis One bears all debugging and support costs
- Partner bugs become Apsis One problems
- Development costs are high: ~2500 SEK/hour for external plugin fixes

[Erik Andersson]:
> For example, like I think it costs 2500 crown per hour to get the old Microsoft Dynamics plugin fixed and like that's not the cost that is sustainable. The amount of time you can spend on debugging and supporting this and the cost you need to spend to fix these bugs can eat up any revenue you had quite fast if you're unlucky.

### The Partner-Owned Model: Siteshop

**How it works**:
- **Siteshop owns the connector code** and maintains it
- **Siteshop owns the customer relationship** and support
- **Siteshop charges the customer** directly for the integration
- **Apsis One benefits** from inbound data (profile imports, email events) but doesn't charge separately

[Erik Andersson]:
> For Siteshop, APSIS is not charging the customer for the use of the connector. It is Siteshop that does it and the beauty of it is that Siteshop owns the maintenance and the supports for the connector on their side. So if the customers start having issues with the data sync to APSIS, then it is first and foremost Siteshop that handles the customer support cases.

**Benefits for Apsis One**:
- Zero maintenance burden
- All support filtered through Siteshop
- Apsis One only helps with Apsis One logs if needed
- Revenue comes from data processed (profiles, emails), not connector licensing

[Erik Andersson]:
> All of that is handled by Siteshop and I would probably recommend to have that approach going forward also that it is more important to find someone who wants to build a connector to APSIS and also own the customer and the support and in exchange APSIS will get the whatever the customers are using because the amount of time you can spend on debugging and supporting this and the cost you need to spend to fix these bugs can eat up any revenue you had quite fast.

### Recommendation for New Connectors

[Erik Andersson]:
> Really try to be firm with that like no find someone that wants to build an implementation for the generic connector and support it and then you would be more than happy to just add the configuration in APSIS required and of course assist with the development of this connector like that we can never get away from, but that is a completely different other thing.

**New connector strategy**:
1. ✅ Generic connector implementation only
2. ✅ Partner owns the connector code
3. ✅ Partner owns the customer relationship
4. ✅ Apsis One assists with integration-level questions, not debugging
5. ❌ Do NOT build custom connectors for individual customers
6. ❌ Do NOT build legacy-style adapters

---

## Feature Flag Management: Event Tool Sync Example

### Current Issue: Feature Flags vs. Capability Signals

The **Event Tool** sync feature has an account-level feature flag AND relies on the `/integrations` endpoint response for actual capability.

[Lukasz Grabowski]:
> I I checked in the event tool and we don't have this checkbox. I can't remember how it was done, but it looks like we automatically enable sync if there is integration enabled on the section.

[Erik Andersson]:
> You need to have feature flag. I think if memory serves you there are... we don't start sending them before they have told us that now you can start sending it.

**The problem**: Having both a feature flag AND capability detection is redundant. If a CRM doesn't declare support for `can_sync_event_tool_activities`, the option should never appear.

[Erik Andersson]:
> The problem with having that feature flag on the account level is that the customer might have one old instance like their production environment where the event tool syncing is not supported and then of course it shouldn't be visible, but it might also be that they have a section with a develop instance where they want to test out the event tool syncing functionality. So on the account it might differ.

**Better approach**: Remove the feature flag entirely and rely on the `/integrations` endpoint response, which is **per-instance** and therefore more accurate.

[Lukasz Grabowski]:
> I will create story to remove it entirely and rely only on this endpoint.

---

## Customer Migration Path: Enterprise 12.0 → 12.1

### The Challenge

FSC Enterprise customers on version **12.0** (legacy connector) want to upgrade to **12.1** (generic connector). However:

1. **Backward compatibility**: You cannot use the generic connector on 12.0 instances
2. **On-premise hesitation**: Enterprise customers with on-premise deployments are reluctant to upgrade
3. **Support burden**: Apsis One must maintain the 12.0 connector indefinitely for existing customers

[Erik Andersson]:
> Enterprise customers are usually like very big corporate customers who don't like change. So if they have a like on premise version, they aren't very hesitant to upgrade or migrate. That has been a recurring headache for us.

### Why Can't We Hide the Old Connector?

[Lukasz Grabowski]:
> Shouldn't we hide the old connector? I mean the old integration to enterprise is 12.0?

[Erik Andersson]:
> It's not that easy because unfortunately there might still be old versions of the enterprise sold for whatever reason and you can't use the new connector on the old version.

**Scenario**: A new customer purchases an existing 12.0 license and wants to use Apsis One. If we hide the 12.0 connector, we block them from integrating at all.

### Why Web CRM Could Hide Its Legacy Connector

Web CRM is a **SaaS service** with automatic updates. Every customer is on the latest version, so:
- All new customers get 12.1 (generic connector support)
- All existing customers upgraded automatically
- The old connector can be hidden (or shown only for backward compatibility access)

[Erik Andersson]:
> Web CRM has done this exact journey. You had web CRM which we built a connector for and then you had web CRM where they added support for the generic connector. Now web CRM is a SaaS service like you don't have any on premise instances or things. So every customer could be migrated to the new version and in that case we could hide the old integration.

### Recommendation: Consultancy-Led Migration

The ideal path is to work with **Consultancy** (the professional services team) to:
1. Identify old-version customers
2. Plan migrations to 12.1 or newer
3. Test data migration
4. Handle customer communication

[Erik Andersson]:
> The dream state is of course that every old, every old one is migrated and I would highly suggest that like this is pushed for in tandem with consultancy because it's consultancy that typically handles the actual migration because they sit with the customer and their data.

---

## Key Takeaways

1. **Three connector types exist**: Legacy (unsustainable, maintained for backward compatibility), Generic (modern standard), Third-Party (partner-owned, minimal Apsis support)

2. **Generic Connector is the future**: All new integrations must use the generic connector pattern. This shifts data transformation responsibility to the CRM partner.

3. **Responsibility inversion is critical**: Legacy connectors failed because Apsis One had to adapt to each CRM's quirks (default dates, type inconsistencies, etc.). Generic connectors succeed because CRMs conform to a strict contract.

4. **Feature flags and capability signals**: CRMs declare what they support via feature flags in the `/integrations` response. Apsis One only shows/sends data for declared capabilities.

5. **Installer Options are semantic configuration**: Each CRM requires field name mappings, entity names, ID field names, and feature toggles configured per integration.

6. **Consent is bidirectional**: Unlike all other data, consent MUST sync both CRM→Apsis One (inbound) and Apsis One→CRM (outbound) to prevent compliance violations.

7. **Outbound mappings enhance form data**: Mapping Apsis One attributes back to CRM fields allows richer lead creation from form submissions. Without it, the CRM must guess field meanings.

8. **Partner-owned is preferable**: New connectors should be built and owned by partners (like Siteshop) with Apsis One providing integration support, not bearing the full maintenance and support burden.

9. **Justin is the integration platform**: All connectors are installed and managed within Justin, with per-instance feature capability declaration.

10. **Feature flag cleanup needed**: The Event Tool feature flag is redundant and should be removed in favor of relying solely on the `/integrations` endpoint response.

---

## Unresolved Questions and Action Items

1. **FSC Enterprise 12.1 outbound mapping visibility**: A bug exists where outbound mappings are not visible in the 12.1 UI despite being supported. [Erik to investigate]

2. **Event Tool feature flag**: Remove the account-level feature flag and rely entirely on `/integrations` endpoint capability signals. [Lukasz to create story]

3. **Feature flag for event tool configuration details**: Exact implementation details on where the feature flag is stored and how to remove it were not finalized.

4. **Code repository structure**: Next session will cover the actual codebase, file structure, and where developers should look for connector implementations.

5. **Inbound/Outbound flow deep dive**: Full details of how data flows in both directions, batching, event listeners, and audience integration. Deferred to future session.

6. **Migration strategy for FSC Enterprise 12.0 customers**: Formal plan for identifying and migrating old customers to 12.1. Involves coordination with Consultancy.

---

## Session Logistics

- **Recording length**: 1h 33m 42s
- **Break taken**: ~8 minutes at 51m mark for water/throat break
- **Next session planned**: Code repository structure and file organization
- **Estimated timeline**: 3 months of knowledge transfer sessions